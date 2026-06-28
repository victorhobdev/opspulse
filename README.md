# OpsPulse

Sistema para **cadastro, execução agendada e acompanhamento de rotinas HTTP** — com histórico de execuções, autenticação, controle de concorrência e visibilidade do que foi executado, quando e com qual resultado.

## Demo

- **App:** https://opspulse-self.vercel.app/
**Demo account**
  
- Email: demo@opspulse.app
- Senha: opspulse

> **Problema que resolve:** ter um lugar simples para **agendar, executar e acompanhar rotinas HTTP** (pings, webhooks e checagens), com histórico persistido e rastreabilidade do que deu certo/errado.
>
> A prioridade é ser **simples, funcional e barato de manter**, usando free tiers sempre que possível.

---

## O que é o OpsPulse

Pense nele como um **“cron + monitor + histórico”** minimalista.

Você consegue:
- Criar conta e logar (**Supabase Auth**)
- Cadastrar rotinas HTTP (URL + método + headers seguros)
- Definir frequência por `interval_minutes` (MVP sem cron string)
- Rodar manualmente (teste rápido)
- Rodar automaticamente (scheduler via **Azure Timer Trigger**)
- Ver histórico de execuções (status, HTTP status, duração, timestamps)
- Ver “saúde” do backend via `/health`

---

## Stack

**Back-end**
- Azure Functions (Python)
- Endpoints REST
- Timer Trigger como scheduler
- Configuração via App Settings (env vars)
- Logs como fonte de verdade de produção

**Banco + Auth**
- Supabase (PostgreSQL + Auth)
- Front autentica direto no Supabase
- Backend valida o usuário com token do Supabase
- Banco “de verdade”: constraints, índices e consistência

**Front-end**
- React + Vite + TypeScript
- Tailwind + shadcn/ui (UI moderna, tema claro)
- Deploy no Vercel
- SPA routing configurado (regras de rewrite)

---

## Decisões técnicas

### Scheduler
- Agendamento por `interval_minutes` (>= 5)
- Busca rotinas "devidas" por `next_run_at`
- Proteção de concorrência com lock (`lock_until`, `locked_by`)
- `MAX_CONCURRENCY` para limitar execuções em paralelo
- Timeout controlado na execução HTTP

### Controle de drift
O Timer Trigger pode acordar alguns segundos antes do slot. Para não perder execuções:
- Query considera slack configurável (`DUE_SLACK_SECONDS`, default 3s)
- Próximo `next_run_at` é ancorado no slot devido, evitando drift acumulado
- Timestamps padronizados com truncagem para minuto

### Segurança de produção
- Segredos **não** vão para o banco
- `auth_mode = NONE | SECRET_REF`
- `secret_ref` aponta para env var no backend (Azure App Settings)
- Backend bloqueia headers sensíveis em `headers_json` (ex.: `Authorization`, `Cookie`, `X-API-Key`)
- Backend usa `SERVICE_ROLE` apenas no servidor (nunca no front)

### Observações de deploy
Pontos encontrados na operação real:
- Deploy separado (Vercel vs Azure)
- SPA rewrite (refresh em rota)
- Diferenças de tipos/serialização (Pydantic/URL)
- Variáveis de ambiente e tokens (CRLF, etc.)
- Consistência de enums/constraints no banco

---

## Modelo de dados (resumo)

Tabelas:
- `workspaces`: `id`, `owner_id`, `name`, `created_at`
- `routines`: `workspace_id`, `name`, `kind`, `interval_minutes`, `next_run_at`, `last_run_at`, `is_active`,
  `endpoint_url`, `http_method`, `headers_json`, `auth_mode`, `secret_ref`, `lock_until`, `locked_by`, `created_at`, `updated_at`
- `routine_runs`: `routine_id`, `triggered_by (MANUAL|SCHEDULE)`, `status (SUCCESS|FAIL)`, `http_status`, `duration_ms`,
  `error_message`, `started_at`, `finished_at`, `created_at`

Índices principais (foco scheduler e histórico):
- `idx_routines_scheduler (is_active, next_run_at)`
- `idx_runs_routine_created (routine_id, created_at desc)`

Mais detalhes: `docs/DB_OVERVIEW.md`

---

## Estrutura do repositório

- `api/` → Azure Functions (Python)
- `web/` → React + Vite + TS
- `docs/` → documentação técnica (DB, decisões, etc.)
- `tools/` → scripts utilitários (ex.: token/automação local)

---

## Rodando localmente

### Pré-requisitos
- Node 18+ (recomendado 20+)
- pnpm
- Python 3.11+ (ou 3.10+)
- Azure Functions Core Tools v4

---

### 1) Configurar Supabase
Crie um projeto no Supabase e obtenha:
- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`

Crie as tabelas conforme `docs/DB_OVERVIEW.md` (ou via SQL equivalente).

---

### 2) API (Azure Functions)

Dentro de `api/`, crie `local.settings.json`:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "python",

    "SUPABASE_URL": "https://xxxx.supabase.co",
    "SUPABASE_ANON_KEY": "xxxxx",
    "SUPABASE_SERVICE_ROLE_KEY": "xxxxx",

    "HTTP_TIMEOUT_SECONDS": "8",
    "MAX_CONCURRENCY": "5",
    "LOCK_LEASE_SECONDS": "45",
    "SCHEDULER_BATCH_LIMIT": "20",
    "DUE_SLACK_SECONDS": "3"
  }
}
```

Instalar deps e rodar:

```bash
cd api
python -m venv .venv
# ativar venv
pip install -r requirements.txt
func start
```

Endpoints principais:
- `GET /api/health`
- `POST /api/routines`
- `GET /api/routines`
- `GET /api/routines/{id}`
- `PATCH /api/routines/{id}`
- `DELETE /api/routines/{id}`
- `POST /api/routines/{id}/run`
- `GET /api/routines/{id}/runs`

---

### 3) WEB (Vite + React)

Dentro de `web/`, crie `.env.local`:

```bash
VITE_SUPABASE_URL=https://xxxx.supabase.co
VITE_SUPABASE_ANON_KEY=xxxxx
VITE_API_BASE_URL=http://localhost:7071/api
```

Rodar:

```bash
cd web
pnpm install
pnpm dev
```

---

## Deploy (produção)

### API (Azure Functions)
- configurar App Settings com as mesmas env vars do `local.settings.json` (sem commitar secrets)
- `SUPABASE_SERVICE_ROLE_KEY` fica somente no backend

### Web (Vercel)
- setar `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, `VITE_API_BASE_URL`
- manter `vercel.json` com rewrite para SPA routing

---

## Custos (visão rápida)
- Supabase: free tier (DB + Auth)
- Vercel: free tier para front
- Azure Functions: free grant/consumo 

---

## Roadmap (ideias futuras)
- Cron string (opcional) + timezone handling
- RLS no Supabase (polimento de segurança real)
- Página de observabilidade (filtros, agregações, alertas)
- Integrações (webhook, Slack/Email) para falhas
- “AI Run Summary” para falhas (enviando contexto mínimo sanitizado)

---

## Licença
MIT (ver `LICENSE`)
