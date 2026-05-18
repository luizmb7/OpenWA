# 🚀 Guia Ultra-Detalhado — OpenWA no EasyPanel

## O que você vai instalar

O **OpenWA** é composto por **dois serviços principais** que precisam ser criados separadamente no EasyPanel:

| # | Serviço | O que é | Porta interna |
|---|---|---|---|
| 1 | **api** | Backend NestJS + Chromium (engine WhatsApp) | `2785` |
| 2 | **dashboard** | Frontend React servido via Nginx | `80` |

Opcionalmente, para produção robusta:

| # | Serviço | O que é |
|---|---|---|
| 3 | **postgres** | Banco de dados PostgreSQL (template nativo) |
| 4 | **redis** | Cache Redis (template nativo) |

---

## 📋 ANTES DE COMEÇAR — Pré-requisitos

### 1. Servidor com recursos mínimos

O OpenWA usa **Chromium** por dentro (para simular o WhatsApp Web). Isso consome bastante recurso.

| Recurso | Mínimo | Recomendado |
|---|---|---|
| CPU | 1 vCPU | 2 vCPU |
| RAM | 1.5 GB | 2 GB+ |
| Disco | 10 GB | 20 GB+ |

> [!CAUTION]
> Com menos de 1.5 GB de RAM o Chromium pode não inicializar e a sessão do WhatsApp nunca ficará online. Se isso acontecer, você verá erros como `Target closed` ou `spawn chromium ENOENT` nos logs.

### 2. DNS configurado ANTES do deploy

O EasyPanel gera certificados HTTPS automaticamente via Let's Encrypt. Para isso funcionar, **os domínios precisam já estar apontando para o IP do servidor** antes de você configurar no EasyPanel.

**Como configurar no seu provedor de DNS:**

Crie dois registros tipo **A**:

```
api.openwa.seudominio.com   →  IP_DO_SEU_SERVIDOR
openwa.seudominio.com       →  IP_DO_SEU_SERVIDOR
```

> [!NOTE]
> Se você não tiver um domínio, pode usar subdomínios do próprio EasyPanel (ex: `*.easypanel.host`), dependendo do plano. Verifique nas configurações do EasyPanel em **Settings > Domains**.

### 3. Código no GitHub (ou GitLab)

O EasyPanel faz o build diretamente a partir do repositório Git.

- Se o repositório for **público**: sem configuração extra
- Se for **privado**: precisará configurar acesso (explicado no Passo 2)

---

## 🗂️ PARTE 1 — Criando o Projeto no EasyPanel

### Passo 1 — Acessar o EasyPanel e criar um Projeto

1. Acesse `https://IP_DO_SERVIDOR:3000` (ou o domínio do seu EasyPanel)
2. Faça login com suas credenciais de administrador
3. Na tela inicial, você verá a lista de projetos (pode estar vazia)
4. Clique no botão **"Create Project"** (geralmente um botão verde ou `+` no topo)
5. Na janela que aparecer:
   - **Name**: Digite `openwa`
   - **Description**: Opcional — pode deixar em branco
6. Clique em **"Create"**
7. Você será redirecionado para a **tela do projeto**, que está vazia por enquanto

---

## 🖥️ PARTE 2 — Serviço da API (Backend)

### Passo 2 — Criar o Serviço da API

1. Dentro do projeto `openwa`, clique em **"+ Create Service"** (ou **"+ Service"**)
2. Aparecerá uma lista de tipos de serviços. Selecione **"App"**
   - _Não confunda com "Postgres", "Redis", "MySQL" — esses são templates separados. Aqui é "App" mesmo._
3. Uma janela pedirá o nome do serviço:
   - **Name**: `api`
4. Clique em **"Create"**
5. O serviço `api` será criado e você verá sua página de configuração com várias abas no topo: **General, Source, Build, Environment, Domains, Mounts, Ports, etc.**

---

### Passo 3 — Conectar o Repositório GitHub

Na página do serviço `api`, clique na aba **"Source"**.

