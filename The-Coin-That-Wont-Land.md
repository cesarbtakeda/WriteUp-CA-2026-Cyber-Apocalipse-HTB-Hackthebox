# The Coin That Won't Land
### Resolvido por Joaov1t

## Informações do desafio

| Campo | Valor |
|---|---|
| Plataforma | Hack The Box |
| Categoria | Quantum |
| Desafio | The Coin That Won't Land |
| Aplicação | Oathbinding |
| Alvo utilizado | `http://154.57.164.73:30832` |
| Strands por rodada | 8 |
| Rodadas | 32 |
| Bases de medição | Z e X |
| Objetivo | Passar por todas as rodadas do protocolo e recuperar a flag |

> O endereço da instância pode mudar caso o container seja reiniciado. Nesse caso, basta substituir a URL usada nos comandos.

---

# 1. Descrição

> After the realm cracked apart, nobody trusted a word anyone said, so the Registry built the Oathbinding to settle disputes for good. You swear a vow before the court, the Warden seals it in a box nobody can open, and later they call you back and ask what you swore. Lie about it and the box catches you. Lands, titles, old debts, it is all decided this way now, because it is the one thing nobody can argue with.
>
> Or so they say. The court throws question after question at you, round after round, waiting for you to fold. Get through every round without a slip and the confession you drag out is your flag.

## Tradução

Depois que o reino se partiu, ninguém mais confiava na palavra de ninguém. Por isso, o Registro criou o **Oathbinding** para resolver disputas de uma vez por todas.

Você faz um juramento perante a corte, o Guardião o sela em uma caixa que ninguém consegue abrir e, mais tarde, chamam você de volta para perguntar o que foi jurado.

Caso você minta, a caixa o denuncia.

Terras, títulos e dívidas antigas agora são decididos dessa forma, porque supostamente esse é o único sistema contra o qual ninguém consegue argumentar.

Ou pelo menos é isso que dizem.

A corte lança pergunta após pergunta, rodada após rodada, esperando que você cometa algum erro. Passe por todas as rodadas sem falhar e a confissão obtida será sua flag.

---

# 2. Reconhecimento da aplicação

Ao acessar a página, a interface apresenta as características principais do protocolo:

```text
8 strands sealed each round
32 rounds to hold
2 bases, Z and X
1 confession, the deed
```

Portanto, o desafio exige:

```text
8 strands por rodada
32 rodadas
2 bases possíveis: Z e X
```

No total, o protocolo realiza:

```text
8 × 32 = 256 verificações
```

A aba **The Hearing** também apresenta os movimentos de cada rodada:

```text
01 - Commit the vow
02 - The court's toss
03 - Read your half
04 - Open and be believed
```

Essas etapas representam:

```text
1. Criar um compromisso.
2. Receber a escolha da base.
3. Medir a metade mantida.
4. Revelar os resultados e ser verificado.
```

---

# 3. Leitura do protocolo

A própria interface expõe os endpoints disponíveis:

```text
POST /api/new
POST /api/commit
POST /api/peek
POST /api/open
GET  /api/oath
```

O endpoint `/api/oath` apresenta as regras do protocolo.

Defini o endereço do alvo em uma variável:

```bash
export BASE_URL='http://154.57.164.73:30832'
```

Em seguida, consultei o documento completo:

```bash
curl -s "$BASE_URL/api/oath" | jq
```

A documentação informa:

```text
POST /api/new
```

Abre uma nova audiência e retorna um token, a quantidade de strands e o número de rodadas.

```text
POST /api/commit
```

Recebe um circuito para cada strand. Cada strand possui dois qubits chamados `a` e `b`, inicialmente no estado `|00>`.

O cliente mantém o qubit `a`.

O Guardião sela o qubit `b`.

```text
POST /api/peek
```

Mede os qubits `a` na base exigida pela corte:

```text
Z ou X
```

```text
POST /api/open
```

Recebe os resultados declarados pelo cliente. O Guardião mede os qubits `b` na mesma base.

A rodada só é aceita caso todos os valores sejam iguais.

---

# 4. Entendendo a fraqueza

O protocolo tenta funcionar como um esquema de compromisso.

Em um compromisso clássico, o usuário deveria escolher um valor antes de saber qual desafio será feito:

```text
Escolher um valor
        ↓
Selar o valor
        ↓
Receber o desafio
        ↓
Revelar o valor original
```

O problema é que o protocolo permite enviar circuitos quânticos arbitrários envolvendo os qubits `a` e `b`.

Assim, em vez de preparar dois valores clássicos independentes, é possível criar um par emaranhado.

A preparação utilizada foi:

```text
H em a
CX de a para b
```

Na sintaxe aceita pela API:

```json
[
  ["H", "a"],
  ["CX", "a", "b"]
]
```

---

# 5. Preparação do estado de Bell

Cada strand começa em:

```text
|00>
```

Aplicando Hadamard em `a`:

```text
H(a)
```

obtemos:

```text
(|00> + |10>) / √2
```

Depois, aplicando uma porta controlada:

```text
CX(a, b)
```

obtemos:

```text
(|00> + |11>) / √2
```

Esse estado é chamado de:

```text
|Φ+>
```

ou estado de Bell `Phi Plus`.

O ponto importante é que os dois qubits possuem resultados perfeitamente correlacionados nas duas bases utilizadas pelo desafio.

---

# 6. Correlação na base Z

Na base computacional Z:

```text
|Φ+> = (|00> + |11>) / √2
```

Os únicos resultados possíveis são:

```text
a = 0 e b = 0
```

ou:

```text
a = 1 e b = 1
```

Portanto:

```text
a = b
```

sempre que ambos forem medidos na base Z.

---

# 7. Correlação na base X

Na base X, o mesmo estado pode ser escrito como:

```text
|Φ+> = (|++> + |-->) / √2
```

Os únicos resultados possíveis são:

```text
a = + e b = +
```

ou:

```text
a = - e b = -
```

Na representação binária utilizada pela API, os dois lados continuam retornando o mesmo bit.

Portanto:

```text
a = b
```

também quando ambos são medidos na base X.

---

# 8. Estratégia de exploração

Para cada um dos oito strands, preparei um par de Bell:

```json
[
  ["H", "a"],
  ["CX", "a", "b"]
]
```

O fluxo de cada rodada ficou:

```text
Preparar 8 pares de Bell
        ↓
Enviar para /api/commit
        ↓
Receber a base escolhida pela corte
        ↓
Medir os qubits a na mesma base com /api/peek
        ↓
Copiar os resultados obtidos
        ↓
Enviar os mesmos valores para /api/open
        ↓
O Guardião mede os qubits b
        ↓
Os resultados coincidem
```

Não é necessário adivinhar previamente os bits.

A medição dos qubits `a` produz resultados aleatórios, mas os qubits `b`, mantidos pelo Guardião, ficam correlacionados com eles.

---

# 9. Testes manuais dos endpoints

## 9.1 Abrindo uma audiência

```bash
curl -s -X POST "$BASE_URL/api/new" \
  -H 'Content-Type: application/json' \
  -d '{}' | jq
```

A resposta contém valores semelhantes a:

```json
{
  "token": "...",
  "strands": 8,
  "rounds": 32
}
```

O token deve ser reutilizado em todas as chamadas seguintes.

---

## 9.2 Estrutura do compromisso

O corpo enviado ao endpoint `/api/commit` utiliza uma lista de slots.

Cada slot contém o circuito de um strand:

```json
{
  "token": "TOKEN_DA_SESSAO",
  "slots": [
    [
      ["H", "a"],
      ["CX", "a", "b"]
    ]
  ]
}
```

Como o protocolo exige oito strands, esse circuito precisa ser repetido oito vezes.

---

# 10. Criação do solver

Para automatizar as 32 rodadas, criei um script usando apenas bibliotecas nativas do Python.

Isso evita dependências externas como `requests`.

