# What the Shard Displayed
# Resolvido por Joaov1t

## Explicação inicial

O arquivo `capture(1).sr` é uma captura lógica criada pelo **sigrok/PulseView**.

Diferente do desafio anterior, essa captura utiliza dois canais digitais. O comportamento dos sinais mostra uma comunicação **I²C**, formada por:

```text
D0 = SCL
D1 = SDA
```

Durante a análise, encontramos três dispositivos no barramento:

```text
0x3C = display OLED SSD1306
0x68 = relógio RTC
0x50 = memória EEPROM
```

O dispositivo principal é o display no endereço `0x3C`. O microcontrolador envia três imagens completas para ele:

1. O desenho de um olho.
2. O horário `05:17`.
3. A flag dividida em duas linhas.

Para resolver o desafio, precisamos decodificar o I²C, extrair os bytes enviados ao SSD1306 e reconstruir seus framebuffers de `128x64` pixels.

---

## 1. Identificando o arquivo

Primeiro, verificamos o tipo do arquivo:

```bash
file 'capture(1).sr'
```

Saída:

```text
capture(1).sr: Zip archive data
```

Arquivos `.sr` do sigrok são arquivos ZIP contendo os dados brutos da captura e seus metadados.

Podemos listar seu conteúdo com:

```bash
unzip -l 'capture(1).sr'
```

Saída relevante:

```text
version
metadata
logic-1-1
logic-1-2
logic-1-3
...
```

---

## 2. Verificando os metadados

Para ler os metadados sem extrair o arquivo:

```bash
unzip -p 'capture(1).sr' metadata
```

Saída:

```ini
[global]
sigrok version=0.5.2

[device 1]
capturefile=logic-1
total probes=8
samplerate=2 MHz
total analog=0
probe1=D0
probe2=D1
unitsize=1
```

Com isso, identificamos:

```text
Sample rate: 2 MHz
Canal D0: clock
Canal D1: dados
```

---

## 3. Identificando o protocolo no PulseView

Abra a captura:

```bash
pulseview 'capture(1).sr'
```

No PulseView, adicione o decoder **I²C** e configure:

```text
SCL: D0
SDA: D1
```

Ao analisar os endereços, aparecem:

```text
0x3C
0x68
0x50
```

O endereço `0x3C` recebe a seguinte sequência de inicialização:

```text
AE D5 80 A8 3F D3 00 40 8D 14 20 00
A1 C8 DA 12 81 CF D9 F1 DB 40 A4 A6 AF
```

Essa sequência é característica do controlador de display **SSD1306**.

O display possui:

```text
128 x 64 pixels
```

Como cada byte representa 8 pixels verticais:

```text
128 x 64 / 8 = 1024 bytes
```

Portanto, cada imagem completa enviada ao display possui `1024` bytes.

---

## 4. Criando o solver

O script abaixo:

1. Abre o arquivo `.sr` como ZIP.
2. Junta todos os blocos `logic-1-*`.
3. Decodifica o protocolo I²C.
4. Localiza as escritas no endereço `0x3C`.
5. Extrai os framebuffers do SSD1306.
6. Converte cada framebuffer para uma imagem PBM.

Crie o script:

```bash
cat > solve.py << 'EOF'
#!/usr/bin/env python3

import sys
import zipfile
from pathlib import Path

SCL_BIT = 0
SDA_BIT = 1

OLED_ADDRESS = 0x3C
FRAME_SIZE = 1024
WIDTH = 128
HEIGHT = 64


def part_number(name: str) -> int:
    try:
        return int(name.rsplit("-", 1)[1])
    except (IndexError, ValueError):
        return 0


def read_capture(path: Path) -> bytes:
    with zipfile.ZipFile(path, "r") as archive:
        parts = sorted(
            (
                name
                for name in archive.namelist()
                if name.startswith("logic-1-")
            ),
            key=part_number,
        )

        if not parts:
            raise RuntimeError(
                "Nenhum bloco logic-1-* foi encontrado."
            )

        return b"".join(
            archive.read(name)
            for name in parts
        )


def get_level(sample: int, bit: int) -> int:
    return (sample >> bit) & 1


def decode_i2c(samples: bytes):
    transactions = []

    active = False
    bits = []
    current_bytes = []

    previous_scl = get_level(samples[0], SCL_BIT)
    previous_sda = get_level(samples[0], SDA_BIT)

    for position in range(1, len(samples)):
        sample = samples[position]

        scl = get_level(sample, SCL_BIT)
        sda = get_level(sample, SDA_BIT)

        # START: SDA cai enquanto SCL está alto.
        if (
            previous_sda == 1
            and sda == 0
            and scl == 1
        ):
            if active and current_bytes:
                transactions.append(current_bytes)

            active = True
            bits = []
            current_bytes = []

        # STOP: SDA sobe enquanto SCL está alto.
        elif (
            previous_sda == 0
            and sda == 1
            and scl == 1
        ):
            if active and current_bytes:
                transactions.append(current_bytes)

            active = False
            bits = []
            current_bytes = []

        # Os dados são lidos na borda de subida do clock.
        elif (
            active
            and previous_scl == 0
            and scl == 1
        ):
            bits.append(sda)

            # 8 bits de dados + 1 bit de ACK.
            if len(bits) == 9:
                value = 0

                for bit in bits[:8]:
                    value = (value << 1) | bit

                ack = bits[8]
                current_bytes.append((value, ack))
                bits = []

        previous_scl = scl
        previous_sda = sda

    return transactions


def extract_frames(transactions):
    frames = []
    framebuffer = bytearray()
    collecting = False

    for transaction in transactions:
        values = [
            value
            for value, _ in transaction
        ]

        if len(values) < 2:
            continue

        address_byte = values[0]
        address = address_byte >> 1
        read_write = address_byte & 1

        if address != OLED_ADDRESS or read_write != 0:
            continue

        control_byte = values[1]
        payload = values[2:]

        # Comando que define todas as colunas e páginas:
        # 21 00 7F 22 00 07
        if (
            control_byte == 0x00
            and payload == [
                0x21, 0x00, 0x7F,
                0x22, 0x00, 0x07,
            ]
        ):
            framebuffer = bytearray()
            collecting = True
            continue

        # 0x40 indica dados gráficos.
        if control_byte == 0x40 and collecting:
            framebuffer.extend(payload)

            if len(framebuffer) >= FRAME_SIZE:
                frames.append(
                    bytes(framebuffer[:FRAME_SIZE])
                )

                framebuffer = bytearray()
                collecting = False

    return frames


def framebuffer_to_pixels(framebuffer: bytes):
    pixels = [
        [0 for _ in range(WIDTH)]
        for _ in range(HEIGHT)
    ]

    for index, value in enumerate(framebuffer):
        page = index // WIDTH
        x = index % WIDTH

        for bit in range(8):
            y = page * 8 + bit

            if y < HEIGHT:
                pixels[y][x] = (
                    value >> bit
                ) & 1

    return pixels


def save_pbm(pixels, output_path: Path):
    # Formato PBM binário P4.
    with output_path.open("wb") as file:
        file.write(
            f"P4\n{WIDTH} {HEIGHT}\n".encode()
        )

        for row in pixels:
            for start in range(0, WIDTH, 8):
                value = 0

                for bit in range(8):
                    # No PBM, 1 representa pixel preto.
                    # Invertemos para manter fundo preto
                    # e desenho branco ao visualizar.
                    pixel = row[start + bit]
                    value = (
                        value << 1
                    ) | (0 if pixel else 1)

                file.write(bytes([value]))


def main():
    if len(sys.argv) != 2:
        print(
            f"Uso: {sys.argv[0]} capture.sr"
        )
        raise SystemExit(1)

    capture_path = Path(sys.argv[1])

    if not capture_path.is_file():
        print(
            f"[-] Arquivo não encontrado: "
            f"{capture_path}"
        )
        raise SystemExit(1)

    samples = read_capture(capture_path)
    transactions = decode_i2c(samples)
    frames = extract_frames(transactions)

    print(
        f"[+] Samples carregados: "
        f"{len(samples)}"
    )

    print(
        f"[+] Transações I2C: "
        f"{len(transactions)}"
    )

    print(
        f"[+] Framebuffers encontrados: "
        f"{len(frames)}"
    )

    for index, framebuffer in enumerate(
        frames,
        start=1,
    ):
        pixels = framebuffer_to_pixels(
            framebuffer
        )

        output_path = Path(
            f"frame_{index}.pbm"
        )

        save_pbm(pixels, output_path)

        print(
            f"[+] Imagem salva: "
            f"{output_path}"
        )


if __name__ == "__main__":
    main()
EOF
```

---

## 5. Executando o solver

Execute:

```bash
python3 solve.py 'capture(1).sr'
```

Saída:

```text
[+] Samples carregados: 4000000
[+] Transações I2C: 200
[+] Framebuffers encontrados: 3
[+] Imagem salva: frame_1.pbm
[+] Imagem salva: frame_2.pbm
[+] Imagem salva: frame_3.pbm
```

O script cria três arquivos:

```text
frame_1.pbm
frame_2.pbm
frame_3.pbm
```

---

## 6. Visualizando as imagens

Para abrir a primeira imagem:

```bash
xdg-open frame_1.pbm
```

Ela mostra o desenho de um olho.

Para abrir a segunda:

```bash
xdg-open frame_2.pbm
```

Ela mostra:

```text
05:17
```

Para abrir a terceira:

```bash
xdg-open frame_3.pbm
```

Ela mostra a flag dividida em duas linhas:

```text
HTB{3v3ry_crow_
w3ars_h3r_3y3s}
```

Também podemos converter a imagem para PNG usando o ImageMagick:

```bash
magick frame_3.pbm frame_3.png
```

E abrir:

```bash
xdg-open frame_3.png
```

---

## 7. Flag

Juntando as duas linhas exibidas no último framebuffer:

```text
HTB{3v3ry_crow_w3ars_h3r_3y3s}
```
