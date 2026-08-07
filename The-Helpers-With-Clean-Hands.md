# HTB OSINT — The Helpers With Clean Hands
## Resolvido por Joaov1t
## Informações do desafio

- **Plataforma:** Hack The Box
- **Categoria:** OSINT
- **Desafio:** The Helpers With Clean Hands
- **Objetivo:** reconstruir a cadeia entre a instituição de caridade, seu fornecedor, a holding controladora e a próxima entrega.
- **Formato da flag:** `HTB{SUPPLIER_FEEDS_TOWN}`

---

## 1. Descrição original

> The war of succession has ended on paper, but the Northern Crown Road still bleeds quietly. Villages that once resisted the Crown's claim now receive "mercy" from the Mercy Lantern Relief Trust — a charity delivering burial candles, lamp oil, and grief counselling to communities broken by the fighting. Behind the trust stands Ash & Wick Provisioners Ltd, a wholesale supplier of ceremonial goods whose invoices are clean, whose warehouse is spotless, and whose registered office is a letterbox on Cinder Lane. The charity's trustees believe they serve the grieving. The supplier's director, Nera Sorn, knows otherwise. Above Sorn sits Quiet Mercy Holdings Ltd, a non-trading shell wholly owned by a fiduciary foundation with restricted disclosure — a structure designed so that the same handful of names can sign both the purchase orders and the relief manifests without any single office ever being answerable. The next delivery is scheduled for Harrowgate. The receiving clerk is Sister Merrow. Somewhere in the tender records, the company filings, and the couriered dispatch notes, the chain of control runs from a charity that teaches mercy to a holding company that appoints the hands that make one possible. An intelligence desk has been set up with access to the Companies Register, the Tender Hall, an archive mirror, and the courier's intercepted mail. Trace the structure end to end and file the findings.
>
> Use the Oath Submission form to confirm your findings, then assemble the flag from the verified answers.

---

## 2. Tradução da descrição

A guerra de sucessão terminou no papel, mas a Northern Crown Road ainda sangra silenciosamente. Vilarejos que resistiram à reivindicação da Coroa agora recebem “misericórdia” da **Mercy Lantern Relief Trust**, uma instituição de caridade que entrega velas funerárias, óleo para lamparinas e aconselhamento de luto às comunidades destruídas pelos combates.

Por trás da instituição está a **Ash & Wick Provisioners Ltd**, fornecedora atacadista de artigos cerimoniais, com notas fiscais limpas, depósito impecável e escritório registrado em uma simples caixa postal na Cinder Lane.

Os administradores da caridade acreditam estar ajudando os enlutados. A diretora da fornecedora, **Nera Sorn**, sabe que existe algo além disso.

Acima dela está a **Quiet Mercy Holdings Ltd**, uma holding sem atividade comercial, pertencente integralmente a uma fundação fiduciária com informações restritas. A estrutura permite que o mesmo pequeno grupo de pessoas assine pedidos de compra e manifestos de ajuda sem que um único escritório seja responsabilizado.

A próxima entrega está programada para **Harrowgate**, e a pessoa responsável pelo recebimento é **Sister Merrow**.

Nos registros de licitações, documentos empresariais e mensagens interceptadas do courier existe uma cadeia de controle que começa em uma caridade e termina em uma holding que nomeia as pessoas responsáveis pelas operações.

O objetivo é usar o **Companies Register**, o **Tender Hall**, o **Archive Mirror** e o **Courier Mail** para reconstruir toda essa cadeia, validar as respostas no formulário **Oath Submission** e montar a flag.

---

## 3. Entendendo o painel de investigação

Ao abrir o endereço do desafio no navegador, somos apresentados a uma mesa de inteligência com vários aplicativos:

- **Analyst Briefing:** resumo inicial e evidências encontradas em campo;
- **Companies Register:** registro de empresas, instituições de caridade, diretores e acionistas;
- **Tender Hall:** registros de licitações e lotes de entrega;
- **Archive Mirror:** cópias atuais e antigas dos sites das organizações;
- **Ledgerglass Browser:** navegador interno para os sites arquivados;
- **Courier Mail:** mensagens de e-mail interceptadas;
- **Evidence Satchel:** local para salvar evidências;
- **Threadboard:** painel para relacionar pessoas e organizações;
- **Submission:** formulário final para validar as respostas.

