# Wireless Connections
## Resolvido por Joaov1t
## Explicação inicial

O arquivo `firmware.bin` é um dump completo da memória flash de uma placa **ESP32-S3**.

A descrição informa que existe uma “relay key” escondida dentro do dispositivo. Durante a análise inicial, encontramos duas strings suspeitas:

```text
SpyHouse
bugged26
```

Entretanto, essas strings são apenas o nome e a senha de uma rede Wi-Fi usada pelo dispositivo. A flag não está armazenada diretamente em texto claro.

Analisando o aplicativo do firmware, encontramos:

- um bloco criptografado com 32 bytes;
- uma rotina que cria um seed de 24 bits;
- um gerador pseudoaleatório LCG;
- uma operação XOR usada para descriptografar o bloco.

Como o seed possui apenas 24 bits, podemos testar todas as possibilidades até que o texto descriptografado comece com `HTB{`.

---

## 1. Identificando o arquivo

Primeiro, verificamos o arquivo recebido:

```bash
file firmware.bin
```

Saída:

```text
firmware.bin: DOS executable (COM)
```

Essa identificação está incorreta. O comando `file` interpreta os primeiros bytes do bootloader como instruções de um executável DOS.

Verificando o tamanho:

```bash
ls -lh firmware.bin
```

Saída:

```text
-rw-r--r-- 1 user user 8.0M firmware.bin
```

O tamanho de 8 MB é compatível com um dump completo da flash de um ESP32-S3.

---

## 2. Procurando strings no firmware

Podemos iniciar a análise procurando strings legíveis:

```bash
strings -a -t x firmware.bin | grep -E 'SpyHouse|bugged26'
```

Saída:

```text
1012b bugged26
10134 SpyHouse
```

As duas strings aparecem próximas uma da outra dentro da partição do aplicativo:

```text
SSID:     SpyHouse
Password: bugged26
```

A tentativa abaixo não funciona:

```text
HTB{bugged26}
```

Isso acontece porque `bugged26` é apenas a senha da rede Wi-Fi, e não a chave escondida pedida pelo desafio.

---

## 3. Identificando a imagem ESP32

A tabela de partições está localizada no offset `0x8000`.

Podemos visualizar essa região com:

```bash
hexdump -C -s 0x8000 -n 192 firmware.bin
```

A partição principal do aplicativo, chamada `app0`, começa em:

```text
0x10000
```

O cabeçalho da imagem no offset `0x10000` começa com o byte mágico:

```text
E9
```

Esse é o formato de imagem utilizado pelos chips ESP32.

A imagem possui seis segmentos. O primeiro segmento relevante é carregado em:

```text
Endereço virtual: 0x3c0a0020
Offset no arquivo: 0x10020
```

---

## 4. Localizando o bloco criptografado

Durante a análise da função principal no Ghidra, encontramos uma referência para o endereço:

```text
0x3c0b103b
```

Esse endereço pertence ao primeiro segmento da imagem.

Para convertê-lo em offset dentro de `firmware.bin`, usamos:

```text
offset = 0x10020 + (0x3c0b103b - 0x3c0a0020)
offset = 0x2103b
```

Extraindo 32 bytes desse offset:

```bash
dd if=firmware.bin bs=1 skip=$((0x2103b)) count=32 status=none | od -An -tx1
```

Saída:

```text
10 a0 68 e7 5d e6 e7 0f
12 bb b2 f3 ca b4 d2 02
cb de d8 20 0d e5 68 92
42 00 be 14 f0 7b ba 01
```

Em formato hexadecimal contínuo:

```text
10a068e75de6e70f12bbb2f3cab4d202cbded8200de568924200be14f07bba01
```

---

## 5. Entendendo a criptografia

A rotina de descriptografia utiliza um gerador congruencial linear, também chamado de **LCG**.

A atualização do estado é:

```text
state = state * 0x0019660D + 0x3C6EF35F
```

Como o cálculo utiliza um inteiro de 32 bits, o resultado é limitado com:

```text
state &= 0xFFFFFFFF
```

Depois, um byte do estado é usado como chave:

```text
key_byte = (state >> 16) & 0xFF
```

Cada byte do bloco é descriptografado com XOR:

```text
plaintext[i] = ciphertext[i] XOR key_byte
```

A rotina monta o seed usando os três primeiros octetos do endereço IP:

```text
seed = (ip[0] << 16) | (ip[1] << 8) | ip[2]
```

Isso produz um seed de apenas 24 bits:

```text
0x000000 até 0xFFFFFF
```

