# Withered Registry
## Resolvido por Joaov1t

> **Categoria:** Secure Coding  
> **Stack:** Go, HTTP, sessões e HMAC  
> **Repositório:** `core_application`  
> **Branch de entrega:** `developer`  
> **Instância utilizada:** `http://154.57.164.72:31333`  
> **Status final:** `SOLVED`

---

## 1. Descrição do desafio

### Texto original

> **Withered Registry**
>
> Chancellor Veylen Marr summons Rin Kagetsura after a forbidden recognition appears against the Ash-Vault roll, though no authorised Registry hand admits invoking it. At the next council bell, the Registry will seal its new crown roll; if the false recognition remains, Vaultrune can present a manufactured lineage as lawful proof and claim a place in the struggle for the Salt Crown. Rin must reverse the warded slate, follow seven records to discover how the rite entered the Registry, and repair the breach before Veylen seals the roll.

### Tradução

O chanceler Veylen Marr convoca Rin Kagetsura depois que um reconhecimento proibido aparece nos registros da Ash-Vault, embora nenhuma pessoa autorizada do Registro admita tê-lo realizado.

No próximo toque do sino do conselho, o Registro selará sua nova lista real. Caso o reconhecimento falso permaneça, os Vaultrune poderão apresentar uma linhagem fabricada como prova legítima e reivindicar um lugar na disputa pela Coroa de Sal.

Rin precisa reverter a lousa protegida, seguir os registros que revelam como o rito entrou no sistema e corrigir a falha antes que Veylen sele a lista.

### Objetivo técnico

Impedir que uma requisição assinada por uma field slate use dados controlados pelo cliente para escolher qual casa receberá o reconhecimento privilegiado.

A casa afetada deve ser determinada exclusivamente pela sessão autenticada.

---

## 2. Como funciona o Secure Coding do HTB

Neste tipo de desafio, o objetivo não é obter uma shell.

O fluxo é:

```text
Clonar o repositório
        |
        v
Analisar código e testes
        |
        v
Identificar a vulnerabilidade
        |
        v
Criar um patch seguro
        |
        v
Executar testes e build
        |
        v
Commit na branch developer
        |
        v
Push para o servidor HTB
        |
        v
Pull Request automática
        |
        v
Avaliação e flag
```

O servidor cria uma Pull Request automaticamente sempre que um novo commit é enviado para a branch `developer`.

---

## 3. Preparação e clone do repositório

Crie um diretório para o desafio:

```bash
mkdir -p ~/CTF/withered-registry
cd ~/CTF/withered-registry
```

O repositório pode ser clonado com a conta de desenvolvedor apresentada pela plataforma:

```bash
cat > 01_clonar_repositorio.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

TARGET="${1:-http://154.57.164.72:31333}"
TARGET="${TARGET%/}"

BASE="$HOME/CTF/withered-registry"
REPO="$BASE/core_application"

mkdir -p "$BASE"
cd "$BASE"

if [[ ! -d "$REPO/.git" ]]; then
    git clone \
        "${TARGET/http:\/\//http:\/\/htb_developer:HTBDeveloperPassword@}/git/core_application.git" \
        "$REPO"
fi

cd "$REPO"
git fetch origin --prune

if git show-ref --verify --quiet refs/heads/developer; then
    git checkout developer
else
    git checkout -b developer origin/developer
fi

echo
echo "===== BRANCH ATUAL ====="
git branch --show-current

echo
echo "===== BRANCHES ====="
git branch -a

echo
echo "===== ÚLTIMOS COMMITS ====="
git log --all --graph --decorate --oneline -20

echo
echo "===== ARQUIVOS ====="
find . \
    -path './.git' -prune -o \
    -path './node_modules' -prune -o \
    -path './.venv' -prune -o \
    -type f \
    -print |
sort
EOF

chmod +x 01_clonar_repositorio.sh

./01_clonar_repositorio.sh \
  http://154.57.164.72:31333
```

A branch obrigatória para entrega era:

```text
developer
```

O servidor rejeitava pushes diretos para `main`.

---

## 4. Enumeração do código

Os arquivos importantes estavam no backend Go:

```text
registry-backend/
├── cmd/registry/
├── internal/server/
│   ├── routes.go
│   ├── server.go
│   ├── server_test.go
│   └── ...
├── internal/store/
├── go.mod
└── go.sum
```

