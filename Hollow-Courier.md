# Hollow Courier — HTB Secure Coding Write-up
## Resolvido por Joaov1t


> **Categoria:** Secure Coding  
> **Aplicação:** Flask + Gunicorn atrás de Caddy  
> **Repositório:** `core_application`  
> **Branch de entrega:** `developer`  
> **Instância utilizada:** `http://154.57.164.82:31600`  
> **Status final:** `SOLVED`  
> **Hard Score:** `60/60`

---

## 1. Descrição do desafio

### Texto original

> Lysa Harrowmere has intercepted a Vaultrune route packet ordering the Ashguard away from a Crownspire supply road. She warns Garran Voss that one of tonight's couriers is carrying the order through Salt Gate under a convincing writ. If the decree is sealed before the watch changes, Vaultrune gains an undefended road into Crownspire and another lie acquires the force of law. Garran must identify the compromised passage, uncover how it survived the gate's checks, and stop it without blocking genuine inner-yard runners.
>
> **Objective:** Restore trustworthy forwarded-client provenance across the Caddy and Flask boundary while preserving the legitimate internal route.

### Tradução

Lysa Harrowmere interceptou um pacote de rota dos Vaultrune ordenando que a Ashguard abandonasse uma estrada de abastecimento de Crownspire.

Ela avisa Garran Voss de que um dos mensageiros daquela noite está levando a ordem pelo Portão de Sal usando um decreto falsificado convincente. Caso o decreto seja selado antes da troca da guarda, os Vaultrune ganharão uma estrada desprotegida para Crownspire, e outra mentira receberá força de lei.

Garran precisa identificar a passagem comprometida, descobrir como ela sobreviveu às verificações do portão e impedir o abuso sem bloquear os mensageiros internos legítimos.

### Objetivo técnico

Restaurar uma origem confiável para o endereço do cliente encaminhado entre o **Caddy** e o **Flask**, mantendo funcional a rota interna legítima.

---

## 2. Entendendo o Secure Coding do HTB

Este desafio não exigia obter uma shell ou explorar o host diretamente.

O fluxo era:

1. Clonar o repositório fornecido pelo HTB.
2. Trabalhar exclusivamente na branch `developer`.
3. Localizar a vulnerabilidade no código e na configuração.
4. Criar uma correção segura.
5. Executar os testes locais.
6. Fazer commit e push.
7. O HTB criar uma Pull Request automaticamente.
8. O avaliador compilar, testar e pontuar o patch.
9. Recuperar a flag pelo endpoint `/flag`.

A arquitetura da aplicação era:

```text
Cliente externo
      |
      v
Proxy/edge privado da infraestrutura
      |
      v
Caddy :8000
      |
      v
Gunicorn + Flask em 127.0.0.1:5000
```

O Caddy era responsável por encaminhar o endereço original do cliente para o Flask por meio de cabeçalhos como:

```http
X-Forwarded-For
X-Real-IP
X-Forwarded-Proto
```

O Flask utilizava `ProxyFix` para reconstruir `request.remote_addr`.

---

## 3. Preparação do ambiente

Foi criado um script para clonar o projeto e entrar corretamente na branch de desenvolvimento.

```bash
cat > 01_clonar_hollow.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

TARGET="154.57.164.82:31600"
BASE="$HOME/CTF/Hollow-Courier"
REPO="$BASE/core_application"

mkdir -p "$BASE"
cd "$BASE"

if [[ ! -d "$REPO/.git" ]]; then
    git clone \
        "http://htb_developer:HTBDeveloperPassword@${TARGET}/git/core_application.git"
fi

cd "$REPO"
git fetch origin

if git show-ref --verify --quiet refs/heads/developer; then
    git checkout developer
else
    git checkout -b developer origin/developer
fi

echo
echo "===== BRANCH ATUAL ====="
git branch --show-current

echo
echo "===== ESTRUTURA ====="
find checkpoint -maxdepth 4 -type f | sort
EOF

chmod +x 01_clonar_hollow.sh
./01_clonar_hollow.sh
```

