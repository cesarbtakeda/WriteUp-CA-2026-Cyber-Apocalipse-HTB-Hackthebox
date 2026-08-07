# The Forged Signet

## Informações do desafio

| Campo | Valor |
|---|---|
| Plataforma | Hack The Box |
| Categoria | Quantum |
| Desafio | The Forged Signet |
| Alvo utilizado | `http://154.57.164.69:31958` |
| Tamanho do registro | 64 qubits |
| Objetivo | Recuperar o segredo binário `s` e submetê-lo ao endpoint `/api/forge` |

> O endereço da instância pode mudar quando o container for reiniciado. Caso isso aconteça, basta alterar a variável `BASE_URL` utilizada nos comandos e no solver.

---

## Descrição

> A long time ago the Brine Signet was the one thing nobody could fake; it was how the Crown proved a seal was real. Then it broke, and everything went bad. Every seal is still checked against a secret called the First Mark, a hidden string `s` that lives inside the verifier. The verifier has a flaw it was never meant to have: if two inputs differ by exactly the Mark, it cannot tell them apart, so `f(x)` and `f(x XOR s)` always look the same to it. The Registry hid the verifier behind a quantum oracle and swore the Mark was safe. It is not.

### Tradução

Há muito tempo, o **Brine Signet** era a única coisa que ninguém conseguia falsificar; era assim que a Coroa provava que um selo era verdadeiro.

Depois ele foi quebrado, e tudo começou a dar errado.

Cada selo ainda é verificado com um segredo chamado **First Mark**, uma sequência oculta `s` que existe dentro do verificador.

O verificador possui uma falha: se duas entradas diferirem exatamente pelo valor do segredo, ele não consegue distingui-las. Portanto:

```text
f(x) = f(x XOR s)
```

O Registro escondeu o verificador atrás de um oráculo quântico e afirmou que o segredo estava seguro.

Ele não está.

---

# 1. Reconhecimento da aplicação

Ao acessar a página principal, a interface informa:

```text
REGISTER: 64 QUBITS
PROMISE: f(x) = f(x XOR s)
SECRET NON-ZERO FIRST MARK s
```

Também são apresentados três componentes:

```text
ORACLE
RESONANCE
FORGE
```

Eles representam as três etapas do desafio:

```text
ORACLE     -> consultar os detalhes do problema
RESONANCE  -> executar o circuito e coletar medições
FORGE      -> enviar o segredo recuperado
```

A própria aplicação disponibiliza os endpoints:

```text
GET  /api/oracle
POST /api/run
POST /api/forge
```

Defini a URL do alvo em uma variável para facilitar os próximos comandos:

```bash
export BASE_URL='http://154.57.164.69:31958'
```

---

# 2. Consulta ao datasheet do oracle

O endpoint `/api/oracle` apresenta as informações do circuito:

```bash
curl -s "$BASE_URL/api/oracle" | jq
```

A estrutura relevante é equivalente a:

```text
each run:

H^n
U_f
measure & discard(output)
L
measure(input)
```

A camada `L` é escolhida pelo usuário e é aplicada individualmente a todos os qubits antes da medição final.

As opções apresentadas pela aplicação são:

```text
I
X
Y
Z
H
S
SDG
```

---

# 3. Identificação do problema de Simon

A propriedade:

```text
f(x) = f(x XOR s)
```

corresponde diretamente à promessa matemática do **problema de Simon**.

O segredo procurado é uma sequência binária não nula:

```text
s ∈ {0,1}^64
```

O circuito começa aplicando uma porta Hadamard em todos os qubits:

```text
H^n
```

Isso cria uma superposição de todas as possíveis entradas `x`.

Em seguida, o oracle `U_f` calcula a função:

```text
|x>|0> -> |x>|f(x)>
```

Como:

```text
f(x) = f(x XOR s)
```

as entradas `x` e `x XOR s` ficam relacionadas à mesma saída.

Quando o registrador de saída é medido e descartado, o registrador de entrada colapsa para um estado proporcional a:

```text
|x> + |x XOR s>
```