Foi criado um script para procurar os componentes ligados ao rito de reconhecimento, HMAC, sessão e seleção de casa.

```bash
cat > 02_analisar_codigo.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
cd "$ROOT"

echo "===== ARQUIVOS GO ====="

find registry-backend \
    -type f \
    \( -name '*.go' -o -name 'go.mod' -o -name 'go.sum' \) \
    -print |
sort

echo
echo "===== REFERÊNCIAS IMPORTANTES ====="

grep -RInE \
    'handleRecognise|recogniseRequest|targetHouse|house_id|GrantCrownWrit|principalFrom|houseOf|ownsHouse|requireSlate|HMAC|URL\.Query|RawQuery' \
    registry-backend ||
true

echo
echo "===== ROUTES.GO ====="

sed -n '1,320p' \
    registry-backend/internal/server/routes.go

echo
echo "===== TESTES DO SERVIDOR ====="

sed -n '1,700p' \
    registry-backend/internal/server/server_test.go
EOF

chmod +x 02_analisar_codigo.sh
./02_analisar_codigo.sh
```

---

## 5. Código vulnerável

A rota de reconhecimento recebia um `house_id` controlado pelo cliente:

```go
type recogniseRequest struct {
	HouseID string `json:"house_id"`
}
```

A função `targetHouse()` permitia que o destino viesse da query string ou do corpo JSON:

```go
func targetHouse(r *http.Request, body *recogniseRequest) string {
	if h := strings.TrimSpace(r.URL.Query().Get("house_id")); h != "" {
		return h
	}

	return strings.TrimSpace(body.HouseID)
}
```

O parâmetro da query tinha prioridade.

No handler, a casa escolhida pelo usuário era passada para a operação privilegiada:

```go
var body recogniseRequest

if r.Body != nil {
	_ = json.NewDecoder(r.Body).Decode(&body)
}

house := targetHouse(r, &body)

p, _ := principalFrom(r)

writ, err := s.store.GrantCrownWrit(
	house,
	p.Username,
)
```

A chamada vulnerável era:

```go
GrantCrownWrit(house, p.Username)
```

O valor de `house` vinha da requisição.

---

## 6. Entendendo a vulnerabilidade

A aplicação utilizava duas formas diferentes de confiança:

1. Uma assinatura HMAC da field slate.
2. Uma sessão autenticada do scribe.

A assinatura HMAC provava que a requisição havia sido produzida por um dispositivo que possuía a chave da slate.

Ela não provava que aquele dispositivo ou usuário tinha autorização para agir em nome de qualquer casa enviada no campo `house_id`.

O fluxo vulnerável era:

```text
Scribe autenticado da Rookhold
        |
        | requisição com HMAC válido
        | house_id = ashvault
        v
/rite/recognise
        |
        v
GrantCrownWrit("ashvault", usuário)
        |
        v
Ash-Vault recebe reconhecimento indevido
```

Isso caracteriza uma falha de autorização e um comportamento de **confused deputy**.

O servidor possuía autoridade para alterar o registro, mas aceitava que o cliente escolhesse o escopo da operação privilegiada.

---

## 7. Vetor adicional pela query string

A função vulnerável dava prioridade a:

```go
r.URL.Query().Get("house_id")
```

Uma requisição poderia possuir um corpo legítimo e uma query apontando para outra casa:

```http
POST /rite/recognise?house_id=ashvault
Content-Type: application/json

{"house_id":"rookhold"}
```

A assinatura da slate protegia os bytes previstos pelo protocolo, mas a query podia participar da seleção do destino.

Mesmo que a assinatura fosse válida, o destino da operação continuava sendo controlado pelo cliente.

---

## 8. Primeira tentativa e motivo da rejeição

A primeira correção implementada comparava a casa enviada pelo cliente com a casa da sessão:

```go
if !ownsHouse(p, house) {
	writeJSON(w, http.StatusForbidden, map[string]string{
		"error": "a scribe may only recognise the house they serve",
	})
	return
}

writ, err := s.store.GrantCrownWrit(
	houseOf(p),
	p.Username,
)
```

Essa correção bloqueava operações cross-house e todos os testes locais passavam.

Entretanto, a Pull Request foi rejeitada.