**Se o repositório for PÚBLICO:**

1. Em "Source Type", selecione **"Git"**
2. No campo **"Repository"**, cole a URL do repositório:
   ```
   https://github.com/rmyndharis/OpenWA.git
   ```
3. No campo **"Branch"**, digite: `main`
4. Clique em **"Save"**

**Se o repositório for PRIVADO (seu fork ou cópia privada):**

1. Em "Source Type", selecione **"Git"**
2. Você verá uma chave SSH pública exibida pelo EasyPanel (algo como `ssh-ed25519 AAAA...`)
3. **Copie essa chave SSH inteira**
4. Abra o GitHub em outra aba → Vá no repositório → **Settings > Deploy keys > Add deploy key**
5. Cole a chave copiada, dê um nome (ex: `EasyPanel`) e clique em **"Add key"**
6. Volte ao EasyPanel e preencha a URL do repositório no formato SSH:
   ```
   git@github.com:SEU_USUARIO/openwa.git
   ```
7. Branch: `main`
8. Clique em **"Save"**

> [!TIP]
> **Alternativa mais fácil para repositório privado:** Vá em **EasyPanel > Settings > GitHub** e conecte sua conta GitHub com um Personal Access Token (PAT). Depois disso, ao criar um serviço App, haverá uma opção "GitHub" (diferente de "Git") que lista seus repositórios automaticamente.

---

### Passo 4 — Configurar o Build (Dockerfile)

Ainda no serviço `api`, clique na aba **"Build"**.

1. Em **"Build Method"**, selecione **"Dockerfile"**
2. No campo **"Dockerfile"**, verifique se está: `Dockerfile`
   - _O Dockerfile da API está na raiz do projeto. O caminho correto é apenas `Dockerfile`, sem barras ou subpastas._
3. No campo **"Build Context"** (pode aparecer como "Context"): deixe em branco ou `.`
   - _Isso significa que o contexto de build é a raiz do repositório_
4. Clique em **"Save"**

> [!NOTE]
> O `Dockerfile` da raiz faz um **multi-stage build**: primeiro compila o TypeScript (stage `builder`) e depois cria a imagem de produção com Chromium (stage `production`). Isso é normal e pode demorar **5 a 10 minutos** no primeiro build.

---

### Passo 5 — Configurar as Variáveis de Ambiente da API

Clique na aba **"Environment"** do serviço `api`.

Você verá uma caixa de texto onde pode colar as variáveis. Cole o bloco abaixo **e edite os valores marcados com `← EDITE ISSO`**:

```env
# === CORE ===
NODE_ENV=production
PORT=2785
LOG_LEVEL=info

# === BANCO DE DADOS (SQLite — sem serviço externo) ===
DATABASE_TYPE=sqlite
DATABASE_NAME=/app/data/openwa.sqlite
DATABASE_SYNCHRONIZE=false

# === ENGINE WHATSAPP ===
ENGINE_TYPE=whatsapp-web.js
SESSION_DATA_PATH=/app/data/sessions
PUPPETEER_HEADLESS=true
PUPPETEER_ARGS=--no-sandbox,--disable-setuid-sandbox,--disable-dev-shm-usage,--disable-gpu

# === STORAGE (arquivos de mídia) ===
STORAGE_TYPE=local
STORAGE_LOCAL_PATH=/app/data/media

# === REDIS (desabilitado por padrão) ===
REDIS_ENABLED=false

# === WEBHOOKS ===
WEBHOOK_TIMEOUT=10000
WEBHOOK_MAX_RETRIES=3
WEBHOOK_RETRY_DELAY=5000

# === RATE LIMIT ===
RATE_LIMIT_TTL=60
RATE_LIMIT_MAX=100

# === PLUGINS ===
PLUGINS_ENABLED=true
PLUGINS_DIR=/app/data/plugins

# === SEGURANÇA — OBRIGATÓRIO EDITAR! ===
API_MASTER_KEY=TROQUE_ISSO_POR_UMA_CHAVE_FORTE    ← EDITE ISSO

# === CORS (qual domínio pode chamar a API) ===
CORS_ORIGINS=https://openwa.seudominio.com         ← EDITE ISSO
```

