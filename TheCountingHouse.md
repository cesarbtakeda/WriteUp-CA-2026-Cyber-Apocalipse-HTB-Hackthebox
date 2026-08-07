# The Counting House 
# Resolvido por Joaov1t

> **Categoria:** Quantum  
> **Desafio:** The Counting House  
> **Instância utilizada:** `http://154.57.164.77:32635`  
> **Objetivo:** forjar a nota de entrada, ler o livro selado, vencer o seal-check e liquidar o lote  
> **Clearing price:** `60739`  
> **Status final:** `SOLVED`

---

## 1. Descrição do desafio

### Texto original

> **The Counting House**
>
> Eastreach is where the rich houses go to buy the things that actually matter: lands, titles, old war debts. The Gilded Knife runs the place and built it so you cannot cheat it. You show a note to get a seat, your bid goes into a sealed book nobody reads, and when it is done you pay the winning price exactly. No haggling, no getting out of it.
>
> Or so they say. You cannot afford a seat, so you will fake one. There is a sealed book you are not allowed to read; you will read it anyway. And there is a seal at the end nobody has ever beaten; you are going to beat it, and walk out with the deed.

### Tradução

Eastreach é onde as casas ricas compram aquilo que realmente importa: terras, títulos e antigas dívidas de guerra.

A Gilded Knife administra o local e afirma ter construído um sistema impossível de trapacear. O participante apresenta uma nota para conseguir um assento, deposita seu lance em um livro selado e, ao final, precisa pagar exatamente o preço vencedor.

Entretanto, o objetivo do desafio era:

1. Forjar uma nota de entrada sem possuir o valor exigido.
2. Conseguir um assento no leilão.
3. Ler os lances armazenados no livro quântico selado.
4. Passar por 24 rodadas de verificação do selo.
5. Forjar uma nova nota correspondente ao clearing price.
6. Liquidar o lote e recuperar a escritura contendo a flag.

---

## 2. Visão geral da solução

O desafio possuía quatro etapas principais:

```text
Forjar a nota de entrada
          |
          v
Ler 6 lances de 16 bits
          |
          v
Vencer 24 rodadas com 8 pares de Bell
          |
          v
Forjar a nota do clearing price e liquidar
```

Os resultados encontrados foram:

```text
Nota de entrada: 4919
Perfil do hash: último byte do SHA-256
Ordem dos bits: LSB
Estado exigido: superposição uniforme sobre ker(H_v)
Lances: 6 valores de 16 bits
Seal-check: 24 rodadas e 8 pares de Bell
Clearing price: 60739
```

---

## 3. Preparação do ambiente

```bash
mkdir -p ~/CTF/the-counting-house
cd ~/CTF/the-counting-house
```

O desafio utilizava somente HTTP e circuitos enviados como JSON, então não foi necessário instalar Qiskit ou outro simulador quântico.

As dependências locais eram apenas:

```bash
python3 --version
curl --version
```

---

## 4. Reconhecimento inicial da API

Foi utilizado o script abaixo para baixar a página principal, consultar o mercado e criar uma sessão descartável.

```bash
cat > 01_recon_counting_house.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

TARGET="${1:-http://154.57.164.77:32635}"
TARGET="${TARGET%/}"

OUT="recon_counting_house"
rm -rf "$OUT"
mkdir -p "$OUT"

echo "[*] Alvo: $TARGET"

curl -fsS \
    --connect-timeout 5 \
    --max-time 20 \
    -D "$OUT/headers.txt" \
    "$TARGET/" \
    -o "$OUT/index.html"

echo
echo "===== /api/market ====="

curl -fsS \
    --connect-timeout 5 \
    --max-time 20 \
    "$TARGET/api/market" |
tee "$OUT/market.json" |
python3 -m json.tool

echo
echo "===== /api/new ====="

curl -fsS \
    --connect-timeout 5 \
    --max-time 20 \
    -X POST \
    -H 'Content-Type: application/json' \
    -d '{}' \
    "$TARGET/api/new" |
tee "$OUT/new.json" |
python3 -m json.tool

echo
echo "===== ROTAS E ASSETS ====="

grep -Eo \
    '(/api/[A-Za-z0-9_./-]+|src="[^"]+"|href="[^"]+")' \
    "$OUT/index.html" |
sort -u |
tee "$OUT/routes.txt"

echo
echo "[+] Reconhecimento salvo em $OUT/"
EOF

chmod +x 01_recon_counting_house.sh

./01_recon_counting_house.sh \
  http://154.57.164.77:32635
```