O problema era conceitual: mesmo sendo validado, o `house_id` do cliente ainda participava da autorização da operação.

A exigência correta era:

> O escopo da ação privilegiada deve ser derivado exclusivamente da sessão autenticada.

Portanto, o backend não deveria aceitar, rejeitar ou modificar o comportamento com base no `house_id` recebido.

Ele deveria simplesmente ignorá-lo.

---

## 9. Correção definitiva

A implementação final removeu `recogniseRequest` e `targetHouse()` do fluxo de reconhecimento.

O handler passou a derivar a casa exclusivamente do principal autenticado:

```go
house := houseOf(p)
```

A mutação final tornou-se:

```go
writ, err := s.store.GrantCrownWrit(
	house,
	p.Username,
)
```

O JSON e a query continuavam podendo ser enviados pelo cliente antigo, mas não possuíam mais autoridade.

---

## 10. Aplicando o patch definitivo

Execute dentro do repositório:

```bash
cd ~/CTF/withered-registry/core_application
```

Crie o script completo:

````bash
cat > corrigir_withered_registry.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel 2>/dev/null)" || {
    echo "[-] Execute dentro do repositório core_application."
    exit 1
}

cd "$ROOT"

ROUTES="registry-backend/internal/server/routes.go"
TESTS="registry-backend/internal/server/recognise_authz_test.go"

if [[ "$(git branch --show-current)" != "developer" ]]; then
    git checkout developer
fi

if [[ ! -f "$ROUTES" ]]; then
    echo "[-] Arquivo não encontrado: $ROUTES"
    exit 1
fi

echo "[*] Reescrevendo o handler de reconhecimento..."

python3 <<'PY'
from pathlib import Path

path = Path("registry-backend/internal/server/routes.go")
source = path.read_text(encoding="utf-8")

function_start = source.find(
    "func (s *Server) handleRecognise("
)

if function_start == -1:
    raise SystemExit(
        "[-] handleRecognise não encontrado."
    )

comment_markers = (
    "// handleRecognise raises a house to crown standing and records the writ.",
    "// handleRecognise raises the authenticated scribe's own house to crown",
)

comment_positions = [
    source.find(marker)
    for marker in comment_markers
    if source.find(marker) != -1
]

comment_start = (
    min(comment_positions)
    if comment_positions
    else function_start
)

legacy_positions = []

for marker in (
    "type recogniseRequest struct",
    "// targetHouse reads the house",
    "func targetHouse(",
):
    position = source.find(marker)

    if position != -1 and position < function_start:
        legacy_positions.append(position)

replace_start = (
    min(legacy_positions)
    if legacy_positions
    else comment_start
)

brace_start = source.find("{", function_start)

if brace_start == -1:
    raise SystemExit(
        "[-] Abertura do handler não encontrada."
    )

depth = 0
function_end = None
in_string = False
in_raw_string = False
escaped = False

for index in range(brace_start, len(source)):
    char = source[index]

    if in_raw_string:
        if char == "`":
            in_raw_string = False
        continue

    if in_string:
        if escaped:
            escaped = False
        elif char == "\\":
            escaped = True
        elif char == '"':
            in_string = False
        continue

    if char == '"':
        in_string = True
        continue

    if char == "`":
        in_raw_string = True
        continue

    if char == "{":
        depth += 1
    elif char == "}":
        depth -= 1

        if depth == 0:
            function_end = index + 1
            break

if function_end is None:
    raise SystemExit(
        "[-] Final do handler não encontrado."
    )

new_handler = '''// handleRecognise raises the authenticated scribe's own house to crown
// standing and records the writ. The request must carry a valid slate seal.
func (s *Server) handleRecognise(w http.ResponseWriter, r *http.Request) {
\tif !s.requireSlate(w, r) {
\t\treturn
\t}

\tp, ok := principalFrom(r)
\tif !ok {
\t\twriteJSON(w, http.StatusUnauthorized, map[string]string{
\t\t\t"error": "a scribe session is required",
\t\t})
\t\treturn
\t}

\t// Recognition is fully session scoped. The slate seal authenticates the
\t// field device and request bytes, but client input cannot select the house
\t// affected by this privileged ledger mutation.
\thouse := houseOf(p)

\twrit, err := s.store.GrantCrownWrit(house, p.Username)
\tif err != nil {
\t\tif errors.Is(err, store.ErrNotFound) {
\t\t\twriteJSON(w, http.StatusNotFound, map[string]string{
\t\t\t\t"error": "no such house",
\t\t\t})
\t\t\treturn
\t\t}

\t\twriteJSON(w, http.StatusInternalServerError, map[string]string{
\t\t\t"error": "could not seal writ",
\t\t})
\t\treturn
\t}

\twriteJSON(w, http.StatusOK, map[string]any{
\t\t"sealed":   true,
\t\t"house":    writ.HouseID,
\t\t"standing": "crown",
\t\t"writ":     writ,
\t})
}'''