**Explicação de cada variável importante:**

| Variável | O que faz | O que colocar |
|---|---|---|
| `NODE_ENV` | Modo de execução | `production` sempre |
| `PORT` | Porta que a API escuta | Mantenha `2785` |
| `DATABASE_TYPE` | Tipo do banco | `sqlite` (simples) ou `postgres` (robusto) |
| `DATABASE_NAME` | Caminho do SQLite | `/app/data/openwa.sqlite` — fica no volume |
| `DATABASE_SYNCHRONIZE` | Sincroniza schema automaticamente | `false` em produção |
| `ENGINE_TYPE` | Motor do WhatsApp | Mantenha `whatsapp-web.js` |
| `PUPPETEER_HEADLESS` | Chrome sem janela | `true` sempre em servidor |
| `PUPPETEER_ARGS` | Flags de segurança do Chrome | **Não altere essas flags** |
| `STORAGE_TYPE` | Onde salvar mídias | `local` (no volume) |
| `REDIS_ENABLED` | Habilita cache Redis | `false` inicialmente |
| `API_MASTER_KEY` | Chave mestra de segurança | Gere com `openssl rand -hex 32` |
| `CORS_ORIGINS` | Origens permitidas na API | URL do seu dashboard |

**Como gerar uma `API_MASTER_KEY` segura:**

Se você tiver terminal, rode:
```bash
openssl rand -hex 32
```
Isso gera algo como: `a3f8b2c1d4e5f6...` — copie e cole como valor da variável.

Clique em **"Save"** ao terminar.

---

### Passo 6 — Configurar o Volume Persistente (CRÍTICO)

> [!CAUTION]
> **Este é o passo mais importante.** Sem o volume, toda vez que o container reiniciar você perderá: as sessões do WhatsApp conectadas, o banco de dados SQLite, e os arquivos de mídia. Você teria que escanear o QR Code de novo toda vez.

Na aba **"Mounts"** (ou **"Storage"**) do serviço `api`:

1. Clique em **"Add Mount"** (ou **"+"**)
2. Uma janela pedirá as configurações. Preencha:
   - **Type**: selecione **"Volume"**
   - **Name** (nome do volume): `openwa-data`
   - **Mount Path** (caminho dentro do container): `/app/data`
3. Clique em **"Confirm"** ou **"Save"**

Após salvar, o EasyPanel criará um diretório persistente em:
```
/etc/easypanel/projects/openwa/api/volumes/openwa-data
```
Este diretório sobrevive a reinicializações e atualizações do container.

---

### Passo 7 — Configurar o Domínio e HTTPS da API

Na aba **"Domains"** do serviço `api`:

1. Clique em **"Add Domain"**
2. Preencha os campos:
   - **Domain**: `api.openwa.seudominio.com` _(substitua pelo seu domínio real)_
   - **Port**: `2785` _(essa é a porta que o NestJS escuta — obrigatório colocar correto)_
   - **HTTPS**: habilite o toggle/checkbox para ativar Let's Encrypt
   - **Path** (se aparecer): deixe `/` ou em branco
3. Clique em **"Save"**

> [!WARNING]
> Se o DNS ainda não propagou, o HTTPS vai falhar na geração do certificado. Nesse caso, espere a propagação do DNS (pode levar até 24h, geralmente 5-30 min) e clique em **"Refresh"** ou **"Redeploy"** depois.

---

### Passo 8 — Fazer o Primeiro Deploy da API

Agora que tudo está configurado, clique no botão **"Deploy"** (geralmente no topo direito da página do serviço, ou na aba **"General"**).

O que acontece durante o deploy:
1. EasyPanel clona o repositório
2. Executa `docker build` usando o `Dockerfile`
3. **Stage 1 (builder):** instala node_modules e compila TypeScript → gera pasta `dist/`
4. **Stage 2 (production):** instala Chromium + dependências do sistema, copia a pasta `dist/`
5. Inicia o container com `dumb-init node dist/main`