```bash
cat > oathbinding_solver.py << 'PY'
#!/usr/bin/env python3

import json
import re
import sys
import urllib.error
import urllib.request
from typing import Any


BASE_URL = (
    sys.argv[1].rstrip("/")
    if len(sys.argv) > 1
    else "http://154.57.164.73:30832"
)

TIMEOUT = 20
FLAG_RE = re.compile(r"HTB\{[^}]+\}")


def api_post(
    path: str,
    payload: dict[str, Any],
) -> Any:
    """
    Envia uma requisição POST com JSON para a API.
    """

    body = json.dumps(
        payload
    ).encode("utf-8")

    request = urllib.request.Request(
        BASE_URL + path,
        data=body,
        headers={
            "Content-Type": "application/json",
            "Accept": "application/json",
        },
        method="POST",
    )

    try:
        with urllib.request.urlopen(
            request,
            timeout=TIMEOUT,
        ) as response:
            raw = response.read().decode(
                "utf-8",
                errors="replace",
            )

    except urllib.error.HTTPError as error:
        raw = error.read().decode(
            "utf-8",
            errors="replace",
        )

        raise RuntimeError(
            f"HTTP {error.code} em {path}:\n{raw}"
        ) from error

    except urllib.error.URLError as error:
        raise RuntimeError(
            f"Falha ao acessar {BASE_URL + path}: "
            f"{error}"
        ) from error

    try:
        return json.loads(raw)

    except json.JSONDecodeError as error:
        raise RuntimeError(
            f"Resposta não-JSON em {path}:\n{raw}"
        ) from error


def find_value(
    obj: Any,
    names: set[str],
) -> Any | None:
    """
    Procura recursivamente uma chave em uma resposta JSON.
    """

    if isinstance(obj, dict):
        for key, value in obj.items():
            if str(key).lower() in names:
                return value

        for value in obj.values():
            found = find_value(
                value,
                names,
            )

            if found is not None:
                return found

    elif isinstance(obj, list):
        for item in obj:
            found = find_value(
                item,
                names,
            )

            if found is not None:
                return found

    return None


def find_flag(obj: Any) -> str | None:
    """
    Procura uma flag HTB{...} em qualquer parte da resposta.
    """

    if isinstance(obj, str):
        match = FLAG_RE.search(obj)

        if match:
            return match.group(0)

    elif isinstance(obj, dict):
        for value in obj.values():
            flag = find_flag(value)

            if flag:
                return flag

    elif isinstance(obj, list):
        for value in obj:
            flag = find_flag(value)

            if flag:
                return flag

    return None


def parse_token(response: Any) -> str:
    """
    Extrai o token retornado por /api/new.
    """

    token = find_value(
        response,
        {
            "token",
            "session",
            "session_token",
            "hearing_token",
        },
    )

    if not isinstance(token, str) or not token:
        raise RuntimeError(
            "Token não encontrado em /api/new:\n"
            + json.dumps(
                response,
                indent=2,
            )
        )

    return token


def parse_int(
    response: Any,
    names: set[str],
    default: int,
) -> int:
    """
    Extrai números como strands e rounds.
    """

    value = find_value(
        response,
        names,
    )

    try:
        number = int(value)

    except (TypeError, ValueError):
        return default

    if number <= 0:
        return default

    return number


def parse_basis(response: Any) -> str:
    """
    Converte o desafio da corte:

        c = 0 -> base Z
        c = 1 -> base X
    """

    value = find_value(
        response,
        {
            "c",
            "challenge",
            "challenge_bit",
            "basis",
        },
    )

    if isinstance(value, bool):
        return "X" if value else "Z"

    if isinstance(value, int):
        if value == 0:
            return "Z"

        if value == 1:
            return "X"

    if isinstance(value, str):
        normalized = value.strip().upper()

        if normalized in {"0", "Z"}:
            return "Z"

        if normalized in {"1", "X"}:
            return "X"

        match = re.search(
            r"(?:^|\b)([01ZX])(?:\b|$)",
            normalized,
        )

        if match:
            mapping = {
                "0": "Z",
                "1": "X",
                "Z": "Z",
                "X": "X",
            }

            return mapping[
                match.group(1)
            ]

    raise RuntimeError(
        "Base não encontrada em /api/commit:\n"
        + json.dumps(
            response,
            indent=2,
        )
    )


def normalize_bits(
    value: Any,
    expected: int,
) -> list[int] | None:
    """
    Converte strings ou listas binárias para list[int].
    """

    if isinstance(value, str):
        compact = value.strip()

        if re.fullmatch(
            rf"[01]{{{expected}}}",
            compact,
        ):
            return [
                int(bit)
                for bit in compact
            ]

    if isinstance(value, list):
        if len(value) != expected:
            return None

        bits: list[int] = []

        for item in value:
            if isinstance(item, bool):
                bits.append(int(item))

            elif isinstance(item, int):
                if item not in {0, 1}:
                    return None

                bits.append(item)

            elif isinstance(item, str):
                item = item.strip()

                if item not in {"0", "1"}:
                    return None

                bits.append(int(item))

            else:
                return None

        return bits

    return None


def parse_outcomes(
    response: Any,
    strands: int,
) -> list[int]:
    """
    Extrai os resultados retornados por /api/peek.
    """

    preferred_keys = {
        "outcomes",
        "values",
        "results",
        "measurements",
        "bits",
    }

    if isinstance(response, dict):
        for key, value in response.items():
            if str(key).lower() in preferred_keys:
                bits = normalize_bits(
                    value,
                    strands,
                )

                if bits is not None:
                    return bits

        for value in response.values():
            try:
                return parse_outcomes(
                    value,
                    strands,
                )

            except RuntimeError:
                pass

    elif isinstance(response, list):
        bits = normalize_bits(
            response,
            strands,
        )

        if bits is not None:
            return bits

        for item in response:
            try:
                return parse_outcomes(
                    item,
                    strands,
                )

            except RuntimeError:
                pass

    elif isinstance(response, str):
        bits = normalize_bits(
            response,
            strands,
        )

        if bits is not None:
            return bits

    raise RuntimeError(
        "Resultados não encontrados em /api/peek:\n"
        + json.dumps(
            response,
            indent=2,
        )
    )


def main() -> None:
    print(f"[*] Alvo: {BASE_URL}")
    print("[*] Criando uma nova audiência...")

    new_response = api_post(
        "/api/new",
        {},
    )

    token = parse_token(
        new_response
    )

    strands = parse_int(
        new_response,
        {
            "strands",
            "strand_count",
            "slots",
        },
        8,
    )

    rounds = parse_int(
        new_response,
        {
            "rounds",
            "round_count",
        },
        32,
    )

    print(f"[+] Token: {token}")
    print(
        f"[+] Strands por rodada: {strands}"
    )
    print(
        f"[+] Total de rodadas: {rounds}"
    )

    # Cada strand prepara:
    #
    # |Phi+> = (|00> + |11>) / sqrt(2)
    #
    # H(a) seguido de CX(a, b).
    #
    # Os qubits possuem resultados iguais
    # nas bases Z e X.

    slots = [
        [
            ["H", "a"],
            ["CX", "a", "b"],
        ]
        for _ in range(strands)
    ]

    for round_number in range(
        1,
        rounds + 1,
    ):
        print()
        print(
            f"[*] Rodada "
            f"{round_number}/{rounds}"
        )

        print(
            "[*] Preparando pares de Bell..."
        )

        commit_response = api_post(
            "/api/commit",
            {
                "token": token,
                "slots": slots,
            },
        )

        basis = parse_basis(
            commit_response
        )

        print(
            "[+] Base escolhida "
            f"pela corte: {basis}"
        )

        peek_response = api_post(
            "/api/peek",
            {
                "token": token,
                "basis": basis,
            },
        )

        values = parse_outcomes(
            peek_response,
            strands,
        )

        values_string = "".join(
            map(str, values)
        )

        print(
            "[+] Valores medidos "
            f"nos qubits a: {values_string}"
        )

        open_response = api_post(
            "/api/open",
            {
                "token": token,
                "values": values,
            },
        )

        flag = find_flag(
            open_response
        )

        if flag:
            print()
            print(
                "[+] Todas as rodadas "
                "foram concluídas!"
            )
            print(f"[+] Flag: {flag}")
            return

        message = find_value(
            open_response,
            {
                "message",
                "status",
                "result",
            },
        )

        if isinstance(
            message,
            (str, int, bool),
        ):
            print(
                f"[+] Resposta: {message}"
            )

        else:
            print("[+] Rodada aceita.")

    print()
    print(
        "[*] Todas as rodadas terminaram."
    )
    print(
        "[*] A flag não foi localizada "
        "automaticamente na resposta final."
    )


if __name__ == "__main__":
    try:
        main()

    except KeyboardInterrupt:
        print("\n[-] Interrompido.")
        sys.exit(130)

    except RuntimeError as error:
        print(f"\n[-] {error}")
        sys.exit(1)
PY
```