updated = (
    source[:replace_start].rstrip()
    + "\n\n"
    + new_handler
    + source[function_end:]
)

path.write_text(
    updated.rstrip() + "\n",
    encoding="utf-8",
)

print("[+] Handler corrigido.")
PY

echo "[*] Criando testes de regressão..."

cat > "$TESTS" <<'GO'
package server

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

type sessionScopedRecognitionResponse struct {
	Sealed bool   `json:"sealed"`
	House  string `json:"house"`
}

func performSessionScopedRecognition(
	t *testing.T,
	srv *Server,
	cookie *http.Cookie,
	path string,
	payload []byte,
) sessionScopedRecognitionResponse {
	t.Helper()

	req := httptest.NewRequest(
		http.MethodPost,
		path,
		strings.NewReader(string(payload)),
	)

	req.Header.Set(
		"Content-Type",
		"application/json",
	)

	req.AddCookie(cookie)
	sealReq(req, payload)

	rec := httptest.NewRecorder()
	srv.Handler().ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf(
			"recognise %s: got HTTP %d, want 200: %s",
			path,
			rec.Code,
			rec.Body.String(),
		)
	}

	var response sessionScopedRecognitionResponse

	if err := json.Unmarshal(
		rec.Body.Bytes(),
		&response,
	); err != nil {
		t.Fatalf(
			"decode recognition response: %v",
			err,
		)
	}

	return response
}

func assertRecognitionHouseState(
	t *testing.T,
	srv *Server,
	houseID string,
	expectedStanding string,
	expectedWrits int,
) {
	t.Helper()

	house, err := srv.store.House(houseID)

	if err != nil {
		t.Fatalf(
			"read house %q: %v",
			houseID,
			err,
		)
	}

	if house.Standing != expectedStanding {
		t.Fatalf(
			"house %q standing: got %q, want %q",
			houseID,
			house.Standing,
			expectedStanding,
		)
	}

	writs, err := srv.store.WritsForHouse(houseID)

	if err != nil {
		t.Fatalf(
			"read writs for house %q: %v",
			houseID,
			err,
		)
	}

	if len(writs) != expectedWrits {
		t.Fatalf(
			"house %q writ count: got %d, want %d",
			houseID,
			len(writs),
			expectedWrits,
		)
	}
}

func TestRecogniseIgnoresForeignBodyTarget(
	t *testing.T,
) {
	srv, _ := newTestServer(t)
	cookie := login(t, srv)

	response := performSessionScopedRecognition(
		t,
		srv,
		cookie,
		"/rite/recognise",
		[]byte(`{"house_id":"ashvault"}`),
	)

	if !response.Sealed ||
		response.House != "rookhold" {
		t.Fatalf(
			"response: sealed=%v house=%q, want true/rookhold",
			response.Sealed,
			response.House,
		)
	}

	assertRecognitionHouseState(
		t,
		srv,
		"rookhold",
		"crown",
		1,
	)

	assertRecognitionHouseState(
		t,
		srv,
		"ashvault",
		"sworn",
		0,
	)
}

func TestRecogniseIgnoresForeignQueryTarget(
	t *testing.T,
) {
	srv, _ := newTestServer(t)
	cookie := login(t, srv)

	response := performSessionScopedRecognition(
		t,
		srv,
		cookie,
		"/rite/recognise?house_id=ashvault",
		[]byte(`{"house_id":"ashvault"}`),
	)

	if !response.Sealed ||
		response.House != "rookhold" {
		t.Fatalf(
			"response: sealed=%v house=%q, want true/rookhold",
			response.Sealed,
			response.House,
		)
	}

	assertRecognitionHouseState(
		t,
		srv,
		"rookhold",
		"crown",
		1,
	)

	assertRecognitionHouseState(
		t,
		srv,
		"ashvault",
		"sworn",
		0,
	)
}