A resolução pode ser feita inteiramente pela interface, sem força bruta e sem automatizar tentativas.

---

## 4. Evidência inicial

Abrimos o aplicativo **Analyst Briefing**.

A cópia verificada do aviso de aquisição apresenta:

| Campo | Valor |
|---|---|
| Referência | `ML-22-771` |
| Comprador | `Mercy Lantern Relief Trust` |
| Registro do fornecedor | `VR-118204` |
| Descrição | Winter vigil materials and burial assistance equipment |
| Região | Northern Crown Road |
| Situação | Awarded |

A instrução principal é encontrar a empresa por trás do número:

```text
VR-118204
```

O próprio briefing avisa para não confundir a instituição de caridade compradora com a empresa fornecedora.

---

## 5. Identificando o fornecedor comercial

No menu inferior, abrimos o **Companies Register**.

No campo de busca, pesquisamos pelo número obtido no briefing:

```text
VR-118204
```

O registro encontrado é:

```text
Ash & Wick Provisioners Ltd
```

### Dados importantes do registro

| Campo | Valor |
|---|---|
| Número da empresa | `VR-118204` |
| Nome registrado | `Ash & Wick Provisioners Ltd` |
| Nome comercial anterior | `Ashwick Municipal Supply` |
| Situação | Active |
| Data de incorporação | 2024-11-03 |
| Escritório registrado | 14 Cinder Lane, Eastreach |
| Atividade | Artigos cerimoniais, materiais municipais e suprimentos funerários |
| Diretora | `Nera Sorn` |
| Acionista | `Quiet Mercy Holdings Ltd — 100%` |

Assim, a resposta da primeira pergunta é:

```text
Ash & Wick Provisioners Ltd
```

---

## 6. Encontrando a holding controladora

No próprio registro da **Ash & Wick Provisioners Ltd**, a seção **Ownership & Control** informa que 100% das ações pertencem a:

```text
Quiet Mercy Holdings Ltd
```

Podemos clicar no nome da acionista ou pesquisá-la diretamente no **Companies Register**.

O registro correspondente é:

```text
Quiet Mercy Holdings Ltd — VR-091144
```

### Dados importantes da holding

| Campo | Valor |
|---|---|
| Número da empresa | `VR-091144` |
| Nome | `Quiet Mercy Holdings Ltd` |
| Situação | Active |
| Data de incorporação | 2023-06-14 |
| Escritório registrado | 14 Cinder Lane, Eastreach |
| Atividade | Holding sem atividade comercial; gestão de participações em subsidiárias |
| Diretores | `Nera Sorn` e `Ilyra Venn` |
| Acionista | `Crown Road Fiduciary Foundation — 100%` |

O registro também contém um aviso de controle beneficiário informando que **Ilyra Venn** possui direitos de nomeação e remoção sobre o conselho da empresa.

A resposta da segunda pergunta é:

```text
Quiet Mercy Holdings Ltd
```

---

## 7. Identificando a diretora em comum

Agora comparamos os registros das duas empresas.

### Ash & Wick Provisioners Ltd

```text
Director: Nera Sorn
```

### Quiet Mercy Holdings Ltd

```text
Directors: Nera Sorn, Ilyra Venn
```

A pessoa que aparece tanto na fornecedora quanto na holding é:

```text
Nera Sorn
```

Essa é a resposta da terceira pergunta.

---

## 8. Analisando a licitação

Abrimos o aplicativo **Tender Hall** e pesquisamos pela referência obtida no briefing:

```text
ML-22-771
```

O registro apresenta:

| Campo | Valor |
|---|---|
| Comprador | Mercy Lantern Relief Trust |
| Fornecedor | Ash & Wick Provisioners Ltd |
| Registro do fornecedor | `VR-118204` |
| Data da adjudicação | 2026-07-08 |
| Valor | 184,400 silver crowns |
| Região | Northern Crown Road |
| Situação | Awarded |