O comando:

```bash
git checkout developer
```

já criou uma branch local rastreando `origin/developer`.

Por isso, executar depois:

```bash
git checkout -b developer origin/developer
```

retornava:

```text
fatal: a branch named 'developer' already exists
```

Esse erro era inofensivo. A branch já existia e estava correta.

O comando `tree` também não estava instalado. Foi substituído por:

```bash
find checkpoint -maxdepth 4 -type f | sort
```

---

## 4. Enumeração e análise do código

Para localizar rapidamente os componentes ligados ao proxy e à origem do cliente:

```bash
cat > 02_analisar_codigo.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
cd "$ROOT"

echo "===== CADDYFILE ====="
sed -n '1,200p' checkpoint/conf/Caddyfile

echo
echo "===== APPLICATION FACTORY ====="
sed -n '1,180p' checkpoint/app/__init__.py

echo
echo "===== CONTROLE DE ORIGEM ====="
sed -n '1,220p' checkpoint/app/gate.py

echo
echo "===== ROTAS DO PORTÃO ====="
sed -n '1,260p' checkpoint/app/routes.py

echo
echo "===== REFERÊNCIAS IMPORTANTES ====="
grep -RInE \
    'ProxyFix|trusted_proxies|trusted_proxies_strict|X-Forwarded|remote_addr|INTERNAL_NETWORKS|require_internal|gate/decree' \
    checkpoint/app checkpoint/conf checkpoint/tests || true
EOF

chmod +x 02_analisar_codigo.sh
./02_analisar_codigo.sh
```

---

## 5. Código vulnerável

### 5.1 Caddyfile original

A configuração original era:

```caddyfile
# Application perimeter.

:8000 {
	reverse_proxy 127.0.0.1:5000 {
		# Accept forwarding metadata from the deployment's private proxy tier.
		trusted_proxies private_ranges
		header_up X-Real-IP {remote_host}
		header_up X-Forwarded-Proto {scheme}
	}
}
```

A aplicação aceitava metadados de encaminhamento vindos de proxies privados, mas a cadeia de IPs encaminhados não era tratada de forma estrita da direita para a esquerda.

Em uma infraestrutura na qual o Caddy recebe a conexão por meio de outro proxy privado, um valor fornecido pelo cliente em `X-Forwarded-For` poderia sobreviver até a aplicação.

### 5.2 ProxyFix no Flask

No arquivo `checkpoint/app/__init__.py`:

```python
# Deployment expects an edge cache and the application perimeter.
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=2, x_proto=1, x_host=1)
```

O Flask confiava em dois valores de `X-Forwarded-For`.

Isso só é seguro quando os dois valores são produzidos por componentes confiáveis. Caso um deles seja controlado pelo usuário, o `request.remote_addr` pode ser reconstruído incorretamente.

### 5.3 Verificação de rede interna

No arquivo `checkpoint/app/gate.py`:

```python
INTERNAL_NETWORKS = (
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("127.0.0.2/32"),
)
```

A decisão de acesso era baseada diretamente em `request.remote_addr`:

```python
def _origin_address() -> str:
    return request.remote_addr or ""
```

```python
def is_internal_request() -> bool:
    try:
        origin = ipaddress.ip_address(_origin_address())
    except ValueError:
        return False

    return any(origin in network for network in INTERNAL_NETWORKS)
```

### 5.4 Rota protegida

A rota crítica era:

```python
@gate_bp.route("/gate/decree", methods=["POST"])
def decree():
    if not gate.require_internal():
        abort(403)
```

Ela selava um decreto com autoridade `CROWN` e deveria aceitar somente requisições internas.

---

## 6. Causa raiz da vulnerabilidade

A falha era uma quebra de confiança na origem do cliente:

```text
Atacante
   |
   | X-Forwarded-For: 127.0.0.2
   v
Proxy privado da infraestrutura
   |
   v
Caddy
   |
   v
Flask + ProxyFix(x_for=2)
   |
   v
request.remote_addr reconstruído incorretamente
```