func TestRecogniseDoesNotRequireClientTarget(
	t *testing.T,
) {
	srv, _ := newTestServer(t)
	cookie := login(t, srv)

	response := performSessionScopedRecognition(
		t,
		srv,
		cookie,
		"/rite/recognise",
		[]byte(`{}`),
	)

	if !response.Sealed ||
		response.House != "rookhold" {
		t.Fatalf(
			"response: sealed=%v house=%q, want true/rookhold",
			response.Sealed,
			response.House,
		)
	}

	assertRecognitionHouseState(
		t,
		srv,
		"rookhold",
		"crown",
		1,
	)
}
GO

echo "[*] Documentando a decisão de segurança..."

cat > SECURITY.md <<'MD'
# Recognition rite authorization

The recognition rite is a privileged, session-scoped operation.

The field slate authenticates request bytes with an HMAC. This proves that the
request came from a device holding the slate key, but it does not grant the
client authority to choose which house receives crown standing.

The vulnerable implementation read house_id from the query string or JSON body
and passed that client-controlled target to the ledger mutation.

An intermediate correction compared the requested house with the authenticated
principal. That still allowed request input to participate in determining the
scope of a privileged action.

The final trust boundary is:

1. The slate HMAC authenticates the field device and request bytes.
2. The session authenticates the scribe.
3. The session principal's HouseID is the sole source of mutation scope.
4. Query-string and JSON house_id values are ignored.

The mutation now derives its scope from the authenticated principal:

    house := houseOf(p)
    writ, err := s.store.GrantCrownWrit(house, p.Username)

The existing field-slate client can continue sending its legacy house_id
property, but that property cannot select, reject, or influence the house
affected by the rite.
MD

echo "[*] Formatando os arquivos Go..."

GO_FILES=(
    "$ROUTES"
    "$TESTS"
)

gofmt -w "${GO_FILES[@]}"

echo "[*] Validando o handler..."

HANDLER="$(
    sed -n \
        '/func (s \*Server) handleRecognise/,/^}/p' \
        "$ROUTES"
)"

if printf '%s\n' "$HANDLER" |
   grep -qE \
   'targetHouse|recogniseRequest|URL\.Query|json\.NewDecoder|body\.HouseID'; then

    echo "[-] O handler ainda utiliza escopo controlado pelo cliente."
    exit 1
fi

if ! printf '%s\n' "$HANDLER" |
     grep -q 'house := houseOf(p)'; then

    echo "[-] O handler não deriva a casa da sessão."
    exit 1
fi

if ! printf '%s\n' "$HANDLER" |
     grep -q 'GrantCrownWrit(house, p.Username)'; then

    echo "[-] A mutação não usa a casa derivada da sessão."
    exit 1
fi

git diff --check

echo
echo "===== STATUS ====="
git status --short

echo
echo "===== HANDLER FINAL ====="
printf '%s\n' "$HANDLER"

echo
echo "===== DIFF ====="
git diff -- \
    "$ROUTES" \
    "$TESTS" \
    SECURITY.md

echo
echo "[+] Patch aplicado."
EOF

chmod +x corrigir_withered_registry.sh
./corrigir_withered_registry.sh
````

---

## 11. Explicação dos testes adicionados

### 11.1 Corpo com outra casa

O teste envia:

```json
{
  "house_id": "ashvault"
}
```

A sessão pertence a `rookhold`.

O resultado correto é:

```text
rookhold  -> crown
ashvault  -> sworn
```

O campo externo é ignorado.

### 11.2 Query string com outra casa

O teste envia:

```http
POST /rite/recognise?house_id=ashvault
```

Mesmo assim, a operação deve afetar apenas a casa da sessão.

### 11.3 Requisição sem `house_id`

O teste envia:

```json
{}
```

Como o escopo vem da sessão, nenhum target fornecido pelo cliente é necessário.

Esse teste comprova que o campo foi realmente removido da decisão do servidor, em vez de apenas validado.

---

## 12. Executando testes, vet e build

Crie o script:

```bash
cat > testar_withered_registry.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
cd "$ROOT/registry-backend"

echo "===== GO VERSION ====="
go version

echo
echo "===== FORMATAÇÃO ====="

FORMAT_DIFF="$(
    gofmt -d \
        internal/server/routes.go \
        internal/server/recognise_authz_test.go
)"

if [[ -n "$FORMAT_DIFF" ]]; then
    echo "[-] Arquivos sem formatação:"
    printf '%s\n' "$FORMAT_DIFF"
    exit 1
fi

echo "[+] Formatação correta."

echo
echo "===== MÓDULOS ====="
go mod download

echo
echo "===== TODOS OS TESTES ====="

go test \
    -count=1 \
    ./...

echo
echo "===== TESTES DE RECONHECIMENTO ====="

go test \
    -count=1 \
    -run 'TestRecognise' \
    -v \
    ./internal/server

echo
echo "===== GO VET ====="

go vet ./...

echo
echo "===== BUILD ====="

go build \
    -trimpath \
    -o /tmp/withered-registry \
    ./cmd/registry

rm -f /tmp/withered-registry

echo
echo "[+] Testes, vet e build concluídos."
EOF

chmod +x testar_withered_registry.sh
./testar_withered_registry.sh
```

Os testes relevantes devem incluir:

```text
TestRecogniseIgnoresForeignBodyTarget
TestRecogniseIgnoresForeignQueryTarget
TestRecogniseDoesNotRequireClientTarget
TestRecogniseFromSlate
TestRecogniseRequiresSeal
TestRecogniseRequiresSession
```

---

## 13. Conferindo o patch

```bash
cat > conferir_withered_registry.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
cd "$ROOT"

ROUTES="registry-backend/internal/server/routes.go"

echo "===== BRANCH ====="
git branch --show-current

echo
echo "===== STATUS ====="
git status --short

echo
echo "===== HANDLER ====="

sed -n \
    '/func (s \*Server) handleRecognise/,/^}/p' \
    "$ROUTES"

echo
echo "===== BUSCA POR TARGET CONTROLADO ====="

HANDLER="$(
    sed -n \
        '/func (s \*Server) handleRecognise/,/^}/p' \
        "$ROUTES"
)"

if printf '%s\n' "$HANDLER" |
   grep -nE \
   'targetHouse|recogniseRequest|URL\.Query|json\.NewDecoder|body\.HouseID'; then

    echo "[-] Foi encontrada uma entrada insegura."
    exit 1
else
    echo "[+] Nenhum target fornecido pelo cliente é utilizado."
fi

echo
echo "===== DIFF ====="

git diff -- \
    SECURITY.md \
    registry-backend/internal/server/routes.go \
    registry-backend/internal/server/recognise_authz_test.go

echo
echo "===== DIFF CHECK ====="

git diff --check
EOF

chmod +x conferir_withered_registry.sh
./conferir_withered_registry.sh
```

O handler final deve conter:

```go
p, ok := principalFrom(r)
if !ok {
	// retorna 401
}

house := houseOf(p)

writ, err := s.store.GrantCrownWrit(
	house,
	p.Username,
)
```

Ele não deve conter:

```go
targetHouse(...)
```

```go
r.URL.Query().Get("house_id")
```

```go
json.NewDecoder(r.Body)
```

---

## 14. Commit e push

Depois dos testes passarem:

````bash
cat > enviar_withered_registry.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

TARGET="${1:-http://154.57.164.72:31333}"
TARGET="${TARGET%/}"

ROOT="$(git rev-parse --show-toplevel)"
cd "$ROOT"

if [[ "$(git branch --show-current)" != "developer" ]]; then
    echo "[-] A branch atual não é developer."
    exit 1
fi

git config user.name "joaov1t"
git config user.email "joaov1t@local"

PATCH_FILES=(
    "SECURITY.md"
    "registry-backend/internal/server/routes.go"
    "registry-backend/internal/server/recognise_authz_test.go"
)

git add "${PATCH_FILES[@]}"

echo "===== STAGED DIFF ====="
git diff --cached --stat

echo
echo "===== STAGED CHECK ====="
git diff --cached --check

STAGED_HANDLER="$(
    git show \
        :registry-backend/internal/server/routes.go |
    sed -n \
        '/func (s \*Server) handleRecognise/,/^}/p'
)"

