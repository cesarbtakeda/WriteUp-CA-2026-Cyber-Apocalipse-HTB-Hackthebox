# Fractured Seal — HTB Crypto Write-up
## Resolvido por Joaov1t
## Informações do desafio

- **Nome:** Fractured Seal
- **Categoria:** Cryptography
- **Técnica principal:** RSA Partial Key Exposure
- **Ataque utilizado:** Coppersmith / small roots
- **Arquivos fornecidos:** `encrypt.py`, `fractured_seal.pem` e `flag.enc`

---

## 1. Tradução da descrição

> **Selo Fraturado**
>
> Um dos pergaminhos de chaves mais antigos do Registro sobreviveu à queda de Crownspire, embora o tempo e o fogo tenham poupado apenas fragmentos de sua escrita, e a maioria no cofre o tenha descartado como inútil.
>
> Caldrin não.
>
> Ela sempre dizia que um selo não precisa estar inteiro para ainda se lembrar da porta que um dia abriu.

A descrição indica que recebemos uma chave incompleta, mas que os fragmentos restantes ainda contêm informação suficiente para reconstruí-la.

---

## 2. Objetivo do desafio

O `encrypt.py` cria RSA usando dois primos de 1024 bits:

```python
p = getPrime(1024)
q = getPrime(1024)
n = p * q
e = 0x10001
d = pow(e, -1, (p - 1) * (q - 1))
```

A flag é convertida para um número e criptografada:

```python
m = bytes_to_long(open("flag.txt", "rb").read())
c = pow(m, e, n)
```

Em termos simples:

```text
p e q = primos secretos
n     = p × q
e     = expoente público
d     = expoente privado
m     = mensagem original
c     = mensagem criptografada
```

A criptografia é:

```text
c = m^e mod n
```

A descriptografia é:

```text
m = c^d mod n
```

Para reconstruir `d`, precisamos recuperar `p` e `q`.

O arquivo `fractured_seal.pem` é uma chave privada parcialmente apagada. Mesmo danificada, ela ainda revela quase todo `n` e uma parte grande de `q`.

> Não execute o `encrypt.py` para tentar gerar novamente a chave. Cada execução cria novos primos aleatórios, diferentes dos usados no `flag.enc`.

---

## 3. Resumo do ataque

```text
PEM danificado
      ↓
Recuperar quase todo n
      ↓
Criar somente 2 candidatos para n
      ↓
Recuperar os 588 bits iniciais de q
      ↓
Restam 436 bits desconhecidos
      ↓
Usar Coppersmith
      ↓
Recuperar q
      ↓
Calcular p = n / q
      ↓
Reconstruir d
      ↓
Descriptografar flag.enc
```

A vulnerabilidade é conhecida como:

```text
RSA Partial Key Exposure
```

O RSA não foi fatorado do zero. O ataque funcionou porque uma parte grande de um dos primos permaneceu visível.

---

# Exploração passo a passo

## 4. Confirmar os arquivos

```bash
ls -la
```

Arquivos necessários:

```text
encrypt.py
fractured_seal.pem
flag.enc
```

---

## 5. Inspecionar os arquivos

Crie o script:

```bash
cat > 01_inspect.sh <<'EOF'
#!/usr/bin/env bash

set -u

echo "========================================"
echo "[+] Tipos dos arquivos"
echo "========================================"

file encrypt.py fractured_seal.pem flag.enc

echo
echo "========================================"
echo "[+] Tamanhos"
echo "========================================"

wc -c encrypt.py fractured_seal.pem flag.enc

echo
echo "========================================"
echo "[+] Código de criptografia"
echo "========================================"

cat encrypt.py

echo
echo "========================================"
echo "[+] PEM danificado"
echo "========================================"

nl -ba fractured_seal.pem

echo
echo "========================================"
echo "[+] Tentativa de leitura com OpenSSL"
echo "========================================"

openssl rsa \
    -in fractured_seal.pem \
    -text \
    -noout 2>&1 || true

echo
echo "[+] A falha do OpenSSL é esperada."
echo "[+] O PEM contém asteriscos e outros caracteres inválidos."
EOF
```

Execute:

```bash
chmod +x 01_inspect.sh
./01_inspect.sh
```

### O que observar

O `flag.enc` possui 256 bytes:

```text
256 × 8 = 2048 bits
```

Isso combina com um módulo RSA criado por dois primos de 1024 bits.

O OpenSSL falha porque o conteúdo Base64/DER da chave foi parcialmente substituído por `*`.

---