Os endpoints relevantes eram:

```text
GET  /api/market
POST /api/new
POST /api/enter
POST /api/book
POST /api/seal/commit
POST /api/seal/peek
POST /api/seal/open
POST /api/settle
```

---

## 5. Formato dos circuitos

O servidor aceitava circuitos como uma lista de gates.

Gate de um qubit:

```json
["H", 4]
```

Gate controlado:

```json
["CX", 0, 7]
```

Exemplo de circuito completo:

```json
[
  ["H", 0],
  ["H", 1],
  ["CX", 0, 4],
  ["CX", 1, 6]
]
```

Durante o reconhecimento, uma resposta HTTP `200` não significava necessariamente que o circuito havia sido aceito.

O resultado correto de `/api/enter` precisava conter:

```json
{
  "seated": true
}
```

---

## 6. Forjando a nota de entrada

A nota de entrada exigida possuía valor:

```text
4919
```

Para um valor `v`, a matriz de paridade era construída a partir de:

```text
sha256("eastreach-note-v{v}-{i}")
```

O perfil correto identificado foi:

```text
byte_position = 31
bit_order     = lsb
index_start   = 0
state_type    = kernel
reverse_cx    = false
```

Ou seja:

1. Gerar o SHA-256 para `i = 0, 1, 2...`.
2. Selecionar o último byte do digest, índice `31`.
3. Interpretar seus oito bits em ordem LSB.
4. Manter quatro linhas linearmente independentes sobre GF(2).
5. Calcular uma base do kernel da matriz.
6. Preparar uma superposição uniforme sobre esse kernel.

A matriz encontrada para a nota `4919` foi:

```text
1 0 0 0 1 0 0 0
1 0 0 1 0 1 0 1
1 0 0 0 0 1 0 1
0 1 1 1 0 0 1 0
```

O circuito aceito foi:

```json
[
  ["H", 0],
  ["H", 1],
  ["H", 2],
  ["H", 5],
  ["CX", 0, 4],
  ["CX", 0, 7],
  ["CX", 1, 6],
  ["CX", 2, 6],
  ["CX", 5, 7]
]
```

A resposta obtida foi:

```json
{
  "seated": true
}
```

---

## 7. Lendo o livro quântico

O livro continha:

```text
6 bidders
16 bits por lance
```

Cada qubit precisava ser consultado nas bases `Z` e `X`.

A base correta produzia resultados determinísticos:

```text
Z = 16 zeros e 0 uns
```

ou:

```text
X = 0 zeros e 16 uns
```

A base conjugada produzia resultados aproximadamente aleatórios:

```text
Z = 8 zeros e 8 uns
```

O algoritmo utilizado foi:

1. Medir cada posição 16 vezes em `Z`.
2. Medir a mesma posição 16 vezes em `X`.
3. Calcular o viés absoluto:

```text
bias = |quantidade_de_1 - quantidade_de_0|
```

4. Escolher a base com maior viés.
5. Utilizar o resultado majoritário dessa base como o bit verdadeiro.

Na execução final foram recuperados:

| Bidder | Bits lidos | Little-endian | Big-endian |
|---:|---|---:|---:|
| 0 | `1110110101000011` | 49847 | **60739** |
| 1 | `1100100011001010` | 21267 | 51402 |
| 2 | `1000011010100001` | 34145 | 34465 |
| 3 | `1010001110110010` | 19909 | 41906 |
| 4 | `0001111001000111` | 57976 | 7751 |
| 5 | `0011011110100011` | 50668 | 14243 |

As duas interpretações candidatas eram:

```text
Maior little-endian = 57976
Maior big-endian    = 60739
```

A API permitia mais de uma tentativa de settlement, então ambos foram testados.

---

