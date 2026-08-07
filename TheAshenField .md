# The Ashen Field — HTB Crypto Writeup
## Resolvido por Joaov1t

## Informações do desafio

- **Categoria:** Cryptography
- **Nome:** The Ashen Field
- **Arquivos:** `source.sage` e `output.txt`
- **Objetivo:** recuperar a chave secreta usada para derivar a chave AES e descriptografar a flag.

---

## 1. Contexto do desafio

A descrição fala sobre uma relíquia capaz de preservar uma informação sem revelá-la, utilizando apenas equações públicas.

No código, isso aparece como um esquema de criptografia multivariada inspirado em **HFE — Hidden Field Equations**. Uma chave pública formada por vários polinômios recebe os bits de uma mensagem e devolve outro vetor de bits.

A chave secreta `KEY` é criptografada por esse sistema. Depois, a mesma `KEY` é usada para gerar uma chave AES que protege a flag.

O fluxo é:

```text
KEY de 137 bits
      |
      |-- criptografada pelos polinômios --> encrypted_key
      |
      `-- SHA256(str(KEY)) --> AES_KEY --> AES-ECB(flag)
```

Portanto, precisamos recuperar a `KEY` a partir dos polinômios públicos e do vetor `encrypted_key`.

---

## 2. Análise dos arquivos

Primeiro, listamos os arquivos recebidos:

```bash
ls -la
```

Arquivos importantes:

```text
source.sage
output.txt
```

Podemos visualizar o código com:

```bash
sed -n '1,220p' source.sage
```

O `output.txt` possui apenas três linhas, mas a primeira é muito grande:

```bash
wc -l output.txt
head -c 500 output.txt
```

A estrutura do arquivo é:

```text
Linha 1: chave pública, contendo 137 polinômios
Linha 2: KEY criptografada, contendo 137 bits
Linha 3: flag criptografada em hexadecimal
```

---

## 3. Entendendo a geração da flag

No final do `source.sage`, o programa gera uma `KEY` aleatória com 137 bits:

```python
while True:
    KEY = Integer("".join(str(secrets.randbits(1)) for _ in range(N)), 2)
    if KEY.nbits() == N:
        break
```

Depois, essa chave é criptografada pelos polinômios públicos:

```python
encrypted_key = encrypt(KEY, PK)
```

A chave do AES é o SHA-256 da representação decimal da `KEY`:

```python
AES_KEY = hashlib.sha256(str(KEY).encode()).digest()
```

Por fim, a flag é cifrada com AES no modo ECB:

```python
cipher = AES.new(AES_KEY, AES.MODE_ECB)
enc_flag = cipher.encrypt(pad(FLAG, 16))
```

Assim, depois de recuperar o inteiro `KEY`, podemos recriar exatamente a mesma chave AES:

```python
AES_KEY = hashlib.sha256(str(KEY).encode()).digest()
```

---

## 4. Identificando a vulnerabilidade

O sistema trabalha no corpo finito `GF(2)`:

```python
q = 2
K = GF(q)
```

Em `GF(2)`, os únicos valores possíveis são:

```text
0 e 1
```

Para qualquer bit `x`, temos:

```text
x² = x
x⁴ = x
```

Exemplos:

```text
0² = 0
1² = 1

0⁴ = 0
1⁴ = 1
```

Os polinômios do `output.txt` parecem ser de grau quatro:

```text
x1^4 + x2^2 + x5^4 + 1
```

Porém, quando as entradas são bits, isso equivale a:

```text
x1 + x2 + x5 + 1
```

Portanto, os polinômios não são realmente difíceis de resolver. Todos podem ser transformados em **equações lineares sobre GF(2)**.

Também devemos lembrar que a soma em `GF(2)` é um XOR:

```text
0 + 0 = 0
0 + 1 = 1
1 + 0 = 1
1 + 1 = 0
```

Se uma variável aparecer duas vezes após a simplificação, ela se cancela:

```text
x1⁴ + x1² = x1 + x1 = 0
```

---

## 5. Erro adicional na geração das matrizes

O código tenta gerar duas matrizes para esconder a estrutura do sistema:

```python
while True:
    S_A, T_A = [matrix.random(K, n, n) for _ in range(2)]
    if not False in [S_A.is_singular(), T_A.is_singular()]:
        break