Para obter informação sobre `s`, deve-se aplicar novamente uma porta Hadamard em todos os qubits.

Portanto, a camada correta é:

```text
L = H
```

Depois da segunda camada de Hadamard, cada resultado medido `y` satisfaz:

```text
y · s = 0 mod 2
```

Isso significa que cada medição fornece uma equação linear sobre os bits do segredo.

---

# 4. Entendendo uma medição

Em um exemplo reduzido de quatro bits, considere:

```text
y = 1011
s = s0 s1 s2 s3
```

O produto interno binário é:

```text
y · s = (1&s0) XOR (0&s1) XOR (1&s2) XOR (1&s3)
```

Portanto:

```text
s0 XOR s2 XOR s3 = 0
```

Outra medição:

```text
y = 0110
```

produziria:

```text
s1 XOR s2 = 0
```

No desafio, o mesmo processo acontece com 64 variáveis:

```text
s0, s1, s2, ..., s63
```

As medições formam um sistema linear:

```text
M × s = 0
```

As operações são realizadas em `GF(2)`, onde:

```text
0 + 0 = 0
0 + 1 = 1
1 + 0 = 1
1 + 1 = 0
```

Ou seja, a soma de linhas é implementada usando XOR.

---

# 5. Teste manual do endpoint `/api/run`

Primeiro, fiz uma execução pequena para verificar o formato da resposta:

```bash
curl -s -X POST "$BASE_URL/api/run" \
  -H 'Content-Type: application/json' \
  -d '{"layer":"H","shots":8}' | jq
```

Depois, solicitei o máximo de medições permitido pela interface:

```bash
curl -s -X POST "$BASE_URL/api/run" \
  -H 'Content-Type: application/json' \
  -d '{"layer":"H","shots":256}' \
  | tee measurements.json \
  | jq
```

Cada sequência binária de 64 bits retornada representa uma linha da matriz `M`.

Como algumas medições podem se repetir ou ser linearmente dependentes, o solver pode consultar o endpoint mais de uma vez até atingir rank 63.

Para um segredo de 64 bits, uma matriz com rank 63 deixa apenas uma direção não nula possível no espaço nulo. Essa direção corresponde ao segredo `s`.

---

# 6. Criação do solver

O processo matemático poderia ser realizado manualmente, mas isso exigiria eliminação gaussiana em aproximadamente 63 equações com 64 variáveis.

Por isso, foi automatizada apenas a parte repetitiva da álgebra linear.

Criei o solver:

```bash
cat > solve.py << 'EOF'
#!/usr/bin/env python3

import json
import re
import sys
import urllib.error
import urllib.request


BASE_URL = (
    sys.argv[1].rstrip("/")
    if len(sys.argv) > 1
    else "http://154.57.164.69:31958"
)

N_BITS = 64
BITSTRING_RE = re.compile(r"^[01]{64}$")


def post_json(path: str, payload: dict) -> object:
    """
    Envia uma requisição POST com JSON para a API.
    """

    data = json.dumps(payload).encode()

    request = urllib.request.Request(
        BASE_URL + path,
        data=data,
        headers={
            "Content-Type": "application/json",
            "Accept": "application/json",
        },
        method="POST",
    )

    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            raw = response.read().decode()

    except urllib.error.HTTPError as error:
        body = error.read().decode(errors="replace")
        raise RuntimeError(
            f"HTTP {error.code} em {path}:\n{body}"
        ) from error

    except urllib.error.URLError as error:
        raise RuntimeError(
            f"Não foi possível acessar {BASE_URL}: {error}"
        ) from error

    try:
        return json.loads(raw)

    except json.JSONDecodeError as error:
        raise RuntimeError(
            f"A API não retornou JSON:\n{raw}"
        ) from error


def extract_bitstrings(obj: object) -> list[str]:
    """
    Procura sequências binárias de 64 bits em qualquer parte do JSON.

    Aceita respostas em formato de lista ou histograma.
    """

    results: list[str] = []

    if isinstance(obj, str):
        if BITSTRING_RE.fullmatch(obj):
            results.append(obj)

    elif isinstance(obj, list):
        for item in obj:
            results.extend(extract_bitstrings(item))

    elif isinstance(obj, dict):
        for key, value in obj.items():
            if isinstance(key, str) and BITSTRING_RE.fullmatch(key):
                results.append(key)

            results.extend(extract_bitstrings(value))

    return results


def rref_gf2(
    rows: list[int],
    n_bits: int
) -> tuple[list[int], list[int]]:
    """
    Executa eliminação gaussiana em GF(2).

    Cada linha binária é armazenada como um inteiro.
    A soma entre linhas é realizada usando XOR.
    """

    matrix = list(
        dict.fromkeys(
            row for row in rows
            if row != 0
        )
    )

    rank = 0
    pivots: list[int] = []

    for column in range(n_bits - 1, -1, -1):
        pivot_index = next(
            (
                index
                for index in range(rank, len(matrix))
                if (matrix[index] >> column) & 1
            ),
            None,
        )

        if pivot_index is None:
            continue

        matrix[rank], matrix[pivot_index] = (
            matrix[pivot_index],
            matrix[rank],
        )

        for index in range(len(matrix)):
            if index != rank and (
                (matrix[index] >> column) & 1
            ):
                matrix[index] ^= matrix[rank]

        pivots.append(column)
        rank += 1

        if rank == len(matrix):
            break

    return matrix[:rank], pivots


def nullspace_gf2(
    rows: list[int],
    n_bits: int
) -> list[int]:
    """
    Calcula uma base para o espaço nulo de M × s = 0.
    """

    reduced_rows, pivots = rref_gf2(rows, n_bits)

    pivot_set = set(pivots)

    free_columns = [
        column
        for column in range(n_bits)
        if column not in pivot_set
    ]

    vectors: list[int] = []

    for free_column in free_columns:
        vector = 1 << free_column

        for row, pivot in zip(reduced_rows, pivots):
            row_without_pivot = row & ~(1 << pivot)

            parity = (
                row_without_pivot & vector
            ).bit_count() & 1

            if parity:
                vector |= 1 << pivot

        vectors.append(vector)

    return vectors


def main() -> None:
    measurements: list[int] = []

    print(f"[*] Alvo: {BASE_URL}")
    print("[*] Consultando o oracle com layer H...")

    rank = 0

    for batch in range(1, 9):
        response = post_json(
            "/api/run",
            {
                "layer": "H",
                "shots": 256,
            },
        )

        bitstrings = extract_bitstrings(response)

        if not bitstrings:
            print("[-] Nenhuma sequência de 64 bits encontrada.")
            print("[*] Resposta recebida:")
            print(json.dumps(response, indent=2))
            sys.exit(1)

        for bitstring in bitstrings:
            value = int(bitstring, 2)

            # A linha totalmente zerada não fornece informação.
            if value != 0:
                measurements.append(value)

        _, pivots = rref_gf2(
            measurements,
            N_BITS,
        )

        rank = len(pivots)

        print(
            f"[+] Lote {batch}: "
            f"{len(bitstrings)} medições, "
            f"rank {rank}/63"
        )

        if rank >= 63:
            break

    if rank < 63:
        print(
            "[-] Não foram obtidas 63 equações "
            "linearmente independentes."
        )
        sys.exit(1)

    nullspace = nullspace_gf2(
        measurements,
        N_BITS,
    )

    if len(nullspace) != 1:
        print(
            "[-] Dimensão inesperada do espaço nulo: "
            f"{len(nullspace)}"
        )
        sys.exit(1)

    secret_value = nullspace[0]
    secret_bits = f"{secret_value:064b}"

    if secret_value == 0:
        print(
            "[-] A solução encontrada foi zero, "
            "mas o datasheet afirma que s é não nulo."
        )
        sys.exit(1)

    valid = all(
        (
            measurement & secret_value
        ).bit_count() % 2 == 0
        for measurement in measurements
    )

    if not valid:
        print(
            "[-] O segredo falhou na validação "
            "das equações y · s = 0."
        )
        sys.exit(1)

    print()
    print("[+] First Mark recuperado:")
    print(secret_bits)
    print()
    print(
        "[+] Representação hexadecimal: "
        f"{secret_value:016x}"
    )
    print(
        "[+] Todas as medições satisfazem "
        "y · s = 0."
    )
    print("[*] Enviando o segredo para /api/forge...")

    result = post_json(
        "/api/forge",
        {
            "s": secret_bits,
        },
    )

    print()
    print("[+] Resposta do Forge:")
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
EOF
```