## 6. Estrutura de uma chave privada RSA

Uma chave privada RSA PKCS#1 normalmente contém:

```text
n
e
d
p
q
dp
dq
qInv
```

A relação principal é:

```text
n = p × q
```

Como vários valores relacionados são armazenados juntos, uma chave parcialmente destruída pode continuar vazando informação suficiente para reconstrução.

O corpo do PEM é DER codificado em Base64:

```text
-----BEGIN RSA PRIVATE KEY-----
BASE64
-----END RSA PRIVATE KEY-----
```

No Base64:

```text
4 caracteres = 3 bytes
```

---

## 7. Extrair os fragmentos conhecidos

Crie:

```bash
cat > 02_extract_fragments.py <<'EOF'
#!/usr/bin/env python3

from pathlib import Path
import base64


PEM_FILE = Path("fractured_seal.pem")

BASE64_ALPHABET = (
    "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    "abcdefghijklmnopqrstuvwxyz"
    "0123456789+/"
)


def recover_n_candidates(body_lines):
    body = "".join(body_lines)

    # Tudo antes do primeiro '*' está intacto.
    known_prefix = body.split("*", 1)[0]

    print("[+] Caracteres Base64 conhecidos no início:", len(known_prefix))

    # 356 caracteres formam grupos Base64 completos.
    der_prefix = base64.b64decode(known_prefix[:356])

    print("[+] Bytes DER recuperados:", len(der_prefix))
    print("[+] Início DER:", der_prefix[:16].hex())

    # Os primeiros 12 bytes são cabeçalhos DER.
    # Depois começam os bytes de n.
    n_first_255_bytes = der_prefix[12:]

    print("[+] Bytes completos conhecidos de n:", len(n_first_255_bytes))

    # O caractere seguinte contém os 6 bits superiores
    # do último byte de n.
    final_character = known_prefix[356]
    high_six_bits = BASE64_ALPHABET.index(final_character)
    last_byte_prefix = high_six_bits << 2

    candidates = []

    # Restam 2 bits. Como n é ímpar, o último bit precisa ser 1.
    for low_bits in (0b01, 0b11):
        final_byte = last_byte_prefix | low_bits
        n_bytes = n_first_255_bytes + bytes([final_byte])
        candidates.append(int.from_bytes(n_bytes, "big"))

    return candidates


def recover_q_prefix(body_lines):
    fragment = (
        body_lines[13][52:]
        + body_lines[14]
        + body_lines[15][:30]
    )

    print("[+] Tamanho do segundo fragmento:", len(fragment))

    decoded_fragment = base64.b64decode(fragment[:104])

    print("[+] Bytes decodificados:", len(decoded_fragment))
    print("[+] Início do fragmento:", decoded_fragment[:16].hex())

    # Ignora o final do campo anterior e o cabeçalho DER de q.
    q_complete_bytes = decoded_fragment[6:]

    print("[+] Bytes completos conhecidos de q:", len(q_complete_bytes))

    first_value = BASE64_ALPHABET.index(fragment[104])
    second_value = BASE64_ALPHABET.index(fragment[105])

    next_full_byte = (first_value << 2) | (second_value >> 4)
    next_high_nibble = second_value & 0x0F

    q_prefix = int.from_bytes(
        q_complete_bytes + bytes([next_full_byte]),
        "big",
    )

    q_prefix = (q_prefix << 4) | next_high_nibble

    known_bits = len(q_complete_bytes) * 8 + 8 + 4

    return q_prefix, known_bits


def main():
    lines = PEM_FILE.read_text().splitlines()
    body_lines = lines[1:-1]

    print("========================================")
    print("[+] Recuperando candidatos de n")
    print("========================================")

    candidates = recover_n_candidates(body_lines)

    for index, candidate in enumerate(candidates, 1):
        print()
        print(f"[+] Candidato {index}:")
        print("    bits:", candidate.bit_length())
        print("    últimos 16 bits:", hex(candidate & 0xFFFF))
        print("    é ímpar:", candidate % 2 == 1)

    print()
    print("========================================")
    print("[+] Recuperando prefixo de q")
    print("========================================")

    q_prefix, known_bits = recover_q_prefix(body_lines)

    unknown_bits = 1024 - known_bits

    print()
    print("[+] Bits conhecidos de q:", known_bits)
    print("[+] Bits desconhecidos de q:", unknown_bits)
    print("[+] Tamanho do prefixo:", q_prefix.bit_length())

    print()
    print("[+] Modelo:")
    print("    q = prefixo * 2^436 + x")
    print("    0 <= x < 2^436")


if __name__ == "__main__":
    main()
EOF
```