Um atacante podia tentar enviar:

```http
X-Forwarded-For: 127.0.0.2
```

O endereço `127.0.0.2` estava explicitamente autorizado como o alias loopback utilizado pelo relay interno.

O problema não deveria ser corrigido removendo a rota interna ou bloqueando todas as redes privadas, pois o objetivo exigia preservar os mensageiros internos legítimos.

A correção precisava estabelecer uma fronteira clara:

1. O Caddy identifica quais proxies podem fornecer metadados confiáveis.
2. A cadeia encaminhada é analisada da direita para a esquerda.
3. Qualquer `X-Forwarded-For` enviado diretamente pelo usuário é substituído.
4. O Flask confia apenas na cadeia canônica produzida pela infraestrutura.

---

## 7. Requisição usada para testar o spoofing

O payload utilizado para verificar a proteção da rota foi:

```bash
curl -i \
    -X POST \
    -H 'X-Forwarded-For: 127.0.0.2' \
    --data-urlencode 'order=forged supply-road decree' \
    --data-urlencode 'serial=SECURITY-TEST' \
    'http://154.57.164.82:31600/app/gate/decree'
```

Também foram testadas cadeias prefixadas:

```http
X-Forwarded-For: 127.0.0.2, 203.0.113.25
```

```http
X-Forwarded-For: 192.168.50.10, 203.0.113.25
```

A resposta esperada após a correção era:

```text
HTTP/1.1 403 Forbidden
```

---

## 8. Correção implementada

Foi criado um único script para aplicar o patch completo.