---

# 11. Validação do script

Antes de executar, validei a sintaxe:

```bash
python3 -m py_compile oathbinding_solver.py
```

Quando esse comando não exibe nenhuma mensagem, significa que o arquivo foi interpretado corretamente.

Depois, tornei o script executável:

```bash
chmod +x oathbinding_solver.py
```

Não é necessário utilizar `sudo`, pois o arquivo pertence ao usuário atual.

---

# 12. Execução

Executei o solver apontando para o container:

```bash
python3 oathbinding_solver.py \
  http://154.57.164.73:30832
```

Também seria possível executar diretamente:

```bash
./oathbinding_solver.py \
  http://154.57.164.73:30832
```

A saída segue o formato:

```text
[*] Criando uma nova audiência...
[+] Token: ...
[+] Strands por rodada: 8
[+] Total de rodadas: 32

[*] Rodada 1/32
[*] Preparando pares de Bell...
[+] Base escolhida pela corte: X
[+] Valores medidos nos qubits a: 01011001
[+] Rodada aceita.
```

O processo se repete para todas as rodadas.

---

# 13. Explicação das funções principais

## `api_post()`

Envia requisições JSON para os endpoints:

```python
api_post("/api/new", {})
```

```python
api_post(
    "/api/commit",
    {
        "token": token,
        "slots": slots,
    },
)
```