Execute:

```bash
chmod +x 02_extract_fragments.py
python3 02_extract_fragments.py
```

Resultado principal:

```text
[+] Bytes completos conhecidos de n: 255
[+] Bits conhecidos de q: 588
[+] Bits desconhecidos de q: 436
```

---

## 8. Por que existem dois candidatos para `n`?

Foram recuperados 255 bytes completos de `n` e os 6 bits superiores do último byte:

```text
██████??
```

Os dois bits ausentes poderiam ser:

```text
00
01
10
11
```

Como `p` e `q` são ímpares:

```text
ímpar × ímpar = ímpar
```

Logo, `n` também é ímpar e termina em bit `1`.

Sobram apenas:

```text
01
11
```

Por isso testamos somente dois candidatos.

---

## 9. Por que conhecemos 588 bits de `q`?

O fragmento preservou:

```text
72 bytes completos
+ 1 byte completo
+ 4 bits
```

Calculando:

```text
72 × 8 = 576
576 + 8 + 4 = 588 bits
```

Como `q` tem 1024 bits:

```text
1024 - 588 = 436 bits desconhecidos
```

Representamos:

```text
q = q_aproximado + x
```

ou:

```text
q = prefixo × 2^436 + x
```

---

## 10. Instalar SageMath com Miniforge

O pacote `sagemath` não estava disponível nos repositórios APT do sistema Ubuntu utilizado. Foi usado Miniforge com `conda-forge`.

Crie:

```bash
cat > 00_install_sage_miniforge.sh <<'EOF'
#!/usr/bin/env bash

set -euo pipefail

MINIFORGE_DIR="$HOME/miniforge3"
INSTALLER="/tmp/Miniforge3-$(uname)-$(uname -m).sh"
CONDA="$MINIFORGE_DIR/bin/conda"

echo "========================================"
echo "[+] Instalando dependências"
echo "========================================"

sudo apt update
sudo apt install -y curl ca-certificates bzip2

if [[ ! -x "$CONDA" ]]; then
    echo "[+] Baixando Miniforge..."

    curl \
        --fail \
        --location \
        --retry 3 \
        --output "$INSTALLER" \
        "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"

    bash "$INSTALLER" -b -p "$MINIFORGE_DIR"
    rm -f "$INSTALLER"
else
    echo "[+] Miniforge já instalado."
fi

if [[ ! -x "$MINIFORGE_DIR/envs/sage/bin/sage" ]]; then
    echo "[+] Criando ambiente SageMath..."

    "$CONDA" create \
        --yes \
        --name sage \
        --channel conda-forge \
        sage
else
    echo "[+] Ambiente SageMath já existe."
fi

"$CONDA" run \
    --no-capture-output \
    --name sage \
    sage --version

echo "[+] SageMath instalado corretamente."
EOF
```

Execute:

```bash
chmod +x 00_install_sage_miniforge.sh
./00_install_sage_miniforge.sh
```

Confirme:

```bash
~/miniforge3/bin/conda run \
    --no-capture-output \
    -n sage \
    sage --version
```

> Não misture repositórios do Kali com Ubuntu para instalar SageMath.

---

## 11. Ataque de Coppersmith

Sabemos:

```text
n = p × q
```

Portanto, `q` divide `n`.

Também sabemos:

```text
q = q_aproximado + x
```

Criamos:

```text
f(x) = q_aproximado + x
```

Coppersmith procura a raiz pequena `x` que completa `q`.

Ele não testa todas as `2^436` possibilidades. O método usa álgebra de reticulados para recuperar a parte ausente.

Como `q` possui aproximadamente metade dos bits de `n`, usamos:

```python
beta=0.5
```

---

## 12. Recuperar `p` e `q`

Crie:

```bash
cat > 03_recover_factors.sage <<'EOF'
from pathlib import Path
import base64


PEM_FILE = Path("fractured_seal.pem")

BASE64_ALPHABET = (
    "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    "abcdefghijklmnopqrstuvwxyz"
    "0123456789+/"
)


def recover_n_candidates(body_lines):
    body = "".join(body_lines)
    known_prefix = body.split("*", 1)[0]

    decoded_prefix = base64.b64decode(known_prefix[:356])
    n_first_255_bytes = decoded_prefix[12:]

    final_character = known_prefix[356]
    high_six_bits = BASE64_ALPHABET.index(final_character)
    last_byte_prefix = high_six_bits << 2

    candidates = []

    for low_bits in (0b01, 0b11):
        final_byte = last_byte_prefix | low_bits
        n_bytes = n_first_255_bytes + bytes([final_byte])

        candidates.append(
            Integer(int.from_bytes(n_bytes, "big"))
        )

    return candidates


def recover_q_information(body_lines):
    fragment = (
        body_lines[13][52:]
        + body_lines[14]
        + body_lines[15][:30]
    )

    decoded_fragment = base64.b64decode(fragment[:104])
    q_complete_bytes = decoded_fragment[6:]

    first_value = BASE64_ALPHABET.index(fragment[104])
    second_value = BASE64_ALPHABET.index(fragment[105])

    next_full_byte = (
        (first_value << 2)
        | (second_value >> 4)
    )

    next_high_nibble = second_value & 0x0F

    q_prefix = int.from_bytes(
        q_complete_bytes + bytes([next_full_byte]),
        "big",
    )

    q_prefix = Integer(
        (q_prefix << 4)
        | next_high_nibble
    )

    known_bits = len(q_complete_bytes) * 8 + 8 + 4
    unknown_bits = 1024 - known_bits
    q_approximation = q_prefix << unknown_bits

    return q_approximation, known_bits, unknown_bits


def main():
    pem_lines = PEM_FILE.read_text().splitlines()
    body_lines = pem_lines[1:-1]

    n_candidates = recover_n_candidates(body_lines)

    (
        q_approximation,
        known_bits,
        unknown_bits,
    ) = recover_q_information(body_lines)

    print("[+] Número de candidatos para n:", len(n_candidates))
    print("[+] Bits conhecidos de q:", known_bits)
    print("[+] Bits desconhecidos de q:", unknown_bits)
    print()

    X = Integer(2) ** unknown_bits

    for index, N in enumerate(n_candidates, 1):
        print("========================================")
        print(f"[*] Testando candidato {index} de n")
        print("========================================")
        print("[+] Bits de n:", N.nbits())

        polynomial_ring = PolynomialRing(
            Zmod(N),
            names=("x",),
        )

        x = polynomial_ring.gen()
        polynomial = x + q_approximation

        for epsilon in (0.03, 0.02):
            print()
            print("[*] Executando Coppersmith")
            print("[*] epsilon =", epsilon)

            roots = polynomial.small_roots(
                X=X,
                beta=0.5,
                epsilon=epsilon,
            )

            print("[+] Raízes encontradas:", len(roots))

            for root in roots:
                possible_q_value = Integer(
                    q_approximation + Integer(root)
                )

                possible_q = gcd(N, possible_q_value)

                if possible_q in (1, N):
                    continue

                possible_p = N // possible_q

                if possible_p * possible_q != N:
                    continue

                print()
                print("[+] Fatores recuperados!")
                print("[+] p possui", possible_p.nbits(), "bits")
                print("[+] q possui", possible_q.nbits(), "bits")

                assert possible_p.is_prime()
                assert possible_q.is_prime()
                assert possible_p * possible_q == N

                print("[+] p e q são primos.")
                print("[+] p * q == n.")

                output = (
                    f"{N}\n"
                    f"{possible_p}\n"
                    f"{possible_q}\n"
                )

                Path("recovered_factors.txt").write_text(output)

                print("[+] Valores salvos em recovered_factors.txt")
                return

    print("[-] Nenhum fator foi encontrado.")


main()
EOF
```

---

## 13. Criar executor do SageMath

```bash
cat > 03_run_recover.sh <<'EOF'
#!/usr/bin/env bash

set -euo pipefail

CONDA="$HOME/miniforge3/bin/conda"
SAGE_FILE="03_recover_factors.sage"

if [[ ! -x "$CONDA" ]]; then
    echo "[-] Miniforge não encontrado."
    echo "[-] Execute ./00_install_sage_miniforge.sh"
    exit 1
fi

if [[ ! -f "$SAGE_FILE" ]]; then
    echo "[-] Arquivo $SAGE_FILE não encontrado."
    exit 1
fi

"$CONDA" run \
    --no-capture-output \
    --name sage \
    sage "$SAGE_FILE"
EOF
```

Execute:

```bash
chmod +x 03_run_recover.sh
./03_run_recover.sh
```

Resultado esperado:

```text
[+] Fatores recuperados!
[+] p possui 1024 bits
[+] q possui 1024 bits
[+] p e q são primos.
[+] p * q == n.
```

O script cria:

```text
recovered_factors.txt
```

Confira:

```bash
ls -lh recovered_factors.txt
wc -l recovered_factors.txt
```

O arquivo deve ter três linhas:

```text
n
p
q
```

---

## 14. Reconstruir a chave privada

Com `p` e `q`:

```text
φ(n) = (p - 1)(q - 1)
```

O expoente público é:

```text
e = 65537
```

Reconstruímos:

```text
d = e^-1 mod φ(n)
```

Finalmente:

```text
m = c^d mod n
```

---

## 15. Descriptografar `flag.enc`

Crie:

```bash
cat > 04_decrypt.py <<'EOF'
#!/usr/bin/env python3

from math import gcd
from pathlib import Path


FACTORS_FILE = Path("recovered_factors.txt")
CIPHERTEXT_FILE = Path("flag.enc")


def int_to_bytes(value: int) -> bytes:
    byte_length = max(1, (value.bit_length() + 7) // 8)
    return value.to_bytes(byte_length, "big")


def main():
    print("========================================")
    print("[+] Descriptografia RSA")
    print("========================================")

    if not FACTORS_FILE.exists():
        raise SystemExit(
            "[-] recovered_factors.txt não encontrado."
        )

    if not CIPHERTEXT_FILE.exists():
        raise SystemExit(
            "[-] flag.enc não encontrado."
        )

    lines = FACTORS_FILE.read_text().strip().splitlines()

    if len(lines) != 3:
        raise SystemExit(
            "[-] recovered_factors.txt deve ter 3 linhas."
        )

    n = int(lines[0])
    p = int(lines[1])
    q = int(lines[2])

    print("[+] n possui", n.bit_length(), "bits")
    print("[+] p possui", p.bit_length(), "bits")
    print("[+] q possui", q.bit_length(), "bits")

    if p * q != n:
        raise SystemExit(
            "[-] p * q não é igual a n."
        )

    print("[+] p * q == n")

    e = 65537
    phi = (p - 1) * (q - 1)

    if gcd(e, phi) != 1:
        raise SystemExit(
            "[-] e não possui inverso modular."
        )

    d = pow(e, -1, phi)

    assert (e * d) % phi == 1

    print("[+] Expoente privado d reconstruído.")

    ciphertext_bytes = CIPHERTEXT_FILE.read_bytes()

    ciphertext = int.from_bytes(
        ciphertext_bytes,
        "big",
    )

    if ciphertext >= n:
        raise SystemExit(
            "[-] Ciphertext maior ou igual a n."
        )

    plaintext_integer = pow(ciphertext, d, n)
    plaintext = int_to_bytes(plaintext_integer)

    print()
    print("========================================")
    print("[+] Plaintext recuperado")
    print("========================================")

    try:
        print(plaintext.decode("utf-8"))
    except UnicodeDecodeError:
        print("[+] Bytes:", plaintext)
        print("[+] Hexadecimal:", plaintext.hex())


if __name__ == "__main__":
    main()
EOF
```

Execute:

```bash
chmod +x 04_decrypt.py
python3 04_decrypt.py
```

A flag aparecerá no terminal.

---

## 16. Ordem final dos comandos

```bash
./01_inspect.sh
```

```bash
python3 02_extract_fragments.py
```

```bash
./00_install_sage_miniforge.sh
```

```bash
./03_run_recover.sh
```

```bash
python3 04_decrypt.py
```

---

## 17. Flag

```text
HTB{r3c0v3r1ng_RSA_k3ys___l1k3___Me0w___me0o00o0o0w___Me0w}
```

---

## 18. Conclusão

O desafio usa RSA de 2048 bits com dois primos de 1024 bits. Fatorar `n` diretamente seria inviável.

A chave privada danificada ainda revela:

- quase todo o módulo `n`;
- 588 bits mais significativos de `q`;
- somente 436 bits finais de `q` permanecem desconhecidos.

O ataque de Coppersmith recupera essa parte ausente. Depois:

```text
q recuperado
    ↓
p = n / q
    ↓
φ(n) = (p - 1)(q - 1)
    ↓
d = e^-1 mod φ(n)
    ↓
m = c^d mod n
```

Assim, a chave privada é reconstruída e o ciphertext é descriptografado.

### Conceitos aprendidos

- funcionamento básico do RSA;
- relação entre `p`, `q`, `n`, `e` e `d`;
- estrutura PKCS#1;
- Base64 e DER;
- exposição parcial de chave;
- ataque de Coppersmith;
- recuperação de fatores;
- inverso modular;
- descriptografia RSA.
