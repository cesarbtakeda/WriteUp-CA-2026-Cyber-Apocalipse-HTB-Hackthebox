# The-Raven-That-Landed-Twice
## Resolvida por Joaov1t
**Contexto da machine**
```elixir
Suncourt mantém seus mensageiros chegando sem que pareçam mensageiros — uma aeronave particular deixa registros que ninguém pensa em comparar. Na noite anterior a três testemunhas do Registro mudarem seus depoimentos, uma aeronave executiva escura deslizou para o Crownspire Executive Field sob uma lista de passageiros lacrada. Mas o comprovante de despacho e a planilha de combustível não estavam lacrados: um funcionário do portão copiou o hex Mode-S da aeronave (43E91C) e seu indicativo de chamada (VLR602).

Miren Vale, a analista do Aerial Witness Desk, suspeita que o mesmo voo foi registrado duas vezes — uma vez no céu sob seu indicativo de chamada, e outra vez no solo sob sua matrícula — para esconder quem realmente o operava. A lista de passageiros está selada pela Ordem de Cortesia do Tribunal CCO-2026-0441, mas os identificadores técnicos permanecem abertos.

Você consegue cruzar referências do Registro de Aeronaves (Aircraft Registry), do Registro de Movimento (Movement Ledger), do Correio de Mensageiros (Courier Mail) e do Navegador Skyglass (Skyglass Browser) para provar que o corvo que pousou duas vezes era uma única aeronave, e expor o operador escondido atrás do nome de uma empresa de leasing?

Use o aplicativo Oath Submission para confirmar suas descobertas e, em seguida, envie a flag no seguinte formato:

HTB{MATRÍCULA_FROM_AERÓDROMO_DE_PARTIDA_TO_POSIÇÃO_DE_ESTACIONAMENTO}

Exemplo: HTB{G-NTWK_FROM_BRINDLE_MOOR_TO_7C}

Por ser OSINT apenas pesquisei usando as ferramentas da propria machine

Usei três ferramentas, Aircraft Registry, Skyglass Browser e Movement Ledger

Iniciei a pesquisa no Skyglass para achar o voo, usei o dominio do "radarbox.co" que é uma plataforma de rastreamento de voos em tempo real

Depois da pesquisa ele abre o Movement Ledger e no Ledger a gente filtra por "43E91C" para mais detalhes do voo
```


<img width="1519" height="604" alt="image" src="https://github.com/user-attachments/assets/de5214bb-c813-407b-a0fa-d0c733d450ce" />

```elixir
Aqui ja resolvemos o desafio, achamos o registro do voo, qual é o departamento e o posição de estacionamento,
Agora e so montar a flag, no exemplo ta  "HTB{REGISTRATION_FROM_DEPARTURE_AERODROME_TO_PARKING_STAND}";
A flag montada no fim fica assim "HTB{2-RUNE_FROM_SUNCOURT_FIELD_TO_4B}"

```
<img width="1599" height="600" alt="image" src="https://github.com/user-attachments/assets/bacda197-dbd8-44b1-bd58-895d9921203b" />