## 8. Vencendo o seal-check

A etapa final exigia:

```text
24 rodadas
8 strands por rodada
```

Para cada strand foi enviado um par de Bell no estado:

```text
|Φ+> = (|00> + |11>) / sqrt(2)
```

Circuito utilizado:

```json
[
  ["H", "a"],
  ["CX", "a", "b"]
]
```

Foram enviados oito desses circuitos em `/api/seal/commit`.

O servidor retornava um challenge numérico:

```json
{
  "challenge": 0,
  "round": 0
}
```

ou:

```json
{
  "challenge": 1,
  "round": 1
}
```

Esse valor precisava ser enviado diretamente para `/api/seal/peek`:

```json
{
  "token": "...",
  "basis": 0
}
```

O endpoint retornava as medições da metade `a` dos pares:

```json
{
  "a_outcomes": [0, 1, 0, 0, 1, 1, 0, 1]
}
```

Como as duas metades de `|Φ+>` possuem correlação perfeita nas bases utilizadas pelo desafio, os mesmos valores eram enviados para `/api/seal/open`:

```json
{
  "token": "...",
  "values": [0, 1, 0, 0, 1, 1, 0, 1]
}
```

As 24 rodadas foram aprovadas.

---

## 9. Erros encontrados durante o desenvolvimento

### 9.1 HTTP 200 interpretado como sucesso

A primeira versão considerava qualquer `200 OK` como uma nota válida.

Entretanto, a API podia retornar:

```json
{
  "seated": false
}
```

A correção foi validar explicitamente:

```python
response.get("seated") is True
```

### 9.2 Byte e ordem incorretos do SHA-256

As primeiras construções utilizavam o primeiro byte do digest.

O perfil correto era:

```text
último byte, posição 31
ordem LSB
```

### 9.3 Livro medido somente em X

A primeira tentativa mediu todas as posições apenas em `X`.

Isso produzia bits aleatórios nas posições codificadas em `Z`.

A correção foi testar as duas bases e selecionar aquela com resposta determinística.

### 9.4 Segundo commit antes de abrir o strand-set

O primeiro `/api/seal/commit` havia funcionado:

```json
{
  "challenge": 0,
  "round": 0
}
```

Como o parser esperava uma base textual, ele não reconheceu o challenge numérico e enviou outro commit.

O servidor respondeu:

```json
{
  "error": "open the current strand-set first"
}
```

A correção foi usar o número retornado diretamente no `/api/seal/peek`.

### 9.5 Campo `a_outcomes`

O parser conhecia somente:

```json
{
  "outcomes": [...]
}
```

Mas o seal-check retornava:

```json
{
  "a_outcomes": [...]
}
```

O parser final passou a aceitar ambos.

---

## 10. Solver final completo

O script abaixo realiza todo o ataque automaticamente:

1. Consulta os parâmetros do mercado.
2. Cria uma sessão.
3. Forja a nota de entrada.
4. Lê todos os lances nas bases corretas.
5. Calcula candidatos little-endian e big-endian.
6. Passa pelas 24 rodadas usando pares de Bell.
7. Forja a nota de settlement.
8. Testa os candidatos e recupera a flag.