A licitação está dividida em três lotes:

| Lote | Cidade | Data | Situação | Recebedor | Referência |
|---:|---|---|---|---|---|
| 1 | Riverwake | 2026-07-13 | Completed | Orren Hald | `AWP-RW-4406` |
| 2 | Stoneford | 2026-07-16 | Completed | Caldus Vey | `AWP-SF-4407` |
| 3 | **Harrowgate** | 2026-07-21 | **Scheduled** | **Sister Merrow** | `AWP-HG-4408` |

Os dois primeiros lotes já estão concluídos. Portanto, a próxima entrega é o lote 3.

A resposta da quarta pergunta é:

```text
Harrowgate
```

A resposta da quinta pergunta é:

```text
Sister Merrow
```

---

## 9. Confirmando pelo Archive Mirror

As respostas já podem ser obtidas pelo registro empresarial e pelo Tender Hall, mas o **Archive Mirror** permite confirmar a relação por fontes adicionais.

### 9.1 Mercy Lantern Relief Trust

Abrimos o endereço arquivado:

```text
http://mercylantern.val/
```

A cópia de setembro de 2025 menciona que as entregas eram realizadas por:

```text
Ashwick Municipal Supply
```

Esse é justamente o nome comercial anterior da **Ash & Wick Provisioners Ltd**, encontrado no registro empresarial.

A página antiga da equipe também informa:

```text
Sister Merrow manages delivery acceptance across the northern reach.
```

Ou seja, Sister Merrow era responsável por aceitar as entregas nos lotes regionais.

### 9.2 Ash & Wick Provisioners Ltd

Abrimos:

```text
http://ashwick.val/
```

O site confirma:

- nome: **Ash & Wick Provisioners Ltd**;
- número: `VR-118204`;
- escritório: **14 Cinder Lane, Eastreach**;
- contato da diretoria: `nera.sorn@ashwick.val`.

### 9.3 Quiet Mercy Holdings Ltd

Abrimos:

```text
http://quietmercy.val/
```

O site informa que a organização é uma holding sem atividade comercial e apresenta:

```text
Quiet Mercy Holdings Ltd — VR-091144
```

Também utiliza o endereço:

```text
14 Cinder Lane, Eastreach
```

O mesmo endereço da fornecedora reforça a ligação entre as duas empresas.

---

## 10. Confirmando pelo Courier Mail

Abrimos o aplicativo **Courier Mail** e verificamos as mensagens da caixa de entrada.

### 10.1 E-mail de confirmação de Harrowgate

A mensagem possui o assunto:

```text
AWP-HG-4408 — Harrowgate acceptance
```

Ela foi enviada pela equipe de despacho da **Ash & Wick Provisioners Ltd** para **Sister Merrow**.

O conteúdo confirma que:

- o lote 3 da licitação `ML-22-771` continua programado;
- a entrega será realizada em **Harrowgate**;
- a data é **21 de julho de 2026**;
- Sister Merrow deve assinar o recebimento;
- N. Sorn aprovou a documentação revisada;
- a referência da entrega é `AWP-HG-4408`.

### 10.2 E-mail de aprovação dos lotes

Outra mensagem, enviada por **Nera Sorn** para **Ilyra Venn**, possui o assunto:

```text
Board approval — northern lots
```

Ela confirma que:

- os lotes de Riverwake e Stoneford já estavam confirmados;
- **Harrowgate** era o próximo local ainda não concluído;
- **Sister Merrow** havia sido avisada como responsável pelo recebimento;
- a holding deveria manter em seus registros a estrutura de subcontratação.

Esses e-mails ligam diretamente a fornecedora, a holding, Nera Sorn, Harrowgate e Sister Merrow.

---

## 11. Cadeia completa de controle

Com as informações reunidas, a estrutura fica assim:

```text
Mercy Lantern Relief Trust
        │
        │ compra materiais pela licitação ML-22-771
        ▼
Ash & Wick Provisioners Ltd — VR-118204
        │
        │ 100% das ações
        ▼
Quiet Mercy Holdings Ltd — VR-091144
        │
        │ 100% das ações
        ▼
Crown Road Fiduciary Foundation
```