```bash
cat > corrigir_hollow.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
cd "$ROOT"

if [[ "$(git branch --show-current)" != "developer" ]]; then
    git checkout developer
fi

echo "[*] Removendo arquivos acidentais..."

rm -f \
    aplicar_fix.sh \
    conferir_fix.sh \
    enviar_fix.sh \
    testar_fix.sh \
    validar_remoto.sh \
    exit

echo "[*] Limpando flags locais do índice Git..."

git update-index --no-assume-unchanged \
    checkpoint/conf/Caddyfile \
    checkpoint/app/__init__.py 2>/dev/null || true

git update-index --no-skip-worktree \
    checkpoint/conf/Caddyfile \
    checkpoint/app/__init__.py 2>/dev/null || true

echo "[*] Escrevendo o Caddyfile seguro..."

cat > checkpoint/conf/Caddyfile <<'CADDY'
# Application perimeter.

{
	servers {
		# Only forwarding metadata received through the private perimeter is
		# considered. Parse appended chains from right to left so an external
		# client cannot control the authoritative address.
		trusted_proxies static private_ranges
		trusted_proxies_strict
	}
}

:8000 {
	reverse_proxy 127.0.0.1:5000 {
		# Replace any incoming chain with Caddy's verified client address.
		# Caddy then appends the immediate trusted perimeter proxy, producing
		# the two-value chain expected by Flask's ProxyFix.
		header_up X-Forwarded-For {client_ip}
		header_up X-Real-IP {client_ip}

		# Preserve the original scheme used at the perimeter.
		header_up X-Forwarded-Proto {scheme}
	}
}
CADDY

echo "[*] Mantendo o ProxyFix alinhado com os dois hops confiáveis..."

python3 <<'PY'
from pathlib import Path

path = Path("checkpoint/app/__init__.py")
lines = path.read_text(encoding="utf-8").splitlines()

found = False

for index, line in enumerate(lines):
    if "app.wsgi_app = ProxyFix(" not in line:
        continue

    if index > 0 and lines[index - 1].lstrip().startswith("#"):
        lines[index - 1] = (
            "    # Trust the verified client and immediate perimeter proxy "
            "forwarded by Caddy."
        )

    lines[index] = (
        "    app.wsgi_app = ProxyFix("
        "app.wsgi_app, x_for=2, x_proto=1, x_host=1)"
    )

    found = True
    break

if not found:
    raise SystemExit(
        "[-] Não encontrei ProxyFix em checkpoint/app/__init__.py"
    )

path.write_text("\n".join(lines) + "\n", encoding="utf-8")
PY

echo "[*] Criando testes de regressão..."

cat > checkpoint/tests/test_provenance.py <<'PY'
"""Regression tests for forwarded-client provenance."""

from pathlib import Path

from werkzeug.middleware.proxy_fix import ProxyFix
from werkzeug.test import Client
from werkzeug.wrappers import Request, Response


@Request.application
def _show_remote_address(request: Request) -> Response:
    """Return the client address reconstructed by ProxyFix."""
    return Response(request.remote_addr or "", mimetype="text/plain")


def _resolved_address(forwarded_for: str) -> str:
    """Resolve a two-hop chain using the production trust count."""
    application = ProxyFix(
        _show_remote_address,
        x_for=2,
        x_proto=0,
        x_host=0,
    )

    client = Client(application, Response)

    response = client.get(
        "/",
        headers={"X-Forwarded-For": forwarded_for},
    )

    return response.get_data(as_text=True)


def test_caddy_uses_strict_trusted_proxy_parsing():
    caddyfile = Path("conf/Caddyfile").read_text(encoding="utf-8")

    assert "trusted_proxies static private_ranges" in caddyfile
    assert "trusted_proxies_strict" in caddyfile
    assert "header_up X-Forwarded-For {client_ip}" in caddyfile
    assert "header_up X-Real-IP {client_ip}" in caddyfile


def test_flask_trusts_exactly_two_forwarded_values():
    source = Path("app/__init__.py").read_text(encoding="utf-8")

    assert "x_for=2" in source
    assert "x_for=1" not in source


def test_verified_internal_runner_is_preserved():
    assert (
        _resolved_address("127.0.0.2, 172.20.0.10")
        == "127.0.0.2"
    )


def test_verified_external_client_remains_external():
    assert (
        _resolved_address("203.0.113.25, 172.20.0.10")
        == "203.0.113.25"
    )
PY

echo "[*] Criando a documentação de segurança..."

cat > SECURITY.md <<'MD'
# Forwarded-client provenance

The application is deployed behind Caddy, with an additional private
perimeter hop potentially connecting to Caddy.

The original configuration trusted forwarding metadata from private peers but
did not enable strict right-to-left parsing. A client-controlled value at the
left side of an appended `X-Forwarded-For` chain could therefore become the
address trusted by Flask.

The corrected boundary works as follows:

1. Caddy accepts forwarding metadata only from the private perimeter.
2. Caddy parses appended chains from right to left.
3. Caddy replaces the incoming chain with its verified `client_ip`.
4. Caddy appends the immediate trusted perimeter address when proxying.
5. Flask trusts exactly those two values with `ProxyFix(x_for=2)`.

Consequently, an external caller cannot prepend a loopback or RFC1918 address,
while a genuine runner originating from `127.0.0.2` remains available to the
internal decree route.
MD

echo "[*] Verificando sintaxe Python e o diff..."

python3 -m py_compile \
    checkpoint/app/__init__.py \
    checkpoint/tests/test_provenance.py

git diff --check

echo
echo "================ STATUS ================"
git status --short

echo
echo "================ DIFF =================="
git diff -- \
    SECURITY.md \
    checkpoint/app/__init__.py \
    checkpoint/conf/Caddyfile \
    checkpoint/tests/test_provenance.py

echo
echo "[+] Patch aplicado."
EOF

chmod +x corrigir_hollow.sh
./corrigir_hollow.sh
```

---

## 9. Explicação do patch

### 9.1 `trusted_proxies static private_ranges`

```caddyfile
trusted_proxies static private_ranges
```

O Caddy passa a reconhecer como proxies confiáveis apenas os peers pertencentes às redes privadas esperadas pela infraestrutura.

### 9.2 `trusted_proxies_strict`

```caddyfile
trusted_proxies_strict
```

A cadeia de endereços encaminhados passa a ser processada estritamente da direita para a esquerda.