Como não possuímos a rede original usada pelo dispositivo, podemos testar todos os possíveis seeds e validar o resultado pelo prefixo conhecido:

```text
HTB{
```

---

## 6. Criando o solver

Crie o script:

```bash
cat > solve.py << 'EOF'
#!/usr/bin/env python3

import sys
from pathlib import Path

CIPHERTEXT_OFFSET = 0x2103B
CIPHERTEXT_LENGTH = 32

MULTIPLIER = 0x0019660D
INCREMENT = 0x3C6EF35F
MASK_32 = 0xFFFFFFFF

KNOWN_PREFIX = b"HTB{"


def next_state(state: int) -> int:
    return (
        state * MULTIPLIER + INCREMENT
    ) & MASK_32


def decrypt(ciphertext: bytes, seed: int) -> bytes:
    plaintext = bytearray()
    state = seed

    for encrypted_byte in ciphertext:
        state = next_state(state)
        key_byte = (state >> 16) & 0xFF

        plaintext.append(
            encrypted_byte ^ key_byte
        )

    return bytes(plaintext)


def matches_prefix(
    ciphertext: bytes,
    seed: int,
    prefix: bytes,
) -> bool:
    state = seed

    for index, expected_byte in enumerate(prefix):
        state = next_state(state)
        key_byte = (state >> 16) & 0xFF

        decrypted_byte = (
            ciphertext[index] ^ key_byte
        )

        if decrypted_byte != expected_byte:
            return False

    return True


def find_seed(ciphertext: bytes):
    for seed in range(1 << 24):
        if not matches_prefix(
            ciphertext,
            seed,
            KNOWN_PREFIX,
        ):
            continue

        plaintext = decrypt(ciphertext, seed)

        cleaned = plaintext.rstrip(b"\x00")

        if (
            cleaned.startswith(b"HTB{")
            and cleaned.endswith(b"}")
            and all(
                0x20 <= byte <= 0x7E
                for byte in cleaned
            )
        ):
            return seed, cleaned

    return None, None


def seed_to_ip_prefix(seed: int) -> str:
    first = (seed >> 16) & 0xFF
    second = (seed >> 8) & 0xFF
    third = seed & 0xFF

    return f"{first}.{second}.{third}"


def main():
    if len(sys.argv) != 2:
        print(
            f"Uso: {sys.argv[0]} firmware.bin"
        )
        raise SystemExit(1)

    firmware_path = Path(sys.argv[1])

    if not firmware_path.is_file():
        print(
            f"[-] Arquivo não encontrado: "
            f"{firmware_path}"
        )
        raise SystemExit(1)

    firmware = firmware_path.read_bytes()

    end_offset = (
        CIPHERTEXT_OFFSET
        + CIPHERTEXT_LENGTH
    )

    if len(firmware) < end_offset:
        print(
            "[-] Firmware menor que o "
            "offset esperado."
        )
        raise SystemExit(1)

    ciphertext = firmware[
        CIPHERTEXT_OFFSET:end_offset
    ]

    print(
        f"[+] Bloco criptografado: "
        f"{ciphertext.hex()}"
    )

    print(
        "[+] Testando seeds de 24 bits..."
    )

    seed, plaintext = find_seed(ciphertext)

    if seed is None:
        print(
            "[-] Nenhum seed válido encontrado."
        )
        raise SystemExit(1)

    print(f"[+] Seed encontrado: 0x{seed:06x}")

    print(
        f"[+] Prefixo do IP: "
        f"{seed_to_ip_prefix(seed)}"
    )

    print(
        f"[+] Texto descriptografado: "
        f"{plaintext.decode('ascii')}"
    )


if __name__ == "__main__":
    main()
EOF
```

---

## 7. Executando o solver

Execute:

```bash
python3 solve.py firmware.bin
```

Saída:

```text
[+] Bloco criptografado: 10a068e75de6e70f12bbb2f3cab4d202cbded8200de568924200be14f07bba01
[+] Testando seeds de 24 bits...
[+] Seed encontrado: 0x940a73
[+] Prefixo do IP: 148.10.115
[+] Texto descriptografado: HTB{solv3d_w1th_xt3ns4_dec00mp}
```

O seed encontrado foi:

```text
0x940a73
```

Separando seus três bytes:

```text
0x94 = 148
0x0A = 10
0x73 = 115
```

Portanto, os três primeiros octetos usados pelo firmware eram:

```text
148.10.115
```

---

## 8. Flag

```text
HTB{solv3d_w1th_xt3ns4_dec00mp}
```