A pessoa que conecta a fornecedora e a holding é:

```text
Nera Sorn
```

A próxima operação de entrega é:

```text
Lote 3 → Harrowgate → Sister Merrow
```

---

## 12. Cuidado com os falsos positivos

O banco de dados possui várias organizações com nomes semelhantes, criadas para confundir a investigação:

- **Ashwick Candle Company Ltd**;
- **Ash & Cord Municipal Ltd**;
- **Mercy Lantern Foundation**;
- **Mercy Road Relief Trust**;
- **Quiet Lantern Holdings Ltd**;
- **Wick Provisioning Cooperative**.

Não devemos selecionar empresas apenas porque possuem palavras como `Ash`, `Wick`, `Mercy` ou `Lantern` no nome.

O caminho correto deve partir das referências verificadas:

```text
ML-22-771 → VR-118204 → Quiet Mercy Holdings Ltd
```

Também existe uma licitação antiga da Mercy Lantern Relief Trust, `ML-21-334`, com outro fornecedor. Ela não representa a próxima entrega e, portanto, não deve ser usada para montar a flag.

---

## 13. Respostas do Oath Submission

Abrimos o aplicativo **Submission** e respondemos às cinco perguntas.

### Pergunta 1

**Which commercial company supplies the Mercy Lantern Relief Trust?**

```text
Ash & Wick Provisioners Ltd
```

### Pergunta 2

**Which company owns 100% of that supplier?**

```text
Quiet Mercy Holdings Ltd
```

### Pergunta 3

**Which director appears across both the supplier and its holding company?**

```text
Nera Sorn
```

### Pergunta 4

**Which town is scheduled to receive the next delivery under tender ML-22-771?**

```text
Harrowgate
```

### Pergunta 5

**Who is listed as the Lot 3 receiver?**

```text
Sister Merrow
```

Depois que todas as respostas são validadas, o painel apresenta o resumo das evidências:

```text
Commercial supplier: Ash & Wick Provisioners Ltd (VR-118204)
Holding company: Quiet Mercy Holdings Ltd (VR-091144)
Connecting director: Nera Sorn
Next delivery town: Harrowgate — Lot 3, tender ML-22-771
Lot 3 receiver: Sister Merrow
Delivery reference: AWP-HG-4408
Scheduled date: 2026-07-21
```

---

## 14. Montagem da flag

O formato solicitado é:

```text
HTB{SUPPLIER_FEEDS_TOWN}
```

O fornecedor é:

```text
Ash & Wick Provisioners Ltd
```

Convertendo para o formato da flag:

```text
ASH_&_WICK_PROVISIONERS_LTD
```

A cidade é:

```text
Harrowgate
```

Convertendo:

```text
HARROWGATE
```

O caractere `&` deve ser preservado. Ele não deve ser substituído pela palavra `AND`.

Juntando os valores:

```text
HTB{ASH_&_WICK_PROVISIONERS_LTD_FEEDS_HARROWGATE}
```

---

## 15. Flag final

```text
HTB{ASH_&_WICK_PROVISIONERS_LTD_FEEDS_HARROWGATE}
```

---

## 16. Conclusão

O desafio é resolvido reconstruindo a cadeia a partir de uma referência de licitação e de um número empresarial.

O **Tender Hall** identifica a empresa fornecedora e o próximo lote. O **Companies Register** revela a holding proprietária e a diretora compartilhada. O **Archive Mirror** confirma os nomes antigos, os sites e os endereços. Por fim, o **Courier Mail** confirma que Harrowgate é o próximo destino e que Sister Merrow receberá a entrega.

A cadeia final é:

```text
Mercy Lantern Relief Trust
        ↓
Ash & Wick Provisioners Ltd
        ↓
Quiet Mercy Holdings Ltd
```

Com **Nera Sorn** conectando as empresas e o próximo lote destinado a **Harrowgate**, chegamos à flag válida:

```text
HTB{ASH_&_WICK_PROVISIONERS_LTD_FEEDS_HARROWGATE}
```