Isso é importante porque proxies normalmente acrescentam endereços ao final da cadeia, enquanto um atacante tenta inserir valores falsos no início.

### 9.3 Canonicalização de `X-Forwarded-For`

```caddyfile
header_up X-Forwarded-For {client_ip}
```

O valor recebido do usuário é substituído pelo endereço que o próprio Caddy determinou como cliente válido.

Assim, a aplicação Flask não recebe diretamente uma cadeia arbitrária criada pelo atacante.

### 9.4 `X-Real-IP`

```caddyfile
header_up X-Real-IP {client_ip}
```

O mesmo endereço validado é utilizado no cabeçalho alternativo de IP real.

### 9.5 `ProxyFix(x_for=2)`

```python
app.wsgi_app = ProxyFix(
    app.wsgi_app,
    x_for=2,
    x_proto=1,
    x_host=1,
)
```

O valor `2` foi mantido porque a topologia legítima possuía dois hops de encaminhamento confiáveis:

```text
Cliente validado -> proxy privado imediato
```

Mudar cegamente para `x_for=1` quebraria a reconstrução esperada pela infraestrutura e poderia bloquear a rota interna legítima.

---

## 10. Execução confiável dos testes

Inicialmente, o comando:

```bash
pytest -q
```

retornou:

```text
ModuleNotFoundError: No module named 'app'
```

O problema foi resolvido executando os testes dentro de `checkpoint` e adicionando o diretório ao caminho de importação:

```bash
PYTHONPATH="$PWD" python -m pytest -q
```

Também foi utilizado um banco SQLite temporário diferente em cada execução. Isso evita que alterações feitas por um teste anterior sejam reutilizadas em uma nova execução.

```bash
cat > rodar_testes.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
cd "$ROOT/checkpoint"

echo "[*] Diretório de testes: $PWD"

if [[ ! -d .venv ]]; then
    python3 -m venv .venv
fi

source .venv/bin/activate

python -m pip install --upgrade pip
python -m pip install -r requirements-dev.txt

TEST_DB="$(mktemp /tmp/hollow-courier-tests.XXXXXX.db)"
export GATE_DB="$TEST_DB"

cleanup() {
    rm -f "$TEST_DB"
}

trap cleanup EXIT

echo "[*] Banco temporário: $GATE_DB"

python -m py_compile \
    app/__init__.py \
    app/gate.py \
    app/routes.py \
    tests/test_provenance.py

echo "[*] Executando pytest..."

PYTHONPATH="$PWD" python -m pytest -q

if command -v caddy >/dev/null 2>&1; then
    echo
    echo "[*] Validando Caddyfile..."

    caddy validate \
        --config conf/Caddyfile \
        --adapter caddyfile
else
    echo
    echo "[!] Caddy não está instalado localmente."
    echo "[!] O Caddyfile será validado pelo build do HTB."
fi

echo
echo "[+] Testes concluídos."
EOF

chmod +x rodar_testes.sh
./rodar_testes.sh
```

Resultado obtido:

```text
............................. [100%]
29 passed in 9.83s
```

---

## 11. Falha intermitente em `test_staff_noise_routes_have_workflows`

Ao executar a suíte novamente sem isolar o banco, ocorreu:

```text
FAILED tests/test_pages.py::test_staff_noise_routes_have_workflows
1 failed, 28 passed
```

O teste esperava que a quantidade de `Ink cakes` passasse de `14` para `17`:

```python
assert b">17<" in supplies.data
```

Entretanto, um banco persistente já havia sido alterado na execução anterior. O novo incremento produziu outra quantidade.

Esse erro não estava relacionado à correção do proxy. Era consequência do estado SQLite sendo reutilizado entre execuções.

A solução definitiva foi exportar um `GATE_DB` temporário:

```bash
TEST_DB="$(mktemp /tmp/hollow-courier-tests.XXXXXX.db)"
export GATE_DB="$TEST_DB"
trap 'rm -f "$TEST_DB"' EXIT
```

Dessa forma, cada execução começa com o `seed.sql` limpo.