**Acompanhe os logs em tempo real:**
- Clique na aba **"Logs"** (ou **"Build Logs"**) para ver o processo
- Um build bem-sucedido termina com algo como:
  ```
  Successfully built abc123def456
  Successfully tagged openwa-api:latest
  ```

**Tempo esperado:** 5 a 15 minutos no primeiro build (downloads do Chromium, compilação TypeScript).

**Como saber se subiu corretamente:**
- Na aba **"Logs"** (logs do container em runtime), você deve ver:
  ```
  [Bootstrap] First run detected, creating default configuration...
  🚀 OpenWA is running on: http://localhost:2785
  📚 Swagger docs: http://localhost:2785/api/docs
  ```
- O status do serviço no EasyPanel muda para **🟢 Running** ou **Healthy**

**Teste manual:**
```
https://api.openwa.seudominio.com/api/health
```
Deve retornar: `{"status":"ok","info":{},"error":{},"details":{}}`

---

## 🎨 PARTE 3 — Serviço do Dashboard (Frontend)

### Passo 9 — Criar o Serviço do Dashboard

1. No projeto `openwa`, clique em **"+ Create Service"** → **"App"**
2. **Name**: `dashboard`
3. Clique em **"Create"**

---

### Passo 10 — Configurar Source do Dashboard

Na aba **"Source"** do serviço `dashboard`:

- **Source Type**: `Git` (ou `GitHub` se conectou via PAT)
- **Repository**: mesma URL do repositório OpenWA
- **Branch**: `main`
- Clique em **"Save"**

---

### Passo 11 — Configurar Build do Dashboard (ATENÇÃO ao path!)

Na aba **"Build"** do serviço `dashboard`:

> [!IMPORTANT]
> O dashboard tem seu próprio `Dockerfile` dentro da pasta `dashboard/`. Aqui você precisa informar o caminho correto, diferente do serviço `api`.

1. **Build Method**: `Dockerfile`
2. **Dockerfile**: `dashboard/Dockerfile`
   - _Não esqueça o prefixo `dashboard/` — esse é o Dockerfile do frontend React_
3. **Build Context**: `dashboard`
   - _Isso diz ao Docker para usar a pasta `dashboard/` como raiz do contexto de build_
4. Clique em **"Save"**

> [!NOTE]
> O dashboard usa **Vite** para compilar o React, e depois o Nginx para servir os arquivos estáticos. O `Dockerfile` do dashboard faz exatamente isso: build do Vite → copia para Nginx.

---

### Passo 12 — Configurar Variáveis de Ambiente do Dashboard

Na aba **"Environment"** do serviço `dashboard`:

```env
VITE_API_URL=https://api.openwa.seudominio.com
```

> [!IMPORTANT]
> Substitua `api.openwa.seudominio.com` pelo domínio exato que você configurou para o serviço `api` no Passo 7. Sem essa variável, o dashboard não saberá onde encontrar a API e todas as chamadas falharão.

> [!NOTE]
> Variáveis que começam com `VITE_` são embutidas no bundle JavaScript **durante o build**. Isso significa que se você alterar esse valor depois, precisará fazer um novo deploy (rebuild) do dashboard para ter efeito.

Clique em **"Save"**.

---

### Passo 13 — Configurar Domínio do Dashboard

Na aba **"Domains"** do serviço `dashboard`:

1. Clique em **"Add Domain"**
2. Preencha:
   - **Domain**: `openwa.seudominio.com`
   - **Port**: `80` _(o Nginx dentro do container escuta na porta 80)_
   - **HTTPS**: habilitado
3. Clique em **"Save"**

---

### Passo 14 — Deploy do Dashboard

Clique em **"Deploy"** no serviço `dashboard`.

Tempo esperado: 2 a 5 minutos (build do Vite + Nginx).

