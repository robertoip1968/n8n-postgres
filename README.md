
# n8n atrás do Apache (cPanel) com Docker + Postgres (push via **SSE**) — Guia de Implantação
# Os arquivos que funcionaram estão nesse repositório. Os que estão aqui são para teste e validação

Este roteiro documenta a instalação que **funcionou** para `https://ia.cursatto.com.br`, rodando o **n8n** em Docker (com Postgres) e servindo-o por trás do **Apache** do cPanel (reverse proxy). A solução usa **Server‑Sent Events (SSE)** em `/rest/push` (sem WebSocket) e injeta o cabeçalho **`Origin`** apenas quando estiver ausente, eliminando o erro **“Invalid origin”** e o **connection lost**.

> ✅ Se você copiar/colar os trechos abaixo e seguir as verificações, terá um setup reprodutível para outros domínios/servidores.

---

## Visão geral (arquitetura)

```
Navegador ─HTTPS─> Apache (cPanel)
                       │
               reverse proxy (HTTP)
                       │
                 127.0.0.1:5678
                       │
                  Docker: n8n
                       │
                  Docker: Postgres
```

- O **Apache** responde no 443 com certificado (Let's Encrypt do cPanel).
- O **n8n** escuta **somente** no `127.0.0.1:5678` (sem exposição externa), recebendo o tráfego via proxy.
- O push do editor usa **SSE** em `GET /rest/push?...`. Nós normalizamos `X-Forwarded-*` e **injeta­mos** `Origin` apenas quando está vazio, evitando _Invalid origin_.

---

## Pré‑requisitos

- Servidor com **cPanel/WHM** (Apache EasyApache 4).
- **Docker** e **Docker Compose** instalados (no host onde o n8n/DB irão rodar).
- Acesso root/SSH.
- Domínio/subdomínio apontando para o servidor (ex.: `ia.cursatto.com.br`) e **SSL ativo** no cPanel.

> Dica: se houver **ModSecurity**, manteremos uma exceção **somente** para `/rest/push` (opcional, use apenas se o ModSec bloquear).

---

## 1) Estrutura de pastas

No host (fora do container), crie um diretório de trabalho (ex.: `~/n8n-docker`) e as pastas de dados:

```bash
mkdir -p ~/n8n-docker/data/n8n ~/n8n-docker/data/postgres
cd ~/n8n-docker
```

Dê **permissão** para o usuário padrão do container do n8n (`uid/gid 1000`):

```bash
sudo chown -R 1000:1000 ~/n8n-docker/data/n8n
```

---

## 2) Arquivo `.env` (exemplo funcional)

Crie `~/n8n-docker/.env` (ajuste domínio, senhas e **key**):

```dotenv
# Fuso
TIMEZONE=America/Cuiaba

# Postgres
POSTGRES_USER=n8n
POSTGRES_PASSWORD=PostgresMuitoForte123!
POSTGRES_DB=n8n

# Basic Auth do editor (recomendado em produção)
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=AdminForte456!

# URL pública do n8n
N8N_PORT=5678
N8N_HOST=ia.cursatto.com.br
N8N_PROTOCOL=https
N8N_PROXY_HOPS=1

LETSENCRYPT_EMAIL=suporte@cursatto.com.br
DOMAIN_NAME=ia.cursatto.com.br

# URLs do n8n
WEBHOOK_URL=https://ia.cursatto.com.br/
N8N_EDITOR_BASE_URL=https://ia.cursatto.com.br

# Push por SSE (compatível com Apache)
N8N_PUSH_BACKEND=sse

# Origens permitidas (SEM espaços; use a URL com esquema https)
N8N_SECURITY_ALLOWED_ORIGINS=https://ia.cursatto.com.br

# Chave de criptografia (gere com: openssl rand -base64 32)
N8N_ENCRYPTION_KEY=z82TJlixy2Sul/lpcQRp0a+fxTCjuos+Da7/20ezaMo=

# (Opcional) Logs mais verbosos para diagnóstico
N8N_LOG_LEVEL=debug
```

**Importante:** mantenha **apenas uma** origem em `N8N_SECURITY_ALLOWED_ORIGINS` (inclua o `https://`). Evite duplicar host em `X-Forwarded-Host`.

---

## 3) `docker-compose.yml`

Crie `~/n8n-docker/docker-compose.yml`:

```yaml
version: "3.9"

services:
  n8n_postgres:
    image: postgres:14
    container_name: n8n_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
      - TZ=${TIMEZONE}
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks:
      - n8n_net

  n8n:
    image: docker.n8n.io/n8nio/n8n:1.115.3
    container_name: n8n
    restart: unless-stopped
    depends_on:
      n8n_postgres:
        condition: service_healthy
    user: "1000:1000"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=n8n_postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - GENERIC_TIMEZONE=${TIMEZONE}
      - TZ=${TIMEZONE}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_HOST=${N8N_HOST}
      - N8N_PROTOCOL=${N8N_PROTOCOL}
      - N8N_PORT=${N8N_PORT}
      - N8N_EDITOR_BASE_URL=${N8N_EDITOR_BASE_URL}
      - WEBHOOK_URL=${WEBHOOK_URL}
      - N8N_BASIC_AUTH_ACTIVE=${N8N_BASIC_AUTH_ACTIVE}
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_PUSH_BACKEND=${N8N_PUSH_BACKEND}
      - N8N_SECURITY_ALLOWED_ORIGINS=${N8N_SECURITY_ALLOWED_ORIGINS}
      - N8N_PROXY_HOPS=${N8N_PROXY_HOPS}
      - N8N_LOG_LEVEL=${N8N_LOG_LEVEL}
    volumes:
      - ./data/n8n:/home/node/.n8n
    # NUNCA exponha 0.0.0.0; mantenha só localhost (Apache faz o proxy)
    ports:
      - "127.0.0.1:5678:5678"
    networks:
      - n8n_net

networks:
  n8n_net:
    name: n8n-docker_n8n_net
```

Suba os serviços:

```bash
docker compose up -d
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | egrep 'n8n|postgres'
```

Quando ok, os logs do n8n devem mostrar algo como:

```
n8n ready on ::, port 5678
Editor is now accessible via: https://ia.cursatto.com.br
```

---

## 4) Apache (cPanel) — reverse proxy e cabeçalhos

### 4.1 Arquivo do proxy

Crie/edite **(via root)** o arquivo abaixo (ajuste o caminho do seu vhost SSL no cPanel):

```
/etc/apache2/conf.d/userdata/ssl/2_4/iacursattocom/ia.cursatto.com.br/proxy_n8n.conf
```

Conteúdo **completo** (SSE‑only + normalização de cabeçalhos + injeção condicional de Origin):

```apache
# --- n8n via Apache reverse proxy (SSL vhost, SSE-only) ---
# Requisitos: mod_proxy, mod_proxy_http, mod_headers, mod_rewrite
# Caminho típico no cPanel:
# /etc/apache2/conf.d/userdata/ssl/2_4/iacursattocom/ia.cursatto.com.br/proxy_n8n.conf

# Força HTTP/1.1 (SSE estável atrás de proxy)
Protocols http/1.1

ProxyPreserveHost On
ProxyRequests Off
AllowEncodedSlashes NoDecode
ProxyAddHeaders On
ProxyTimeout 900

# Normaliza cabeçalhos de forward (evita duplicar X-Forwarded-Host)
RequestHeader unset X-Forwarded-Host
RequestHeader set   X-Forwarded-Host "ia.cursatto.com.br" early
RequestHeader set   X-Forwarded-Proto "https" early
RequestHeader set   X-Forwarded-Port  "443"   early

# --- injeta Origin SOMENTE se vier vazio no /rest/push (SSE) ---
RewriteEngine On
RewriteCond %{REQUEST_URI} ^/rest/push [NC]
RewriteCond %{HTTP:Origin} ^$ [NC]
RewriteRule ^ - [E=MISSING_ORIGIN:1]
RequestHeader set Origin "https://ia.cursatto.com.br" env=MISSING_ORIGIN

# --- PUSH por SSE (HTTP), sem websocket e sem nocanon ---
ProxyPass        /rest/push  http://127.0.0.1:5678/rest/push keepalive=On retry=0 flushpackets=on timeout=900
ProxyPassReverse /rest/push  http://127.0.0.1:5678/rest/push

# --- Demais rotas para o n8n ---
ProxyPass        /  http://127.0.0.1:5678/ connectiontimeout=60 timeout=900 keepalive=On
ProxyPassReverse /  http://127.0.0.1:5678/

# Permitir payloads grandes (opcional)
LimitRequestBody 0
```

Aplique e reinicie o Apache:

```bash
/scripts/ensure_vhost_includes --all-users
/scripts/rebuildhttpdconf
/scripts/restartsrv_httpd
```

### 4.2 (Opcional) ModSecurity — exceção **somente** para `/rest/push`

Use **apenas se** o ModSec estiver bloqueando SSE:

Arquivo:
```
/etc/apache2/conf.d/userdata/ssl/2_4/iacursattocom/ia.cursatto.com.br/modsec_n8n.conf
```

Conteúdo:
```apache
<LocationMatch "^/rest/push/?">
  SecRuleEngine Off
</LocationMatch>
```

Reaplique o include e reinicie o Apache (mesmos comandos acima).

---

## 5) Testes e verificação

### 5.1 n8n/DB
```bash
docker logs --tail=200 n8n_postgres
docker logs --tail=200 n8n | egrep -i 'ready on|Editor is now|push|origin|error'
```

### 5.2 SSE do push
- Abra o DevTools → **Network** no editor do n8n, clique **New** workflow.
- Deve aparecer um `GET /rest/push?pushRef=...` com **Status 200** e ficar **pendente** (stream aberto).
- No container, **não** deve aparecer “Invalid origin”.

> **Dica de linha de comando:** um `401 Unauthorized` ao chamar `/rest/push` via `curl` **é esperado** (sem cookie de sessão). Exemplo:
>
> ```bash
> curl -vk -H "Accept: text/event-stream" -H "Origin: https://ia.seu-dominio" \
>   "https://ia.seu-dominio/rest/push?pushRef=test"
> # Deve retornar 401 (sem sessão). No navegador autenticado → 200 pendente.
> ```

---

## 6) Erros comuns & solução

- **`Invalid origin`** no log do n8n  
  Causa: `X-Forwarded-Host` duplicado / `Origin` ausente.  
  Ação: usar o `proxy_n8n.conf` acima (unset + set de `X-Forwarded-*`) e injeção condicional de `Origin` apenas para `/rest/push`. Em `.env`, mantenha **uma** entrada em `N8N_SECURITY_ALLOWED_ORIGINS` (com `https://`).

- **`Connection lost`** ao criar workflow  
  Usar **SSE** (`N8N_PUSH_BACKEND=sse`) + proxy Apache configurado como acima.

- **503 / porta 5678 recusada**  
  Container do n8n não iniciou (DB indisponível). Checar saúde do Postgres e variáveis `DB_*`.

- **`EACCES: permission denied, open '/home/node/.n8n/config'`**  
  Corrija owner: `chown -R 1000:1000 ./data/n8n` (no host), e suba novamente.

- **Cloudflare** (se usar no futuro)  
  Adicione uma **Transform Rule** para fixar o cabeçalho `Origin` para `https://seu-dominio` antes do request chegar ao servidor.

---

## 7) Operação

- **Subir/atualizar**: `docker compose up -d --pull=missing`
- **Recriar após alterações de env**: `docker compose up -d --force-recreate`
- **Backups**: copie `./data/n8n` (workflows/credenciais) e `./data/postgres` (banco).
- **Segurança**: mantenha `N8N_ENCRYPTION_KEY` secreto e **Basic Auth** ativo em produção.

---

## 8) Checklist rápido (para reproduzir)

1. Criar pastas e permissões (`data/n8n` dono 1000:1000).  
2. Salvar `.env` (sem espaços em `ALLOWED_ORIGINS`).  
3. Salvar `docker-compose.yml` e subir `docker compose up -d`.  
4. Criar `proxy_n8n.conf` no vhost SSL.  
5. (Se necessário) `modsec_n8n.conf` para `/rest/push`.  
6. `ensure_vhost_includes` → `rebuildhttpdconf` → `restartsrv_httpd`.  
7. Acessar `https://seu-dominio` → criar workflow → confirmar `GET /rest/push` **200 (pending)**.  
8. Confirmar logs **sem** “Invalid origin”.

---

## 9) Apêndice – Comandos úteis

```bash
# Ver env dentro do container
docker exec -it n8n env | egrep 'N8N_(HOST|PROTOCOL|EDITOR_BASE_URL|PUSH_BACKEND|SECURITY_ALLOWED_ORIGINS)'

# Verificar SSE/origin nos logs
docker logs -f n8n | egrep -i 'push|origin|Invalid origin|ready on'

# Testar HTTP local do n8n via host
curl -sS -D- http://127.0.0.1:5678/ -H 'Host: ia.cursatto.com.br' | head
```

---

**Fim.**