---

## 12. Conferência antes do commit

```bash
cat > conferir_patch.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
cd "$ROOT"

echo "================ BRANCH ================"
git branch --show-current

echo
echo "================ STATUS ================"
git status --short

echo
echo "================ CADDYFILE ============="
sed -n '1,180p' checkpoint/conf/Caddyfile

echo
echo "================ PROXYFIX =============="
grep -n -B 2 -A 3 \
    'app.wsgi_app = ProxyFix' \
    checkpoint/app/__init__.py

echo
echo "================ TESTES ================"
sed -n '1,240p' checkpoint/tests/test_provenance.py

echo
echo "================ DIFF =================="
git diff -- \
    SECURITY.md \
    checkpoint/app/__init__.py \
    checkpoint/conf/Caddyfile \
    checkpoint/tests/test_provenance.py

echo
echo "================ DIFF CHECK ============"
git diff --check
EOF

chmod +x conferir_patch.sh
./conferir_patch.sh
```

Arquivos alterados:

```text
SECURITY.md
checkpoint/app/__init__.py
checkpoint/conf/Caddyfile
checkpoint/tests/test_provenance.py
```

---

## 13. Commit e envio para o HTB

Para evitar o aviso de identidade automática do Git:

```bash
git config user.name "joaov1t"
git config user.email "joaov1t@local"
```

Foi criado o script de envio:

```bash
cat > enviar_patch.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
cd "$ROOT"

if [[ "$(git branch --show-current)" != "developer" ]]; then
    echo "[-] A branch atual não é developer."
    exit 1
fi

git add \
    SECURITY.md \
    checkpoint/app/__init__.py \
    checkpoint/conf/Caddyfile \
    checkpoint/tests/test_provenance.py

echo "================ STAGED DIFF ==========="
git diff --cached --stat
git diff --cached --check

if git diff --cached --quiet; then
    echo "[-] Nenhuma alteração preparada para commit."
    exit 1
fi

MESSAGE_FILE="$(mktemp)"
trap 'rm -f "$MESSAGE_FILE"' EXIT

cat > "$MESSAGE_FILE" <<'MSG'
fix: restore trusted forwarded-client provenance

Configure Caddy to parse trusted forwarded chains from right to left and
replace client-controlled forwarding metadata with its verified client IP.

Keep Flask aligned with the resulting two-hop chain by trusting exactly the
verified client and immediate perimeter proxy. This blocks forged internal
addresses while preserving the legitimate watch relay.
MSG

git commit -F "$MESSAGE_FILE"

echo
echo "[*] Enviando para origin/developer..."

git push -u origin developer

echo
echo "[+] Push concluído."
echo "[+] Verifique a Pull Request em:"
echo "http://154.57.164.82:31600/pulls"
EOF

chmod +x enviar_patch.sh
./enviar_patch.sh
```

Commit criado:

```text
9fa1887 fix: restore trusted forwarded-client provenance
```

O servidor confirmou o registro automático da Pull Request:

```text
remote: [*] Push on developer branch detected, registering PR event...
remote: [post-receive] PR event registered
```

---

## 14. Testes remotos de spoofing

O primeiro teste remoto retornou `415 Unsupported Media Type`.

Isso ocorreu porque a rota utilizava:

```python
request.form.get("order") or request.json and request.json.get("order")
```

A requisição anterior não fornecia um formulário válido nem um JSON com `Content-Type: application/json`.

A correção foi enviar os parâmetros com:

```bash
--data-urlencode
```

Script final:

