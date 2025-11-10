
# Roteiro de instalação do n8n (Docker) atrás do Apache/cPanel com SSL

Este guia documenta a instalação do **n8n** em uma VPS com **Docker** e **cPanel/Apache** fazendo o reverse‑proxy (HTTP/HTTPS + WebSocket) e emissão de SSL. Inclui arquivos prontos: `.env`, `docker-compose.yml` e exemplos de includes do Apache.

---

## 0) Pré‑requisitos
- VPS (ex.: AlmaLinux 9) com **Docker** e **Docker Compose**.
- Acesso root/SSH.
- cPanel/WHM com **Apache** e módulos `proxy_module`, `proxy_http_module`, `proxy_wstunnel_module`, `headers_module` ativos.
- Porta **80/443** liberadas.
- **DNS A** do subdomínio apontando para o IP da VPS (ex.: `ia.seudominio.com.br`).

> Teste DNS: `nslookup ia.seudominio.com.br` deve retornar o IP da VPS.

---

## 1) Subdomínio no cPanel e SSL
1. Em **Domains → Create A New Domain**, crie `ia.seudominio.com.br` (document root padrão pode ser `/public_html`).
2. Em **Zone Editor**, confirme o **A record** para o IP da VPS.
3. Rode o **AutoSSL**. Se falhar por DCV:
   - Teste: `curl -I http://ia.seudominio.com.br/.well-known/acme-challenge/ping.txt` (após configurar proxy também funciona).
   - Verifique que a porta **80** atende no servidor.

---

## 2) Stack Docker
Crie a pasta de trabalho e copie os arquivos anexos (`.env` e `docker-compose.yml`) para **/root/n8n-docker**:

```bash
mkdir -p /root/n8n-docker && cd /root/n8n-docker
```

### 2.1) Gere segredos
```bash
# chave de criptografia do n8n (obrigatória)
openssl rand -base64 32

# senhas aleatórias (se for usar Postgres)
openssl rand -hex 16
```

Edite o arquivo **.env** preenchendo `DOMAIN`, `TIMEZONE`, `N8N_ENCRYPTION_KEY` e credenciais.

### 2.2) Subir a stack
```bash
docker compose up -d
docker logs -f n8n   # aguarde "Editor is now accessible via: https://<DOMAIN>"
```

> Para atualizar futuramente: `docker compose pull && docker compose up -d`

---

## 3) Apache como reverse‑proxy (HTTP + WebSocket)

Crie **includes** do Apache no cPanel para o seu usuário e domínio (troque `<cpuser>` e `<domain>`).

```bash
# HTTP (80)
mkdir -p /etc/apache2/conf.d/userdata/std/2_4/<cpuser>/<domain>/
cp /root/n8n-docker/apache-n8n-proxy-http.conf    /etc/apache2/conf.d/userdata/std/2_4/<cpuser>/<domain>/n8n-proxy.conf

# HTTPS (443)
mkdir -p /etc/apache2/conf.d/userdata/ssl/2_4/<cpuser>/<domain>/
cp /root/n8n-docker/apache-n8n-proxy-ssl.conf     /etc/apache2/conf.d/userdata/ssl/2_4/<cpuser>/<domain>/n8n-proxy.conf

# Aplicar
/scripts/rebuildhttpdconf
/usr/local/cpanel/scripts/restartsrv_httpd
```

> Confirme módulos: `httpd -M | egrep 'proxy|wstunnel|headers'`  
> Deve listar `proxy_module`, `proxy_http_module`, `proxy_wstunnel_module`, `headers_module`.

---

## 4) Testes de saúde

Dentro do servidor:

```bash
# Variáveis importantes
docker exec n8n env | egrep 'N8N_HOST|N8N_PROTOCOL|TRUSTED_PROXIES|WEBHOOK_URL|EDITOR_BASE_URL|SECURE_COOKIE'

# Página principal
curl -I https://ia.seudominio.com.br/

# WebSocket através do proxy (tem que responder, sem 502)
curl -i -H 'Host: ia.seudominio.com.br'      -H 'Connection: Upgrade' -H 'Upgrade: websocket'      'http://127.0.0.1/socket.io/?EIO=4&transport=websocket'
```

No navegador: acesse `https://ia.seudominio.com.br` e verifique que **não** aparece “Connection lost”.

---

## 5) Operação diária

**Subir/derrubar/atualizar**
```bash
cd /root/n8n-docker
docker compose up -d
docker compose down
docker compose pull && docker compose up -d
docker logs -f n8n
```

**Backup mínimo**
- `./data/n8n` (config/usuários/workflows).
- Se usar Postgres: `./data/postgres` ou dump periódico:
  ```bash
  docker exec -t n8n-docker-postgres-1 pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" > /root/backup/n8n_$(date +%F).sql
  ```

---

## 6) Troubleshooting

**Autossl (HTTP DCV 404/503)**  
- Confirme DNS A → IP da VPS e que a porta 80 responde.
- Recrie os includes e reinicie o Apache.

**“Connection lost” no editor**  
- Garanta os blocos `ProxyPass /socket.io` (ws) **em HTTP e HTTPS**.
- Dentro do container, verifique:
  - `N8N_TRUSTED_PROXIES` inclui `127.0.0.1`.
  - `N8N_HOST`, `N8N_PROTOCOL=https`, `N8N_EDITOR_BASE_URL`, `WEBHOOK_URL`.

**`ERR_ERL_UNEXPECTED_X_FORWARDED_FOR` nos logs**  
- Falta `N8N_TRUSTED_PROXIES`. Adicione e `docker compose up -d`.

**Compose não derruba container (overlay2 device busy)**  
```bash
docker rm -f n8n
systemctl restart docker
docker compose up -d
```

---

## 7) Check‑list final
- [ ] DNS A do subdomínio aponta para o IP da VPS.  
- [ ] SSL emitido (AutoSSL).  
- [ ] Includes Apache criados (std e ssl) com `socket.io` (ws).  
- [ ] `N8N_TRUSTED_PROXIES` setado.  
- [ ] Acesso em `https://SEU_SUBDOMINIO` sem “Connection lost”.

---

## 8) Arquivos fornecidos
- `.env` – preencha os valores indicados.
- `docker-compose.yml` – compose com Postgres + n8n.
- `apache-n8n-proxy-http.conf` – include HTTP (80) com WebSocket.
- `apache-n8n-proxy-ssl.conf` – include HTTPS (443) com WebSocket.