```bash
cat > solve_counting_house.py <<'EOF'
#!/usr/bin/env python3
from __future__ import annotations

import hashlib
import json
import sys
import time
import urllib.error
import urllib.request
from pathlib import Path
from typing import Any


TARGET = (
    sys.argv[1]
    if len(sys.argv) > 1
    else "http://154.57.164.77:32635"
).rstrip("/")

BYTE_POSITION = 31
BIT_ORDER = "lsb"
INDEX_START = 0

STAMP = time.strftime("%Y%m%d_%H%M%S")
LOG_FILE = Path(f"counting_house_{STAMP}.jsonl")


def log(event: str, content: Any) -> None:
    with LOG_FILE.open("a", encoding="utf-8") as handle:
        handle.write(
            json.dumps(
                {
                    "event": event,
                    "content": content,
                },
                ensure_ascii=False,
            )
            + "\n"
        )


def api(
    path: str,
    payload: Any | None = None,
    method: str | None = None,
) -> tuple[int, Any]:
    body = None
    headers = {
        "Accept": "application/json",
        "User-Agent": "counting-house-solver/1.0",
    }

    if payload is not None:
        body = json.dumps(
            payload,
            separators=(",", ":"),
        ).encode()

        headers["Content-Type"] = "application/json"

    request = urllib.request.Request(
        TARGET + path,
        data=body,
        headers=headers,
        method=method or (
            "POST"
            if payload is not None
            else "GET"
        ),
    )

    try:
        with urllib.request.urlopen(
            request,
            timeout=30,
        ) as response:
            status = response.status
            raw = response.read().decode(
                "utf-8",
                errors="replace",
            )

    except urllib.error.HTTPError as error:
        status = error.code
        raw = error.read().decode(
            "utf-8",
            errors="replace",
        )

    try:
        parsed: Any = json.loads(raw)
    except json.JSONDecodeError:
        parsed = raw

    log(
        "http",
        {
            "path": path,
            "payload": payload,
            "status": status,
            "response": parsed,
        },
    )

    return status, parsed


def new_token() -> str:
    status, response = api(
        "/api/new",
        {},
        "POST",
    )

    if (
        status != 200
        or not isinstance(response, dict)
        or not response.get("token")
    ):
        raise RuntimeError(
            f"/api/new falhou: HTTP {status}: {response}"
        )

    return str(response["token"])


def gf2_rref(
    rows: list[list[int]],
    columns: int = 8,
) -> tuple[list[list[int]], list[int]]:
    matrix = [
        row[:]
        for row in rows
        if any(row)
    ]

    pivots: list[int] = []
    current_row = 0

    for column in range(columns):
        pivot = next(
            (
                row_index
                for row_index in range(
                    current_row,
                    len(matrix),
                )
                if matrix[row_index][column]
            ),
            None,
        )

        if pivot is None:
            continue

        matrix[current_row], matrix[pivot] = (
            matrix[pivot],
            matrix[current_row],
        )

        for row_index in range(len(matrix)):
            if (
                row_index != current_row
                and matrix[row_index][column]
            ):
                matrix[row_index] = [
                    left ^ right
                    for left, right in zip(
                        matrix[row_index],
                        matrix[current_row],
                    )
                ]

        pivots.append(column)
        current_row += 1

        if current_row == len(matrix):
            break

    return matrix[:current_row], pivots


def gf2_rank(
    rows: list[list[int]],
) -> int:
    _, pivots = gf2_rref(rows)
    return len(pivots)


def nullspace_basis(
    parity_rows: list[list[int]],
) -> list[list[int]]:
    rref, pivots = gf2_rref(parity_rows)

    free_columns = [
        column
        for column in range(8)
        if column not in pivots
    ]

    basis: list[list[int]] = []

    for free_column in free_columns:
        vector = [0] * 8
        vector[free_column] = 1

        for row_index, pivot in enumerate(pivots):
            vector[pivot] = (
                rref[row_index][free_column]
            )

        basis.append(vector)

    return basis


def parity_matrix(
    value: int,
) -> list[list[int]]:
    rows: list[list[int]] = []
    index = INDEX_START

    while len(rows) < 4:
        seed = (
            f"eastreach-note-v{value}-{index}"
        ).encode()

        digest = hashlib.sha256(seed).digest()
        selected = digest[BYTE_POSITION]

        if BIT_ORDER == "lsb":
            row = [
                (selected >> bit) & 1
                for bit in range(8)
            ]
        else:
            row = [
                (selected >> (7 - bit)) & 1
                for bit in range(8)
            ]

        if (
            gf2_rank(rows + [row])
            > gf2_rank(rows)
        ):
            rows.append(row)

        index += 1

    return rows


def span_circuit(
    basis: list[list[int]],
) -> list[list[Any]]:
    systematic, pivots = gf2_rref(basis)

    if len(systematic) != 4:
        raise RuntimeError(
            "A dimensão do kernel não é quatro."
        )

    gates: list[list[Any]] = [
        ["H", pivot]
        for pivot in pivots
    ]

    for row, pivot in zip(
        systematic,
        pivots,
    ):
        for target, enabled in enumerate(row):
            if (
                enabled
                and target != pivot
            ):
                gates.append(
                    [
                        "CX",
                        pivot,
                        target,
                    ]
                )

    return gates


def note_circuit(
    value: int,
) -> list[list[Any]]:
    rows = parity_matrix(value)
    basis = nullspace_basis(rows)
    return span_circuit(basis)


def seated(response: Any) -> bool:
    return (
        isinstance(response, dict)
        and response.get("seated") is True
    )


def open_session(
    entry_value: int,
    attempts: int = 8,
) -> str:
    circuit = note_circuit(entry_value)

    for attempt in range(1, attempts + 1):
        token = new_token()

        status, response = api(
            "/api/enter",
            {
                "token": token,
                "circuit": circuit,
            },
        )

        print(
            f"    entrada {attempt}/{attempts}: "
            f"HTTP={status} resposta={response}"
        )

        if (
            status == 200
            and seated(response)
        ):
            return token

    raise RuntimeError(
        "A nota de entrada não foi aceita."
    )


def outcome_list(
    response: Any,
) -> list[int]:
    if isinstance(response, bool):
        return [int(response)]

    if (
        isinstance(response, int)
        and response in (0, 1)
    ):
        return [response]

    if isinstance(response, str):
        value = response.strip()

        if value in ("0", "1"):
            return [int(value)]

        return []

    if isinstance(response, list):
        bits: list[int] = []

        for item in response:
            extracted = outcome_list(item)

            if extracted:
                bits.extend(extracted)

        return bits

    if isinstance(response, dict):
        for key in (
            "a_outcomes",
            "b_outcomes",
            "outcomes",
            "results",
            "measurements",
            "values",
            "bits",
            "samples",
        ):
            if key not in response:
                continue

            extracted = outcome_list(
                response[key]
            )

            if extracted:
                return extracted

    raise RuntimeError(
        "Não consegui extrair outcomes de: "
        f"{response}"
    )


def read_book(
    token: str,
    rivals: int,
    width: int,
    shots: int = 16,
) -> list[list[int]]:
    book: list[list[int]] = []

    print(
        f"[*] Lendo {rivals} lances de "
        f"{width} bits..."
    )

    for bidder in range(rivals):
        bidder_bits: list[int] = []
        bidder_bases: list[str] = []

        for position in range(width):
            measurements: dict[
                str,
                dict[str, Any],
            ] = {}

            for basis in ("Z", "X"):
                status, response = api(
                    "/api/book",
                    {
                        "token": token,
                        "bidder": bidder,
                        "position": position,
                        "basis": basis,
                        "shots": shots,
                    },
                )

                if status != 200:
                    raise RuntimeError(
                        "/api/book falhou: "
                        f"{response}"
                    )

                values = outcome_list(response)
                ones = sum(values)
                zeros = len(values) - ones

                measurements[basis] = {
                    "ones": ones,
                    "zeros": zeros,
                    "bias": abs(ones - zeros),
                }

            selected_basis = max(
                ("Z", "X"),
                key=lambda basis: measurements[
                    basis
                ]["bias"],
            )

            selected = measurements[
                selected_basis
            ]

            bit = (
                1
                if selected["ones"]
                > selected["zeros"]
                else 0
            )

            bidder_bits.append(bit)
            bidder_bases.append(
                selected_basis
            )

        book.append(bidder_bits)

        little = sum(
            bit << position
            for position, bit
            in enumerate(bidder_bits)
        )

        big = sum(
            bit << (
                width - 1 - position
            )
            for position, bit
            in enumerate(bidder_bits)
        )

        print(
            f"[+] bidder {bidder}: "
            f"bits={''.join(map(str, bidder_bits))} "
            f"bases={''.join(bidder_bases)} "
            f"little={little} big={big}"
        )

    return book


def clearing_candidates(
    book: list[list[int]],
    width: int,
) -> list[int]:
    little_values = [
        sum(
            bit << position
            for position, bit
            in enumerate(bits)
        )
        for bits in book
    ]

    big_values = [
        sum(
            bit << (
                width - 1 - position
            )
            for position, bit
            in enumerate(bits)
        )
        for bits in book
    ]

    candidates: list[int] = []

    for candidate in (
        max(little_values),
        max(big_values),
    ):
        if candidate not in candidates:
            candidates.append(candidate)

    print(
        f"[*] Lances little-endian: "
        f"{little_values}"
    )

    print(
        f"[*] Lances big-endian:    "
        f"{big_values}"
    )

    print(
        f"[*] Candidatos: {candidates}"
    )

    return candidates


def response_failed(
    status: int,
    response: Any,
) -> bool:
    if not 200 <= status < 300:
        return True

    if isinstance(response, dict):
        text = json.dumps(response).lower()

        if any(
            word in text
            for word in (
                '"error"',
                "rejected",
                "failed",
                "wrong",
            )
        ):
            return True

    return False


def bell_slots(
    strands: int,
) -> list[list[list[Any]]]:
    bell = [
        ["H", "a"],
        ["CX", "a", "b"],
    ]

    return [
        [
            gate[:]
            for gate in bell
        ]
        for _ in range(strands)
    ]


def pass_seal(
    token: str,
    rounds: int,
    strands: int,
) -> None:
    print(
        f"[*] Passando {rounds} rodadas "
        "do seal-check..."
    )

    for round_number in range(
        1,
        rounds + 1,
    ):
        status, commit_response = api(
            "/api/seal/commit",
            {
                "token": token,
                "slots": bell_slots(strands),
            },
        )

        if response_failed(
            status,
            commit_response,
        ):
            raise RuntimeError(
                f"Commit falhou: {commit_response}"
            )

        if (
            not isinstance(
                commit_response,
                dict,
            )
            or "challenge"
            not in commit_response
        ):
            raise RuntimeError(
                "Challenge ausente: "
                f"{commit_response}"
            )

        challenge = commit_response[
            "challenge"
        ]

        status, peek_response = api(
            "/api/seal/peek",
            {
                "token": token,
                "basis": challenge,
            },
        )

        if response_failed(
            status,
            peek_response,
        ):
            raise RuntimeError(
                f"Peek falhou: {peek_response}"
            )

        kept_values = outcome_list(
            peek_response
        )[:strands]

        if len(kept_values) != strands:
            raise RuntimeError(
                "Quantidade inesperada de "
                "outcomes no peek."
            )

        status, open_response = api(
            "/api/seal/open",
            {
                "token": token,
                "values": kept_values,
            },
        )

        if response_failed(
            status,
            open_response,
        ):
            raise RuntimeError(
                f"Open falhou: {open_response}"
            )

        print(
            f"    rodada "
            f"{round_number:02d}/{rounds}: "
            f"challenge={challenge} "
            f"values={kept_values} OK"
        )


def extract_flag(
    response: Any,
) -> str | None:
    text = json.dumps(
        response,
        ensure_ascii=False,
    )

    start = text.find("HTB{")

    if start == -1:
        return None

    end = text.find("}", start)

    if end == -1:
        return None

    return text[start : end + 1]


def settle(
    token: str,
    candidates: list[int],
) -> tuple[int, Any, str]:
    for attempt, value in enumerate(
        candidates,
        start=1,
    ):
        status, response = api(
            "/api/settle",
            {
                "token": token,
                "circuit": note_circuit(value),
                "value": value,
            },
        )

        print(
            f"    tentativa {attempt}: "
            f"value={value} "
            f"HTTP={status} "
            f"response={response}"
        )

        flag = extract_flag(response)

        if flag:
            return value, response, flag

    raise RuntimeError(
        "Nenhum candidato liquidou o lote."
    )


def main() -> None:
    print(f"[*] Alvo: {TARGET}")
    print(f"[*] Log:  {LOG_FILE.resolve()}")

    status, market = api("/api/market")

    if (
        status != 200
        or not isinstance(market, dict)
    ):
        raise RuntimeError(
            f"/api/market falhou: {market}"
        )

    note_spec = market.get("note", {})
    book_spec = market.get("book", {})
    settlement_spec = market.get(
        "settlement",
        {},
    )

    entry_value = int(
        note_spec.get(
            "entry_value",
            4919,
        )
    )

    rivals = int(
        book_spec.get(
            "rivals",
            6,
        )
    )

    width = int(
        book_spec.get(
            "bits",
            16,
        )
    )

    rounds = int(
        settlement_spec.get(
            "seal_rounds",
            24,
        )
    )

    strands = int(
        settlement_spec.get(
            "seal_strands",
            8,
        )
    )

    print(
        f"[*] Forjando nota de entrada "
        f"de valor {entry_value}..."
    )

    token = open_session(entry_value)

    print(
        f"[+] Assento obtido. Token: {token}"
    )

    book = read_book(
        token,
        rivals,
        width,
    )

    candidates = clearing_candidates(
        book,
        width,
    )

    pass_seal(
        token,
        rounds,
        strands,
    )

    value, response, flag = settle(
        token,
        candidates,
    )

    result = {
        "target": TARGET,
        "clearing_price": value,
        "response": response,
        "flag": flag,
    }

    Path(
        "counting_house_result.json"
    ).write_text(
        json.dumps(
            result,
            indent=2,
            ensure_ascii=False,
        ),
        encoding="utf-8",
    )

    print()
    print(
        "======================================"
    )
    print(
        " DESAFIO RESOLVIDO"
    )
    print(
        "======================================"
    )
    print(
        f"Clearing price: {value}"
    )
    print(
        f"Flag: {flag}"
    )


if __name__ == "__main__":
    try:
        main()

    except Exception as error:
        print(
            f"\n[-] ERRO: {error}",
            file=sys.stderr,
        )

        print(
            f"[-] Log: {LOG_FILE.resolve()}",
            file=sys.stderr,
        )

        raise SystemExit(1)
EOF

chmod +x solve_counting_house.py

python3 -m py_compile \
  solve_counting_house.py

python3 solve_counting_house.py \
  http://154.57.164.77:32635
```