Deixei o arquivo executável:

```bash
chmod +x solve.py
```

---

# 7. Execução do solver

Executei o script apontando para a instância:

```bash
python3 solve.py "$BASE_URL"
```

A saída esperada é semelhante a:

```text
[*] Alvo: http://154.57.164.69:31958
[*] Consultando o oracle com layer H...
[+] Lote 1: 256 medições, rank 63/63

[+] First Mark recuperado:
<SEGREDO_BINARIO_DE_64_BITS>

[+] Representação hexadecimal: <SEGREDO_EM_HEX>
[+] Todas as medições satisfazem y · s = 0.
[*] Enviando o segredo para /api/forge...

[+] Resposta do Forge:
{
  "flag": "HTB{...}"
}
```

O script executa quatro tarefas principais:

```text
1. Consulta /api/run usando a camada H.
2. Extrai as medições binárias de 64 bits.
3. Resolve M × s = 0 com eliminação gaussiana em GF(2).
4. Envia o segredo recuperado para /api/forge.
```

---

# 8. Validação manual do segredo

Antes de enviar o resultado, o solver valida a condição:

```text
y · s = 0 mod 2
```

para todas as medições coletadas.

No código, isso é feito por:

```python
valid = all(
    (
        measurement & secret_value
    ).bit_count() % 2 == 0
    for measurement in measurements
)
```

A expressão:

```python
measurement & secret_value
```

mantém apenas as posições em que `y` e `s` possuem bit `1`.

Depois:

```python
.bit_count() % 2
```

calcula a paridade.

Para que a equação seja válida, a quantidade de bits ativos deve ser par:

```text
paridade = 0
```

---

# 9. Submissão manual ao Forge

Caso fosse necessário enviar o segredo manualmente, o comando seria:

```bash
SECRET='<SEGREDO_BINARIO_DE_64_BITS>'

curl -s -X POST "$BASE_URL/api/forge" \
  -H 'Content-Type: application/json' \
  -d "{\"s\":\"$SECRET\"}" | jq
```

O campo `s` deve conter exatamente 64 caracteres binários:

```text
0 ou 1
```

Exemplo de verificação do tamanho:

```bash
printf '%s' "$SECRET" | wc -c
```

A saída deve ser:

```text
64
```

---

# 10. Resultado

Após recuperar e enviar corretamente o **First Mark**, o endpoint `/api/forge` retorna a flag.

```text
Flag: HTB{COLOCAR_A_FLAG_AQUI}
```

---

# Conclusão

O desafio implementa o problema de Simon.

A falha fundamental é a existência de um período secreto `s` na função:

```text
f(x) = f(x XOR s)
```

A segunda camada de Hadamard faz com que as medições retornadas sejam ortogonais ao segredo:

```text
y · s = 0 mod 2
```

Cada medição gera uma equação linear. Com equações suficientes, forma-se uma matriz binária de rank 63, cujo espaço nulo possui uma única solução não nula.

A parte quântica do desafio foi utilizada para gerar as relações matemáticas, enquanto a eliminação gaussiana em `GF(2)` foi automatizada para evitar o cálculo manual de uma matriz de 64 colunas.

O fluxo final foi:

```text
Identificar o problema de Simon
        ↓
Selecionar a camada H
        ↓
Coletar medições em /api/run
        ↓
Montar M × s = 0
        ↓
Resolver o espaço nulo em GF(2)
        ↓
Validar y · s = 0
        ↓
Enviar s para /api/forge
        ↓
Obter a flag
```