```

A condição aceita as matrizes quando `False` não aparece na lista. Isso significa que os dois resultados precisam ser `True`:

```text
S_A.is_singular() = True
T_A.is_singular() = True
```

Logo, as duas matrizes escolhidas são **singulares**, ou seja, não são invertíveis.

Isso reduz o rank do sistema. No desafio obtivemos:

```text
Número de variáveis: 137
Rank do sistema:     135
Nullity:               2
```

A nullity indica quantas variáveis livres existem. Com duas variáveis livres, o número total de soluções é:

```text
2² = 4 soluções
```

Em vez de testar `2^137` chaves, precisamos testar apenas quatro.

---

## 6. Estratégia de exploração

O ataque será feito nas seguintes etapas:

1. Ler os três valores do `output.txt`.
2. Separar os 137 polinômios públicos.
3. Trocar cada `xᵢ²` e `xᵢ⁴` por `xᵢ`.
4. Montar um sistema linear sobre `GF(2)`.
5. Resolver o sistema com eliminação de Gauss usando XOR.
6. Encontrar uma solução particular e a base do núcleo.
7. Gerar os quatro candidatos possíveis para `KEY`.
8. Calcular `SHA256(str(KEY))` para cada candidato.
9. Descriptografar a flag com AES-ECB.
10. Validar o padding PKCS#7 e procurar o prefixo `HTB{`.

---

## 7. Preparando o ambiente

Podemos criar um ambiente virtual e instalar o PyCryptodome:

```bash
python3 -m venv venv
source venv/bin/activate
pip install pycryptodome
```

O solver também aceita a biblioteca `cryptography` como alternativa:

```bash
pip install cryptography
```

Os arquivos devem ficar no mesmo diretório:

```text
.
├── output.txt
├── solve.py
└── source.sage
```

---

## 8. Solver

Crie o arquivo:

```bash
nano solve.py
```

Adicione o seguinte código:

```python
#!/usr/bin/env python3
import ast
import hashlib
import re
from pathlib import Path

N = 137


def aes_ecb_decrypt(key: bytes, ciphertext: bytes) -> bytes:
    """Decrypt AES-ECB using either PyCryptodome or cryptography."""
    try:
        from Crypto.Cipher import AES
        return AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
    except ImportError:
        from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
        decryptor = Cipher(algorithms.AES(key), modes.ECB()).decryptor()
        return decryptor.update(ciphertext) + decryptor.finalize()


def pkcs7_unpad(data: bytes, block_size: int = 16) -> bytes:
    if not data:
        raise ValueError("empty plaintext")
    padding = data[-1]
    if padding < 1 or padding > block_size:
        raise ValueError("invalid padding")
    if data[-padding:] != bytes([padding]) * padding:
        raise ValueError("invalid padding")
    return data[:-padding]


def parse_output(path: str):
    lines = Path(path).read_text().splitlines()
    if len(lines) != 3:
        raise ValueError("output.txt must contain exactly 3 lines")

    public_key_line, encrypted_key_line, encrypted_flag_line = lines
    public_polynomials = public_key_line[1:-1].split(", ")
    encrypted_key = ast.literal_eval(encrypted_key_line)
    encrypted_flag = bytes.fromhex(encrypted_flag_line)

    if len(public_polynomials) != N or len(encrypted_key) != N:
        raise ValueError("unexpected public key/ciphertext size")

    return public_polynomials, encrypted_key, encrypted_flag


def boolean_linearize(polynomial: str):
    """
    For bit inputs over GF(2): x^2 = x and x^4 = x.
    Returns the linear coefficient mask and constant term.
    """
    mask = 0

    for variable, _power in re.findall(r"x(\d+)\^(2|4)", polynomial):
        bit = int(variable) - 1
        mask ^= 1 << bit

    constant = int(bool(re.search(r"(^| \+ )1($| \+ )", polynomial)))
    return mask, constant


def solve_affine_system(polynomials, ciphertext):
    # Each row stores [coefficient bit-mask, right-hand side].
    rows = []
    for polynomial, output_bit in zip(polynomials, ciphertext):
        mask, constant = boolean_linearize(polynomial)
        rows.append([mask, output_bit ^ constant])

    # Reduced row-echelon form over GF(2).
    pivot_columns = []
    pivot_row = 0

    for column in range(N):
        candidate = next(
            (row for row in range(pivot_row, N) if (rows[row][0] >> column) & 1),
            None,
        )
        if candidate is None:
            continue

        rows[pivot_row], rows[candidate] = rows[candidate], rows[pivot_row]
        pivot_mask, pivot_rhs = rows[pivot_row]

        for row in range(N):
            if row != pivot_row and ((rows[row][0] >> column) & 1):
                rows[row][0] ^= pivot_mask
                rows[row][1] ^= pivot_rhs

        pivot_columns.append(column)
        pivot_row += 1

    for mask, rhs in rows:
        if mask == 0 and rhs:
            raise ValueError("inconsistent linear system")

    free_columns = [column for column in range(N) if column not in pivot_columns]

    # Particular solution: set all free variables to zero.
    particular = 0
    for row, column in enumerate(pivot_columns):
        if rows[row][1]:
            particular |= 1 << column

    # Basis vectors for the kernel/nullspace.
    kernel_basis = []
    for free_column in free_columns:
        vector = 1 << free_column
        for row, pivot_column in enumerate(pivot_columns):
            if (rows[row][0] >> free_column) & 1:
                vector |= 1 << pivot_column
        kernel_basis.append(vector)

    return particular, kernel_basis, len(pivot_columns), free_columns


def main():
    polynomials, encrypted_key, encrypted_flag = parse_output("output.txt")
    particular, kernel_basis, rank, free_columns = solve_affine_system(
        polynomials, encrypted_key
    )

    print(f"[+] Rank: {rank}")
    print(f"[+] Nullity: {len(kernel_basis)}")
    print(f"[+] Free variables: {[column + 1 for column in free_columns]}")
    print(f"[+] Testing {1 << len(kernel_basis)} candidate keys...")

    for choice in range(1 << len(kernel_basis)):
        solution = particular
        for index, basis_vector in enumerate(kernel_basis):
            if (choice >> index) & 1:
                solution ^= basis_vector

        # Sage Integer.bits() is little-endian, matching x1 = bit 0, x2 = bit 1, ...
        key_integer = solution
        aes_key = hashlib.sha256(str(key_integer).encode()).digest()
        padded_plaintext = aes_ecb_decrypt(aes_key, encrypted_flag)

        try:
            plaintext = pkcs7_unpad(padded_plaintext)
        except ValueError:
            continue

        if plaintext.startswith(b"HTB{"):
            print(f"[+] KEY = {key_integer}")
            print(f"[+] FLAG = {plaintext.decode()}")
            return

    raise SystemExit("[-] No valid flag found")


if __name__ == "__main__":
    main()
```

Salve o arquivo e dê permissão de execução, caso queira executá-lo diretamente:

```bash
chmod +x solve.py
```

---

## 9. Explicação do solver

### 9.1 Leitura do `output.txt`

A função `parse_output()` lê:

- os 137 polinômios;
- os 137 bits da chave criptografada;
- o ciphertext AES em hexadecimal.

```python
public_polynomials = public_key_line[1:-1].split(", ")
encrypted_key = ast.literal_eval(encrypted_key_line)
encrypted_flag = bytes.fromhex(encrypted_flag_line)
```

### 9.2 Linearização dos polinômios

A função `boolean_linearize()` procura termos no formato:

```text
xN^2
xN^4
```

Cada ocorrência ativa ou desativa o coeficiente da variável usando XOR:

```python
mask ^= 1 << bit
```

Isso também trata automaticamente o cancelamento de termos repetidos.

A constante `1`, quando presente, é separada da parte linear:

```python
constant = int(bool(re.search(r"(^| \+ )1($| \+ )", polynomial)))
```

### 9.3 Eliminação de Gauss em GF(2)

Cada equação é armazenada como:

```text
[ máscara dos coeficientes | resultado ]
```

As operações de linha são feitas com XOR:

```python
rows[row][0] ^= pivot_mask
rows[row][1] ^= pivot_rhs
```

No final, o solver encontra:

- colunas pivô;
- colunas livres;
- uma solução particular;
- vetores que formam a base do núcleo.

### 9.4 Teste dos candidatos

Com nullity igual a dois, o solver gera quatro combinações:

```python
for choice in range(1 << len(kernel_basis)):
```

Para cada candidato, ele recria a chave AES:

```python
aes_key = hashlib.sha256(str(key_integer).encode()).digest()
```

Depois descriptografa o ciphertext:

```python
padded_plaintext = aes_ecb_decrypt(aes_key, encrypted_flag)
```

O candidato correto deve possuir padding PKCS#7 válido e começar com:

```text
HTB{
```

---

## 10. Executando o exploit

Com o ambiente virtual ativo, execute:

```bash
python3 solve.py
```

Saída obtida:

```text
[+] Rank: 135
[+] Nullity: 2
[+] Free variables: [134, 137]
[+] Testing 4 candidate keys...
[+] KEY = 110621486878801192554077110255801668498185
[+] FLAG = HTB{e1th3r_gr0bn3r_0r_v4r13ty___1t_st1ll_w0rks!th4nks_f4l4y_f0r_y0ur_4tt4ck_0n_HFE}
```

---

## 11. Flag

```text
HTB{e1th3r_gr0bn3r_0r_v4r13ty___1t_st1ll_w0rks!th4nks_f4l4y_f0r_y0ur_4tt4ck_0n_HFE}
```

---

## 12. Conclusão

O desafio tenta esconder a chave em um sistema de polinômios multivariados inspirado em HFE. Entretanto, todas as entradas são bits e os polinômios são avaliados sobre `GF(2)`.

Nesse corpo:

```text
x² = x⁴ = x
```

Por isso, o sistema inteiro pode ser convertido em álgebra linear e resolvido com eliminação de Gauss usando XOR.

Além disso, um erro lógico faz com que as matrizes de transformação sejam singulares. O sistema possui rank 135 para 137 variáveis, deixando somente duas variáveis livres e quatro chaves candidatas.

A chave correta é identificada ao descriptografar o AES e validar o padding e o prefixo da flag.

### Conceitos aprendidos

- Corpo finito `GF(2)`;
- soma como operação XOR;
- propriedade booleana `x² = x`;
- linearização de polinômios booleanos;
- sistemas lineares binários;
- rank e nullity;
- eliminação de Gauss em `GF(2)`;
- derivação de chave com SHA-256;


```py
#!/usr/bin/env python3
import ast
import hashlib
import re
from pathlib import Path

N = 137


def aes_ecb_decrypt(key: bytes, ciphertext: bytes) -> bytes:
    """Decrypt AES-ECB using either PyCryptodome or cryptography."""
    try:
        from Crypto.Cipher import AES
        return AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
    except ImportError:
        from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
        decryptor = Cipher(algorithms.AES(key), modes.ECB()).decryptor()
        return decryptor.update(ciphertext) + decryptor.finalize()


def pkcs7_unpad(data: bytes, block_size: int = 16) -> bytes:
    if not data:
        raise ValueError("empty plaintext")
    padding = data[-1]
    if padding < 1 or padding > block_size:
        raise ValueError("invalid padding")
    if data[-padding:] != bytes([padding]) * padding:
        raise ValueError("invalid padding")
    return data[:-padding]


def parse_output(path: str):
    lines = Path(path).read_text().splitlines()
    if len(lines) != 3:
        raise ValueError("output.txt must contain exactly 3 lines")

    public_key_line, encrypted_key_line, encrypted_flag_line = lines
    public_polynomials = public_key_line[1:-1].split(", ")
    encrypted_key = ast.literal_eval(encrypted_key_line)
    encrypted_flag = bytes.fromhex(encrypted_flag_line)

    if len(public_polynomials) != N or len(encrypted_key) != N:
        raise ValueError("unexpected public key/ciphertext size")

    return public_polynomials, encrypted_key, encrypted_flag


def boolean_linearize(polynomial: str):
    """
    For bit inputs over GF(2): x^2 = x and x^4 = x.
    Returns the linear coefficient mask and constant term.
    """
    mask = 0

    for variable, _power in re.findall(r"x(\d+)\^(2|4)", polynomial):
        bit = int(variable) - 1
        mask ^= 1 << bit

    constant = int(bool(re.search(r"(^| \+ )1($| \+ )", polynomial)))
    return mask, constant


def solve_affine_system(polynomials, ciphertext):
    # Each row stores [coefficient bit-mask, right-hand side].
    rows = []
    for polynomial, output_bit in zip(polynomials, ciphertext):
        mask, constant = boolean_linearize(polynomial)
        rows.append([mask, output_bit ^ constant])

    # Reduced row-echelon form over GF(2).
    pivot_columns = []
    pivot_row = 0

    for column in range(N):
        candidate = next(
            (row for row in range(pivot_row, N) if (rows[row][0] >> column) & 1),
            None,
        )
        if candidate is None:
            continue

        rows[pivot_row], rows[candidate] = rows[candidate], rows[pivot_row]
        pivot_mask, pivot_rhs = rows[pivot_row]

        for row in range(N):
            if row != pivot_row and ((rows[row][0] >> column) & 1):
                rows[row][0] ^= pivot_mask
                rows[row][1] ^= pivot_rhs

        pivot_columns.append(column)
        pivot_row += 1

    for mask, rhs in rows:
        if mask == 0 and rhs:
            raise ValueError("inconsistent linear system")

    free_columns = [column for column in range(N) if column not in pivot_columns]

    # Particular solution: set all free variables to zero.
    particular = 0
    for row, column in enumerate(pivot_columns):
        if rows[row][1]:
            particular |= 1 << column

    # Basis vectors for the kernel/nullspace.
    kernel_basis = []
    for free_column in free_columns:
        vector = 1 << free_column
        for row, pivot_column in enumerate(pivot_columns):
            if (rows[row][0] >> free_column) & 1:
                vector |= 1 << pivot_column
        kernel_basis.append(vector)

    return particular, kernel_basis, len(pivot_columns), free_columns


def main():
    polynomials, encrypted_key, encrypted_flag = parse_output("output.txt")
    particular, kernel_basis, rank, free_columns = solve_affine_system(
        polynomials, encrypted_key
    )

    print(f"[+] Rank: {rank}")
    print(f"[+] Nullity: {len(kernel_basis)}")
    print(f"[+] Free variables: {[column + 1 for column in free_columns]}")
    print(f"[+] Testing {1 << len(kernel_basis)} candidate keys...")

    for choice in range(1 << len(kernel_basis)):
        solution = particular
        for index, basis_vector in enumerate(kernel_basis):
            if (choice >> index) & 1:
                solution ^= basis_vector

        # Sage Integer.bits() is little-endian, matching x1 = bit 0, x2 = bit 1, ...
        key_integer = solution
        aes_key = hashlib.sha256(str(key_integer).encode()).digest()
        padded_plaintext = aes_ecb_decrypt(aes_key, encrypted_flag)

        try:
            plaintext = pkcs7_unpad(padded_plaintext)
        except ValueError:
            continue

        if plaintext.startswith(b"HTB{"):
            print(f"[+] KEY = {key_integer}")
            print(f"[+] FLAG = {plaintext.decode()}")
            return

    raise SystemExit("[-] No valid flag found")
# Solve.py

if __name__ == "__main__":
    main()
```
- descriptografia AES-ECB;
- padding PKCS#7.