---

## 11. Resultado da execução final

A nota de entrada foi aceita:

```text
[+] Assento obtido
```

Os candidatos encontrados foram:

```text
57976
60739
```

As 24 rodadas do seal-check foram aprovadas:

```text
rodada 01/24 ... OK
rodada 02/24 ... OK
...
rodada 24/24 ... OK
```

A primeira tentativa de settlement falhou:

```text
value=57976
reason=that is not the clearing price
settled=false
```

A segunda tentativa foi aceita:

```text
value=60739
settled=true
```

O resultado final foi salvo como:

```json
{
  "target": "http://154.57.164.77:32635",
  "clearing_price": 60739,
  "response": {
    "deed": "HTB{f0rg3d_r34d_4nd_s3ttl3d_t0_th3_s1lv3r_ec6270b4074c94c226a579949c75e3d8}",
    "settled": true
  },
  "flag": "HTB{f0rg3d_r34d_4nd_s3ttl3d_t0_th3_s1lv3r_ec6270b4074c94c226a579949c75e3d8}"
}
```

---

## 12. Flag

```text
HTB{f0rg3d_r34d_4nd_s3ttl3d_t0_th3_s1lv3r_ec6270b4074c94c226a579949c75e3d8}
```

---

## 13. Resumo técnico

### Nota forjada

A verificação aceitava um estado pertencente ao kernel da matriz de paridade derivada do SHA-256.

O estado correto foi preparado com:

```text
último byte do SHA-256
bits em ordem LSB
superposição uniforme sobre ker(H_v)
```

### Livro selado

Cada bit estava codificado em uma base entre `Z` e `X`.

Medindo nas duas bases, a correta era identificada pela resposta determinística e a conjugada pela distribuição aleatória.

### Seal-check

Foram utilizados oito pares de Bell por rodada.

O challenge numérico foi enviado diretamente ao endpoint de peek, e os `a_outcomes` foram reproduzidos no open graças à correlação do estado `|Φ+>`.

### Settlement

Os bits precisavam ser interpretados em big-endian.

O clearing price correto era:

```text
60739
```

Uma nova nota correspondente a esse valor foi forjada com o mesmo algoritmo de kernel e enviada a `/api/settle`.
