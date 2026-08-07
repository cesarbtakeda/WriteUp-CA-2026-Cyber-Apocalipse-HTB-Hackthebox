# Cinderbound — Engenharia Reversa

## JoaoV1t

O desafio entrega um arquivo chamado `cinderbound.mpy`.

O enunciado fala sobre um juramento selado em uma “máquina estrangeira” e diz para aprender a gramática dessa máquina, encontrar a proteção e entregar a sílaba correta.

Na prática, isso indica que precisamos descobrir qual é o formato do arquivo, analisar o código compilado e encontrar a entrada esperada pela função de validação.

---

## Identificando o arquivo

Primeiro verificamos o tipo do arquivo:

```bash
file cinderbound.mpy
```

O Linux identifica apenas como:

```text
cinderbound.mpy: data
```

Isso não significa que o arquivo esteja corrompido. Apenas indica que o comando `file` não reconheceu automaticamente o formato.

A extensão `.mpy` já é uma pista de que se trata de um arquivo compilado do MicroPython.

Também verificamos o tamanho:

```bash
ls -lh cinderbound.mpy
```

O arquivo possui apenas alguns bytes, então provavelmente contém uma função pequena de validação.

---

## Procurando strings

Usamos o comando `strings` para procurar textos legíveis dentro do arquivo:

```bash
strings -a -n 3 cinderbound.mpy
```

Foram encontradas strings importantes:

```text
judge_src.py
judge
syllable
```

Com isso, podemos imaginar que o código original possuía uma função parecida com:

```python
def judge(syllable):
    ...
```

A função `judge` provavelmente recebe uma entrada chamada `syllable` e verifica se ela é válida.

---

## Verificando o cabeçalho

Para visualizar os primeiros bytes do arquivo:

```bash
hexdump -C cinderbound.mpy | head
```

O arquivo começa com:

```text
4d 06 00 1f
```

O byte `4d` representa a letra `M`, assinatura utilizada nos arquivos `.mpy`.

O byte seguinte, `06`, indica a versão do formato do bytecode.

---

## Desmontando o bytecode

Para analisar o conteúdo, utilizamos a ferramenta oficial `mpy-tool.py` do MicroPython.

Baixamos uma versão compatível:

```bash
git clone --depth 1 --branch v1.19.1 \
https://github.com/micropython/micropython.git
```

Entramos na pasta:

```bash
cd micropython
```

Depois desmontamos o arquivo:

```bash
python3 tools/mpy-tool.py -d ../cinderbound.mpy
```

A saída mostra as instruções executadas pela função.

Entre elas aparecem operações como:

```text
LOAD_GLOBAL len
LOAD_GLOBAL ord
BINARY_OP xor
BINARY_OP mul
BINARY_OP and
LOAD_METHOD append
BINARY_OP eq
```

Isso mostra que o programa:

1. Percorre cada caractere da entrada;
2. Converte o caractere para número usando `ord`;
3. Aplica operações XOR;
4. Salva os resultados em uma lista;
5. Compara essa lista com uma sequência fixa.

---

## Lógica encontrada

A função reconstruída é equivalente a:

```python
def judge(syllable):
    target = [
        57, 129, 154, 31,
        199, 192, 73, 243,
        43, 176, 255, 173,
        54, 203, 67, 15,
    ]

    state = 90
    result = []

    for i in range(len(syllable)):
        character = ord(syllable[i])

        value = (
            character
            ^ state
            ^ ((i * 13) & 0xff)
        )

        state = (state + character) & 0xff
        result.append(value)

    return result == target
```

A lista `target` possui 16 números, indicando que a sílaba correta possui 16 caracteres.

A fórmula utilizada para transformar cada caractere é:

```text
value = character XOR state XOR mask
```

Como XOR é uma operação reversível, podemos inverter a fórmula:

```text
character = value XOR state XOR mask
```

---

## Criando o solver

Criamos o seguinte script:

```bash
cat > solve.py <<'PY'
target = [
    57, 129, 154, 31,
    199, 192, 73, 243,
    43, 176, 255, 173,
    54, 203, 67, 15,
]

state = 90
answer = []

for i, value in enumerate(target):
    mask = (i * 13) & 0xff
    character = value ^ state ^ mask

    answer.append(chr(character))
    state = (state + character) & 0xff

syllable = "".join(answer)

print("[+] Sílaba:", syllable)
print("[+] Flag:", f"HTB{{{syllable}}}")
PY
```

Executamos:

```bash
python3 solve.py
```

A saída foi:

```text
[+] Sílaba: c1nd3rbound_v0w5
[+] Flag: HTB{c1nd3rbound_v0w5}
```

---

## Conclusão

Não foi necessário realizar brute force.

O arquivo continha uma função que transformava cada caractere da entrada utilizando XOR e comparava o resultado com uma lista fixa.

Como XOR é reversível, bastou inverter a operação e recuperar cada caractere da sílaba.

A flag final é:

```text
HTB{c1nd3rbound_v0w5}
```