```python
api_post(
    "/api/peek",
    {
        "token": token,
        "basis": basis,
    },
)
```

```python
api_post(
    "/api/open",
    {
        "token": token,
        "values": values,
    },
)
```

---

## `parse_basis()`

A corte pode retornar o desafio como bit ou nome da base.

O script converte:

```text
0 -> Z
1 -> X
```

Também aceita diretamente:

```text
Z
X
```

---

## `parse_outcomes()`

O endpoint `/api/peek` pode retornar as medições em diferentes formatos.

O parser aceita, por exemplo:

```json
{
  "values": [0, 1, 0, 1, 1, 0, 0, 1]
}
```

ou:

```json
{
  "outcomes": "01011001"
}
```

O resultado é normalizado para:

```python
[0, 1, 0, 1, 1, 0, 0, 1]
```

---

## `find_flag()`

Após cada `/api/open`, o script procura recursivamente por:

```text
HTB{...}
```

Assim, a execução termina automaticamente quando a flag é retornada.

---

# 14. Resultado

Após passar pelas 32 rodadas, a aplicação retornou:

```text
HTB{epr_0ath_0p3ns_b0th_w4ys_d7d71899c79947e391ef7870e1e0ff82}
```

## Flag

```text
HTB{epr_0ath_0p3ns_b0th_w4ys_d7d71899c79947e391ef7870e1e0ff82}
```

---

# 15. Interpretação da flag

A flag contém:

```text
epr_0ath_0p3ns_b0th_w4ys
```

Isso faz referência ao paradoxo EPR e ao fato de o compromisso poder ser aberto nas duas bases:

```text
EPR oath opens both ways
```

A exploração utiliza exatamente um par EPR, ou par de Bell, para produzir correlação nas bases Z e X.

---

# Conclusão

O protocolo deveria obrigar o participante a escolher um valor antes de receber o desafio da corte.

Entretanto, como a API permite construir circuitos envolvendo os qubits mantidos pelo usuário e pelo Guardião, foi possível criar pares emaranhados:

```text
H(a)
CX(a, b)
```

Isso produz:

```text
|Φ+> = (|00> + |11>) / √2
```

Esse estado apresenta correlação perfeita nas duas bases aceitas pelo protocolo:

```text
Z
X
```

Assim, o ataque não precisa prever a escolha da corte.

O fluxo final foi:

```text
Abrir uma audiência
        ↓
Preparar 8 pares de Bell
        ↓
Enviar os circuitos para /api/commit
        ↓
Receber a base Z ou X
        ↓
Medir os qubits a com /api/peek
        ↓
Enviar os mesmos resultados para /api/open
        ↓
Repetir durante 32 rodadas
        ↓
Receber a flag
```

O ponto central da falha é:

```text
o valor não foi realmente fixado no momento do compromisso;
ele permaneceu emaranhado até a escolha da base.
```
