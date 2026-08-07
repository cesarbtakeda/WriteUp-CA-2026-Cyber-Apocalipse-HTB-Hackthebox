# Cadence in the Cord
## Resolvido por Joaov1t
## Explicação inicial

O arquivo `capture.sr` é uma captura lógica criada pelo **sigrok/PulseView**.

O desafio possui duas camadas:

1. Uma mensagem normal transmitida por **UART**.
2. Uma mensagem escondida no tempo de silêncio entre cada caractere.

A própria mensagem UART explica o método usado:

- intervalo curto representa `0`;
- intervalo longo representa `1`;
- cada grupo de 8 intervalos forma um caractere.

Portanto, precisamos decodificar o UART, medir a distância entre o início de cada byte e transformar esses intervalos em bits.

---

## 1. Identificando o arquivo

Primeiro, verificamos o tipo do arquivo:

```bash
file capture.sr
```

Saída:

```text
capture.sr: Zip archive data
```

Arquivos `.sr` do sigrok são arquivos ZIP contendo os dados brutos da captura e seus metadados.

Podemos listar o conteúdo com:

```bash
unzip -l capture.sr
```

Saída relevante:

```text
version
metadata
logic-1-1
logic-1-2
logic-1-3
logic-1-4
```

---

## 2. Verificando os metadados

Para ler os metadados sem extrair o arquivo:

```bash
unzip -p capture.sr metadata
```

Saída:

```ini
[global]
sigrok version=0.6.0-git-883c2ac

[device 1]
capturefile=logic-1
total probes=8
samplerate=2 MHz
total analog=0
probe2=D1
unitsize=1
```

Com isso, identificamos:

```text
Canal utilizado: D1
Sample rate: 2 MHz
Tipo de sinal: digital
```

---

## 3. Decodificando o UART no PulseView

Abra a captura:

```bash
pulseview capture.sr
```

No PulseView, adicione um decoder **UART** e configure:

```text
RX: D1
Baud rate: 9600
Data bits: 8
Parity: none
Stop bits: 1
Bit order: LSB first
```

Isso corresponde a:

```text
UART 9600 8N1
```

A mensagem visível começa com:

```text
To the buyer who paid in secrets: what follows is the pleasant tone...
```

No final, ela entrega a lógica da segunda camada:

```text
A long rest raises the mark to one,
a short rest lets it fall to nothing;
count eight rests to every letter.
```

Assim, a mensagem verdadeira não está nos caracteres, mas nos intervalos entre eles.

---

## 4. Entendendo os intervalos

A captura utiliza uma taxa de:

```text
2.000.000 samples por segundo
```

Ao medir a distância entre o início de dois bytes UART consecutivos, encontramos dois grupos:

```text
Intervalo curto: 6094-6095 samples
Intervalo longo: 22097-22098 samples
```

Convertendo para tempo:

```text
6095 / 2.000.000  ≈ 3,05 ms
22098 / 2.000.000 ≈ 11,05 ms
```

Um frame UART 8N1 possui 10 bits:

```text
1 start bit + 8 data bits + 1 stop bit
```

Em 9600 baud, o frame dura aproximadamente:

```text
10 / 9600 ≈ 1,04 ms
```

Retirando o tempo do próprio frame, sobram aproximadamente:

```text
Silêncio curto: 2 ms  → 0
Silêncio longo: 10 ms → 1
```

Os bits escondidos são agrupados de 8 em 8, em ordem **MSB first**.

---

## 5. Criando o solver

Crie o script:

```bash
cat > solve.py << 'EOF'
#!/usr/bin/env python3

import sys
import zipfile
from pathlib import Path

SAMPLE_RATE = 2_000_000
BAUD_RATE = 9600
CHANNEL_BIT = 1  # probe2 = D1


def logic_part_number(name: str) -> int:
    try:
        return int(name.rsplit('-', 1)[1])
    except (IndexError, ValueError):
        return 0


def read_samples(capture_path: Path) -> bytes:
    with zipfile.ZipFile(capture_path, 'r') as archive:
        parts = sorted(
            (
                name
                for name in archive.namelist()
                if name.startswith('logic-1-')
            ),
            key=logic_part_number,
        )

        if not parts:
            raise RuntimeError(
                'Nenhum bloco logic-1-* encontrado no arquivo.'
            )

        return b''.join(archive.read(name) for name in parts)


def find_falling_edges(samples: bytes) -> list[int]:
    edges = []
    previous = (samples[0] >> CHANNEL_BIT) & 1

    for position in range(1, len(samples)):
        current = (samples[position] >> CHANNEL_BIT) & 1

        if previous == 1 and current == 0:
            edges.append(position)

        previous = current

    return edges


def decode_uart(samples: bytes, falling_edges: list[int]):
    samples_per_bit = SAMPLE_RATE / BAUD_RATE
    frames = []
    edge_index = 0

    def level(position: float) -> int:
        index = int(round(position))
        return (samples[index] >> CHANNEL_BIT) & 1

    while edge_index < len(falling_edges):
        start = falling_edges[edge_index]

        start_center = start + 0.5 * samples_per_bit
        stop_center = start + 9.5 * samples_per_bit

        if int(round(stop_center)) >= len(samples):
            break

        # Um frame UART 8N1 válido possui start bit 0
        # e stop bit 1.
        if level(start_center) == 0 and level(stop_center) == 1:
            value = 0

            # UART envia os 8 bits de dados em LSB first.
            for bit_index in range(8):
                bit = level(
                    start
                    + (1.5 + bit_index) * samples_per_bit
                )

                value |= bit << bit_index

            frames.append((start, value))

            # Ignora as transições internas do frame.
            frame_end = start + 9.7 * samples_per_bit

            while (
                edge_index < len(falling_edges)
                and falling_edges[edge_index] < frame_end
            ):
                edge_index += 1
        else:
            edge_index += 1

    return frames


def decode_hidden_message(frames):
    starts = [start for start, _ in frames]

    deltas = [
        right - left
        for left, right in zip(starts, starts[1:])
    ]

    if not deltas:
        raise RuntimeError(
            'Não existem intervalos suficientes.'
        )

    # Os deltas formam dois grupos:
    # intervalos curtos e intervalos longos.
    threshold = (min(deltas) + max(deltas)) / 2

    hidden_bits = [
        1 if delta > threshold else 0
        for delta in deltas
    ]

    decoded = bytearray()

    # Cada grupo de 8 intervalos representa um byte.
    # A ordem da camada escondida é MSB first.
    for offset in range(0, len(hidden_bits), 8):
        group = hidden_bits[offset:offset + 8]

        if len(group) != 8:
            break

        value = 0

        for bit in group:
            value = (value << 1) | bit

        decoded.append(value)

    return decoded.decode('ascii'), deltas, threshold


def main():
    if len(sys.argv) != 2:
        print(f'Uso: {sys.argv[0]} capture.sr')
        raise SystemExit(1)

    capture_path = Path(sys.argv[1])

    if not capture_path.is_file():
        print(f'[-] Arquivo não encontrado: {capture_path}')
        raise SystemExit(1)

    samples = read_samples(capture_path)
    falling_edges = find_falling_edges(samples)
    frames = decode_uart(samples, falling_edges)

    visible_message = ''.join(
        chr(value)
        for _, value in frames
    )

    hidden_message, deltas, threshold = (
        decode_hidden_message(frames)
    )

    short_deltas = [
        delta
        for delta in deltas
        if delta <= threshold
    ]

    long_deltas = [
        delta
        for delta in deltas
        if delta > threshold
    ]

    print(f'[+] Samples carregados: {len(samples)}')
    print(f'[+] Bytes UART: {len(frames)}')

    print(
        '[+] Delta curto: '
        f'{min(short_deltas)}-{max(short_deltas)} samples'
    )

    print(
        '[+] Delta longo: '
        f'{min(long_deltas)}-{max(long_deltas)} samples'
    )

    print('\n[+] Mensagem UART visível:\n')
    print(visible_message)

    print('\n[+] Mensagem escondida nos intervalos:\n')
    print(hidden_message)


if __name__ == '__main__':
    main()
EOF
```

---

## 6. Executando o solver

Execute:

```bash
python3 solve.py capture.sr
```

Saída relevante:

```text
[+] Samples carregados: 16000000
[+] Bytes UART: 593
[+] Delta curto: 6094-6095 samples
[+] Delta longo: 22097-22098 samples

[+] Mensagem escondida nos intervalos:

you read the silence well HTB{th3_f1rst_m4rk_r1ngs_tru3_b3n34th_th3_w0rds}
```

O script realizou as seguintes etapas:

1. Abriu o arquivo `.sr` como ZIP.
2. Juntou os blocos `logic-1-*`.
3. Leu o canal digital `D1`.
4. Encontrou os start bits do UART.
5. Decodificou os caracteres em 9600 baud, 8N1.
6. Mediu o intervalo entre os caracteres.
7. Converteu intervalo curto em `0`.
8. Converteu intervalo longo em `1`.
9. Agrupou os bits de 8 em 8.
10. Converteu os bytes escondidos para ASCII.

---

## 7. Flag

```text
HTB{th3_f1rst_m4rk_r1ngs_tru3_b3n34th_th3_w0rds}
```