**Como verificar:**
- Acesse `https://openwa.seudominio.com` no navegador
- Deve carregar a tela de login/dashboard do OpenWA
- Se ver uma tela em branco, abra o DevTools (F12) → Console e verifique se há erros de CORS ou conexão com a API

---

## 🗄️ PARTE 4 — Banco de Dados PostgreSQL (Opcional, para Produção)

> Pule esta parte se for usar SQLite (Opção A). Siga se quiser mais robustez.

### Passo 15 — Criar o Serviço PostgreSQL

1. No projeto `openwa`, clique em **"+ Create Service"**
2. Desta vez, selecione **"Postgres"** (template nativo — não é "App")
3. Preencha:
   - **Name**: `postgres`
   - **Image**: deixe o padrão (geralmente `postgres:16` ou `postgres:latest`)
   - **User**: `openwa`
   - **Password**: Gere uma senha forte (ex: `openssl rand -hex 16`)
   - **Database**: `openwa`
4. Clique em **"Create"**

### Passo 16 — Obter o Hostname Interno do PostgreSQL

Após criado, clique no serviço `postgres` → aba **"Credentials"** (ou **"Connection"**).

Você verá uma **"Internal Connection URL"** com formato:
```
postgresql://openwa:SUA_SENHA@postgres:5432/openwa
```

O hostname interno é simplesmente **`postgres`** — esse é o nome que o serviço `api` vai usar para se conectar.

> [!NOTE]
> No EasyPanel, todos os serviços dentro do **mesmo projeto** se comunicam pelo **nome do serviço** como hostname. Nenhuma porta precisa ser exposta publicamente para isso funcionar.

### Passo 17 — Atualizar as Variáveis da API para PostgreSQL

Volte ao serviço `api` → aba **"Environment"** e **substitua** as variáveis de banco:

```env
# Remova ou comente estas:
# DATABASE_TYPE=sqlite
# DATABASE_NAME=/app/data/openwa.sqlite

# Adicione estas:
DATABASE_TYPE=postgres
DATABASE_HOST=postgres
DATABASE_PORT=5432
DATABASE_USERNAME=openwa
DATABASE_PASSWORD=SUA_SENHA_DO_POSTGRES
DATABASE_NAME=openwa
DATABASE_SYNCHRONIZE=false
```

Clique em **"Save"** e depois **"Deploy"** para aplicar as mudanças.

---

## ⚡ PARTE 5 — Cache Redis (Opcional)

### Passo 18 — Criar o Serviço Redis

1. No projeto `openwa`, clique em **"+ Create Service"** → **"Redis"** (template nativo)
2. **Name**: `redis`
3. Deixe as configurações padrão (porta 6379, sem senha é aceitável em rede interna)
4. Clique em **"Create"**

### Passo 19 — Habilitar Redis na API

Volte ao serviço `api` → **"Environment"** e atualize:

```env
# Altere estas variáveis:
REDIS_ENABLED=true
REDIS_HOST=redis
REDIS_PORT=6379
```

Clique em **"Save"** e **"Deploy"**.

---

## ✅ PARTE 6 — Verificação Final e Primeira Conexão WhatsApp

### Checklist de verificação

Após todos os deploys, verifique cada item:

- [ ] **API Health**: acesse `https://api.openwa.seudominio.com/api/health` → deve retornar `{"status":"ok"}`
- [ ] **Swagger**: acesse `https://api.openwa.seudominio.com/api/docs` → deve carregar a interface interativa
- [ ] **Dashboard**: acesse `https://openwa.seudominio.com` → deve carregar a interface React
- [ ] **HTTPS**: cadeado verde nos dois domínios
- [ ] **Volume**: no EasyPanel, veja o serviço `api` → "Mounts" → deve mostrar o volume `openwa-data` montado em `/app/data`

### Passo 20 — Conectar o WhatsApp (Primeira Sessão)

Após o deploy bem-sucedido:

1. Acesse o dashboard em `https://openwa.seudominio.com`
2. Vá na seção **"Sessions"** (Sessões)
3. Clique em **"Create Session"** (ou **"New Session"**)
4. Dê um nome para a sessão (ex: `meu-whatsapp`)
5. Clique em **"Start"**
6. Aguarde alguns segundos — o Chromium irá inicializar em background
7. Aparecerá um **QR Code** na tela
8. Abra o WhatsApp no celular → **Configurações → Aparelhos conectados → Conectar um aparelho**
9. Escaneie o QR Code
10. A sessão mudará para status **"Connected"** (ou **"Online"**)

> [!NOTE]
> O QR Code expira em ~20 segundos. Se expirar, clique em "Refresh QR" para gerar um novo. Isso é comportamento normal do WhatsApp Web.

---

## 🔧 Solução de Problemas Comuns

### Problema: Container não sobe / Status "Unhealthy"

**Causa mais comum:** Falta de RAM para o Chromium.

**Solução:**
1. Vá nos logs do serviço `api` → aba **"Logs"**
2. Procure por: `Chromium`, `spawn`, `ENOMEM`, ou `Target closed`
3. Se encontrar: aumente a RAM do servidor ou desative outros processos

---

### Problema: Dashboard carrega mas a API não conecta (erro de CORS)

**Causa:** `CORS_ORIGINS` ou `VITE_API_URL` configurados errado.

**Solução:**
1. Serviço `api` → **"Environment"** → verifique `CORS_ORIGINS`
   - Deve ser exatamente `https://openwa.seudominio.com` (sem barra no final)
2. Serviço `dashboard` → **"Environment"** → verifique `VITE_API_URL`
   - Deve ser exatamente `https://api.openwa.seudominio.com` (sem barra no final)
3. Após corrigir, faça **"Deploy"** nos dois serviços

---

### Problema: Erro 502 Bad Gateway ao acessar o domínio

**Causa:** O EasyPanel ainda não conseguiu se comunicar com o container.

**Checklist:**
1. O container está com status **Running** (verde)?
2. A porta configurada em **"Domains"** está correta? (`2785` para api, `80` para dashboard)
3. Aguarde 1-2 minutos após o deploy — o container pode estar ainda inicializando

---

### Problema: HTTPS não funciona / certificado inválido

**Causa:** DNS não propagou ainda.

**Solução:**
1. Verifique se o DNS propagou: acesse `https://dnschecker.org` e teste seu domínio
2. Quando aparecer o IP correto, volte ao EasyPanel → serviço → **"Domains"** → clique em **"Redeploy"** ou salve novamente

---

### Problema: Sessão do WhatsApp desconecta após reiniciar o container

**Causa:** Volume não configurado corretamente.

**Solução:**
1. Serviço `api` → aba **"Mounts"**
2. Confirme que existe um volume com **Mount Path** = `/app/data`
3. Se não existir, adicione (Passo 6) e faça redeploy

---

## 📐 Resumo da Arquitetura Final

```
Internet
    │
    ▼
[EasyPanel Proxy / Traefik]
    │
    ├── api.openwa.seudominio.com:443 (HTTPS)
    │       │
    │       ▼
    │   [Serviço: api] ── porta 2785
    │       │
    │       ├── Volume: /app/data (sessões, SQLite, mídias)
    │       ├── [postgres] ── hostname: postgres:5432 (opcional)
    │       └── [redis]    ── hostname: redis:6379 (opcional)
    │
    └── openwa.seudominio.com:443 (HTTPS)
            │
            ▼
        [Serviço: dashboard] ── porta 80 (Nginx)
                │
                └── Chama a API via VITE_API_URL
```

---

## 🚀 Ordem recomendada de criação dos serviços

Se for usar PostgreSQL + Redis (Opção B):

```
1. postgres   ← criar primeiro (não depende de nada)
2. redis      ← criar segundo (não depende de nada)
3. api        ← criar terceiro (depende de postgres e redis)
4. dashboard  ← criar por último (depende da api estar online)
```

Se for usar SQLite (Opção A — mais simples):

```
1. api        ← criar primeiro
2. dashboard  ← criar após a api estar online
```