if printf '%s\n' "$STAGED_HANDLER" |
   grep -qE \
   'targetHouse|recogniseRequest|URL\.Query|json\.NewDecoder|body\.HouseID'; then

    echo "[-] O código preparado ainda utiliza um target externo."
    exit 1
fi

if ! printf '%s\n' "$STAGED_HANDLER" |
     grep -q 'house := houseOf(p)'; then

    echo "[-] O código preparado não está session-scoped."
    exit 1
fi

if git diff --cached --quiet; then
    echo "[-] Nenhuma alteração preparada para commit."
    exit 1
fi

MESSAGE_FILE="$(mktemp)"
trap 'rm -f "$MESSAGE_FILE"' EXIT

cat > "$MESSAGE_FILE" <<'MSG'
fix: make recognition fully session scoped

Derive the recognition target exclusively from the authenticated principal.
Treat legacy house_id values from the slate body or query string as
non-authoritative so request input cannot influence the scope of the
privileged ledger mutation.

Add regression coverage proving that foreign targets are ignored and that no
client-selected house is required.
MSG

git commit -F "$MESSAGE_FILE"

echo
echo "[*] Enviando para origin/developer..."

git push -u origin developer

echo
echo "[+] Push concluído."
echo "[+] Pull Requests:"
echo "$TARGET/pulls"
EOF

chmod +x enviar_withered_registry.sh

./enviar_withered_registry.sh \
  http://154.57.164.72:31333
````

O commit final utilizado na solução foi:

```text
f4454f4 fix: make recognition fully session scoped
```

O push registrou uma nova Pull Request automaticamente.

---

## 15. Aguardando o resultado

```bash
cat > aguardar_resultado.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

TARGET="${1:-http://154.57.164.72:31333}"
TARGET="${TARGET%/}"

for ATTEMPT in $(seq 1 24); do
    echo
    echo "===== TENTATIVA $ATTEMPT/24 ====="

    FLAG_JSON="$(
        curl -fsS \
            --connect-timeout 5 \
            --max-time 20 \
            "$TARGET/flag"
    )" || {
        echo "[-] Falha ao consultar /flag."
        sleep 10
        continue
    }

    if command -v jq >/dev/null 2>&1; then
        printf '%s\n' "$FLAG_JSON" | jq .
    else
        printf '%s\n' "$FLAG_JSON"
    fi

    if printf '%s' "$FLAG_JSON" |
       grep -q \
       '"STATUS"[[:space:]]*:[[:space:]]*"SOLVED"'; then

        echo
        echo "[+] Desafio resolvido."
        exit 0
    fi

    echo
    echo "[*] Avaliação ainda em andamento."
    sleep 10
done

echo
echo "[-] A avaliação não terminou no período esperado."
echo "[*] Consulte:"
echo "    $TARGET/pulls"

exit 1
EOF

chmod +x aguardar_resultado.sh

./aguardar_resultado.sh \
  http://154.57.164.72:31333
```

---

## 16. Resultado final

Após a aprovação do patch, o endpoint `/flag` retornou o desafio como resolvido.

### Flag

```text
HTB{h0us3_0f_3mb3rs_4nd_4sh_9f3bffccfcbdf574c312982c90d56f34}
```

---

## 17. Resumo técnico

### Vulnerabilidade

Uma operação privilegiada aceitava o campo `house_id` da query string ou do corpo JSON e utilizava esse valor para escolher qual casa seria alterada.

Uma assinatura HMAC válida autenticava o dispositivo, mas não autorizava o dispositivo a agir em nome de qualquer casa escolhida pelo cliente.

### Correção incorreta intermediária

Validar o `house_id` com:

```go
ownsHouse(p, house)
```

bloqueava operações cross-house, mas ainda permitia que dados do cliente participassem da autorização.

### Correção final

A casa passou a ser derivada exclusivamente da sessão:

```go
house := houseOf(p)
```

A operação privilegiada tornou-se:

```go
writ, err := s.store.GrantCrownWrit(
	house,
	p.Username,
)
```

### Garantias obtidas

- O corpo JSON não escolhe a casa.
- A query string não escolhe a casa.
- Um campo `house_id` divergente é ignorado.
- A requisição funciona sem `house_id`.
- A assinatura HMAC continua obrigatória.
- A sessão autenticada continua obrigatória.
- Apenas a casa associada ao principal autenticado é modificada.