```bash
cat > testar_remoto.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

TARGET="${1:-http://154.57.164.82:31600}"

testar_spoofing() {
    local nome="$1"
    local xff="$2"
    local body
    local status

    body="$(mktemp)"

    status="$(
        curl -sS \
            --max-time 15 \
            -o "$body" \
            -w '%{http_code}' \
            -X POST \
            -H "X-Forwarded-For: $xff" \
            --data-urlencode "order=forged supply-road decree" \
            --data-urlencode "serial=SECURITY-TEST" \
            "$TARGET/app/gate/decree"
    )" || {
        echo "[-] Falha ao conectar em $TARGET"
        rm -f "$body"
        return 1
    }

    echo "[$nome]"
    echo "X-Forwarded-For: $xff"
    echo "HTTP: $status"
    head -c 400 "$body"
    echo
    echo

    rm -f "$body"

    if [[ "$status" != "403" ]]; then
        echo "[-] Esperado HTTP 403, recebido HTTP $status"
        return 1
    fi
}

echo "============ TESTES DE SPOOFING ============"

testar_spoofing \
    "loopback forjado" \
    "127.0.0.2"

testar_spoofing \
    "loopback prefixado" \
    "127.0.0.2, 203.0.113.25"

testar_spoofing \
    "rede privada forjada" \
    "10.20.30.40"

testar_spoofing \
    "cadeia privada prefixada" \
    "192.168.50.10, 203.0.113.25"

echo "[+] Todos os payloads externos receberam HTTP 403."

echo
echo "================ FLAG ==================="

if command -v jq >/dev/null 2>&1; then
    curl -sS --max-time 15 "$TARGET/flag" | jq
else
    curl -sS --max-time 15 "$TARGET/flag"
    echo
fi
EOF

chmod +x testar_remoto.sh
./testar_remoto.sh
```

Resultados:

```text
[loopback forjado]
X-Forwarded-For: 127.0.0.2
HTTP: 403
```

```text
[loopback prefixado]
X-Forwarded-For: 127.0.0.2, 203.0.113.25
HTTP: 403
```

```text
[rede privada forjada]
X-Forwarded-For: 10.20.30.40
HTTP: 403
```

```text
[cadeia privada prefixada]
X-Forwarded-For: 192.168.50.10, 203.0.113.25
HTTP: 403
```

Todos os endereços internos falsificados foram rejeitados.

---

## 15. Resultado do avaliador

O endpoint `/flag` retornou:

```json
{
  "STATUS": "SOLVED",
  "FLAG": "HTB{thr33_kn0ck5_0n_s34led_g4t3s_218875e0ea64916eb83b878775682777}",
  "MESSAGE": "Congratulations on getting the flag!",
  "HARD_SCORE": 60,
  "SOFT_SCORE": {
    "code_quality": 12,
    "security_reasoning": 14,
    "patch_correctness": 7
  }
}
```

Pontuação:

| Critério | Pontuação |
|---|---:|
| Hard score | 60/60 |
| Code quality | 12/15 |
| Security reasoning | 14/15 |
| Patch correctness | 7/10 |
| Soft score total | 33/40 |

O mínimo exigido para o soft score era `28/40`. O patch obteve `33/40`.

---

## 16. Flag

```text
HTB{thr33_kn0ck5_0n_s34led_g4t3s_218875e0ea64916eb83b878775682777}
```

---

## 17. Resumo da vulnerabilidade e da correção

### Vulnerabilidade

A aplicação confiava em uma cadeia `X-Forwarded-For` que podia conter dados fornecidos pelo cliente. Como o Flask utilizava o endereço reconstruído para permitir acesso a uma rota interna, um atacante poderia tentar se passar por `127.0.0.2` ou por um IP RFC1918.

### Correção

- Restrição dos proxies confiáveis às redes privadas da infraestrutura.
- Análise estrita da cadeia da direita para a esquerda.
- Substituição do `X-Forwarded-For` pelo `{client_ip}` validado pelo Caddy.
- Substituição do `X-Real-IP` pelo mesmo endereço validado.
- Manutenção do `ProxyFix(x_for=2)` para preservar os dois hops legítimos.
- Testes de regressão para origens internas e externas.
- Documentação da fronteira de confiança em `SECURITY.md`.

### Impacto final

- Clientes externos não conseguem forjar origem interna.
- A rota `/app/gate/decree` retorna `403` para spoofing.
- O relay interno legítimo continua funcionando.
- Todos os requisitos obrigatórios do avaliador foram aprovados.
