# Guia Completo de Deploy no Coolify — Google Maps Scraper Kit

Este guia ensina a colocar o **Google Maps Scraper Kit** para rodar no seu servidor **Coolify**, aproveitando a estabilidade do Linux para o Playwright/Chromium e permitindo que você dispare coletas remotamente da sua máquina ou via automação.

---

## 🚀 Escolha o seu método de Deploy

Você tem **duas opções** prontas no projeto:

1. **Opção A (Recomendada / Mais Simples):** Scraper direto com porta `8080` e proteção por **Basic Auth** nativa do Coolify.
2. **Opção B (Com Chave de API Token):** Scraper + proxy Caddy embutido que valida o cabeçalho `X-API-Key`.

---

## 📦 Opção A: Deploy Padrão (Porta 8080 + Basic Auth do Coolify)

### 1. Criar o Recurso no Coolify
1. No seu painel do Coolify, entre no seu **Projeto** e **Ambiente**.
2. Clique em **+ New Resource** e selecione **Docker Compose** (ou escolha **Public Repository** apontando para o seu repositório).
3. Cole o conteúdo abaixo (arquivo `docker-compose.coolify.yml`):

```yaml
services:
  google-maps-scraper:
    image: gosom/google-maps-scraper:v1.15.0
    container_name: gmaps-scraper
    restart: unless-stopped
    command: ["-web", "-data-folder", "/gmapsdata"]
    ports:
      - "8080:8080"
    volumes:
      - gmaps_data:/gmapsdata
      - gmaps_cache:/opt

volumes:
  gmaps_data:
  gmaps_cache:
```

### 2. Configurar Domínio e Porta
* **Domains:** Defina o domínio que deseja usar, por exemplo: `https://scraper.seudominio.com`.
* **Port:** Certifique-se de que a porta de roteamento do Coolify seja **`8080`**.

### 3. Ativar Autenticação Básica (Segurança Obrigatória)
> ⚠️ **Importante:** O motor do scraper não possui autenticação interna. Sem proteção, qualquer pessoa na internet que descobrir a URL poderá disparar raspagens no seu servidor.

No Coolify:
1. Vá na aba de configurações do recurso (ou configurações do proxy Traefik).
2. Ative **Basic Authentication**.
3. Defina um usuário e senha (exemplo: `admin` / `sua_senha_segura`).

### 4. Iniciar o Deploy
* Clique no botão **Deploy**.
* O Coolify baixará a imagem Docker e subirá o serviço com SSL automático (Let's Encrypt).

---

## 🔑 Opção B: Deploy com Chave de API (X-API-Key)

Se preferir autenticar via token em vez de usuário/senha:

1. No Coolify, crie um **Docker Compose** colando o conteúdo de `docker-compose.coolify-auth.yml`:
```yaml
services:
  google-maps-scraper:
    image: gosom/google-maps-scraper:v1.15.0
    container_name: gmaps-scraper
    restart: unless-stopped
    command: ["-web", "-data-folder", "/gmapsdata"]
    expose:
      - "8080"
    volumes:
      - gmaps_data:/gmapsdata
      - gmaps_cache:/opt

  auth-proxy:
    image: caddy:alpine
    container_name: gmaps-auth-proxy
    restart: unless-stopped
    ports:
      - "80:80"
    environment:
      - SCRAPER_API_KEY=${SCRAPER_API_KEY:-minha_chave_secreta}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
    depends_on:
      - google-maps-scraper

volumes:
  gmaps_data:
  gmaps_cache:
```
2. Na aba **Environment Variables** do Coolify, defina:
   ```env
   SCRAPER_API_KEY=sua_chave_super_secreta
   ```
3. Aponte o domínio no Coolify para a porta `80` (porta do proxy Caddy).
4. Faça o **Deploy**.

---

## 💻 Conectando sua Máquina Local ao Coolify

Após o deploy no Coolify, configure o seu arquivo `.env` local na raiz da pasta `google-maps-scraper-kit`:

### Se usou a Opção A (Basic Auth do Coolify):
```env
SCRAPER_BASE_URL=https://scraper.seudominio.com
SCRAPER_USER=admin
SCRAPER_PASSWORD=sua_senha_segura
```
*(Você também pode colocar diretamente na URL: `SCRAPER_BASE_URL=https://admin:sua_senha_segura@scraper.seudominio.com`)*

### Se usou a Opção B (Chave de API):
```env
SCRAPER_BASE_URL=https://scraper.seudominio.com
SCRAPER_API_KEY=sua_chave_super_secreta
```

---

## 🧪 Testando a Conexão

Execute um teste rápido no terminal para validar que o servidor no Coolify está respondendo:

```bash
python -c "import os, urllib.request; print(urllib.request.urlopen(os.environ.get('SCRAPER_BASE_URL', 'https://scraper.seudominio.com') + '/api/v1/jobs').read())"
```

Ou simplesmente rode uma raspagem:

```bash
# Busca com geocodificação automática de cidade
python scripts/scrape.py "padarias" --city "São Paulo, SP" --depth 5

# Busca com coordenadas explícitas e sem extração de e-mails (mais rápido)
python scripts/scrape.py "farmácias" -25.4284 -49.2733 --depth 3 --no-email
```

---

## 🛡️ Dicas de Produção no Coolify

1. **Volume Persistente:** O volume `gmaps_data` montado em `/gmapsdata` garante que o histórico de trabalhos e downloads em CSV/JSON fiquem salvos mesmo se o container reiniciar.
2. **Uso de Proxies:** Para raspagens volumosas (mais de 500 estabelecimentos diários), adicione proxies no comando do script ou no payload do trabalho para não sofrer limitação de taxa por parte do Google.
