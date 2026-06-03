# Dashboard-cloaker

## Visão geral

Painel de controle (HTML puro, zero-dependência) para gerenciar o sistema de cloaking hospedado em `growthvendas00-hub/meu-cloaker` na Vercel. O painel:
- **Multi-usuário** com login e isolamento real por conta (cada um só vê os seus dados)
- **Gerencia campanhas** (criar, editar, deletar) via API autenticada `/api/panel`
- **Gera scripts** de integração por campanha para sites de origem
- **Monitora** acessos em tempo real com Logs + filtros (escopados por conta)
- **Protege credenciais** — token do Upstash fica só no servidor, nunca no navegador

**Stack:** HTML5 + CSS3 + Vanilla JS (ES2020)  
**Auth/Dados:** `/api/panel` (Vercel) — sessão HMAC-SHA256, senhas PBKDF2; Upstash Redis server-side  
**Deploy:** alterações em `meu-cloaker` via GitHub → Vercel redeploy automático

### 🔐 Sistema multi-usuário (isolamento server-side)

- **Login obrigatório:** primeiro acesso cria a conta **admin** (action `setup`); depois exige login.
- **Só admin cria contas:** seção "Usuários" em Configurações (criar/excluir, papel user/admin).
- **Isolamento real:** o token do Upstash **nunca** vai ao navegador — fica nas env vars da Vercel. O `/api/panel` valida a sessão e só devolve os dados da conta logada. Um amigo **não consegue** ver os logs/campanhas de outra conta, nem pelo DevTools.
- **Sessão stateless:** token assinado com HMAC-SHA256 (`AUTH_SECRET` ou, na ausência, o próprio token do Upstash), validade 12h, guardado em `localStorage` (`_sess`).
- **Senhas:** PBKDF2-SHA256 (100k iterações) + salt por usuário, gravadas no Redis hash `cloaker_accounts`.
- **Escopo dos dados:** campanhas têm campo `account`; logs vão para `cloaker_logs:<conta>`. Admin adota campanhas legadas órfãs (sem `account`) automaticamente no primeiro `listCampaigns`.

---

## Repositórios

| Repo | Branch | Papel |
|---|---|---|
| `Dashboard-cloaker` | `main` | Este painel — interface do usuário |
| `meu-cloaker` | `main` | API Vercel: `api/filtrar.mjs` (detecção) + `api/panel.mjs` (auth/dados) |
| `cloak` | `main` | White page React/Vite (`burgzdelivery.vercel.app`) — tem script de cloaking no `<head>` |

**URL do dashboard:** `https://dashboard-cloaker.vercel.app`  
**URL da API ao vivo:** `https://meu-cloaker.vercel.app/api/v` (endpoint neutro)  
**URL da white page:** `https://burgzdelivery.vercel.app`

---

## Arquivos-chave

| Arquivo | Papel |
|---|---|
| `index.html` | **Único arquivo de código.** CSS, HTML, JS em um. Sem imports, bundler ou dependências. |
| `.claude/launch.json` | Config do preview: `npx serve` na porta 3456 |

---

## Estrutura do `index.html`

Três seções:
```
<style>          → variáveis CSS, grid, componentes de cards/tabelas/gráficos
<body>           → sidebar com 4 abas + main + modal overlay
<script>         → toda lógica JS (DOMContentLoaded, event handlers, API calls)
```

### Abas (4 no total)

| Aba | O que faz | Salva em |
|---|---|---|
| **Home** (padrão) | Cards de campanhas com stats ao vivo; botão Ver Script por campanha | `/api/panel` → Redis `campaign:<id>` + `cloaker_logs:<conta>` |
| **Nova Campanha** | Formulário: nome, plataforma, blackUrl, whiteUrl, anticlone, burnToken | `/api/panel` action `saveCampaign` (escopado à conta) |
| **Logs** | Tabela (data, IP, país, ação) + filtros + Limpar | `/api/panel` action `getLogs`/`clearLogs` |
| **Configurações** | URL da API, GitHub token/repo/branch/path, proteção AES-256, **Usuários (admin)** | `localStorage` (vercel_url plano; GitHub cifrável) + `/api/panel` (contas) |

> A aba só aparece **após o login**. O overlay de auth (`#auth-overlay`) cobre tudo até autenticar; o app (`#app-root`) só é exibido por `enterApp()`.

### Criptografia de credenciais (AES-256)

- Chave derivada via **PBKDF2** (100k iterações, SHA-256) de uma senha mestra opcional
- Valores cifrados com **AES-GCM 256-bit** + IV aleatório por valor
- Armazenados no `localStorage` com prefixo `E2:` — inúteis sem a senha mestra
- Senha mestra fica apenas em `sessionStorage` (apagada ao fechar o browser)
- Constante de salt: `cloaker_salt_v2` (não mudar — quebraria credenciais já cifradas)

### Dados de logs (estrutura)

Cada entrada é JSON com campos curtos:

```json
{
  "t":   "2026-05-29T05:00:00Z",    // timestamp ISO
  "ip":  "177.12.3.4",              // IP real (x-forwarded-for)
  "dv":  "mobile",                  // "mobile" ou "desktop"
  "ac":  "approved",                // "approved" ou "blocked"
  "rs":  "Visitante aprovado",      // motivo (texto) — NUNCA vai na resposta HTTP
  "dst": "https://oferta.com",      // destino final (vazio se bloqueado)
  "ua":  "Mozilla/5.0...",          // User-Agent completo
  "cid": "camp_1780...",            // ID da campanha (filtro nos Logs)
  "plt": "facebook",                // plataforma da campanha
  "acc": "joao",                    // conta dona da campanha (isolamento)
  "co":  "BR"                       // país (preenchido se IP_INTEL ativo)
}
```

**Compatibilidade:** dashboard também lê campos antigos (`timestamp`, `device`, `action`, `reason`, `destination`) e double-stringify de versões antigas.

### Funções JS principais

| Função | O que faz |
|---|---|
| `rawPanel(action,payload)` | POST cru a `/api/panel` com `Authorization: Bearer <sess>` (status/login/setup/me) |
| `panelCall(action,payload)` | Igual, mas se `authRequired` → limpa sessão e mostra login |
| `initAuth()` | Gate de entrada: valida sessão (`me`), senão decide login vs setup (via `status`) |
| `doLogin()` / `doSetup()` / `logout()` | Fluxos de autenticação; guardam token em `localStorage._sess` |
| `enterApp()` | Esconde overlay, mostra app, popula sidebar, exibe seção admin se `_myRole==='admin'` |
| `loadCampaigns()` | `panelCall('listCampaigns')` → cache `window._campaigns` (mapeia `blackUrl`→`ofertaBlack`) |
| `saveCampaign()` | `panelCall('saveCampaign',{campaign})` (escopado à conta), re-renderiza |
| `deleteCampaign(id)` | `panelCall('deleteCampaign',{id})` (valida posse no servidor) |
| `buildScript(camp)` | Gera `<script>` de cloaking com `apiBase()+'/api/v'`, coleta fbc/gclid/ttclid, fingerprint WebGL |
| `fetchAllLogs(limit)` | `panelCall('getLogs',{limit})` → logs da própria conta |
| `clearLogs()` | `panelCall('clearLogs')` — limpa só os logs da conta |
| `loadAccounts()` / `createAccountUI()` / `deleteAccountUI(u)` | Gestão de usuários (só admin) |
| `setCred(key,val)` / `getCred(key)` | Encrypt/decrypt AES-GCM do token GitHub (opcional, senha mestra) |
| `escHtml(s)` | Escapa `&<>"` para evitar XSS |

---

## API de painel (`meu-cloaker/api/panel.mjs`)

Endpoint **autenticado** que dá acesso a campanhas e logs com isolamento por conta. POST `{action, ...}` com `Authorization: Bearer <token>`. Sempre responde `{ok:true,...}` ou `{ok:false, error, authRequired?}`.

| Action | Auth | O que faz |
|---|---|---|
| `status` | — | `{configured}` — já existe admin? |
| `setup` | — | Cria primeiro admin (só se não houver contas). Retorna token. |
| `login` | — | Valida senha (PBKDF2 + timing-safe). Retorna token + role. |
| `me` | sessão | Retorna `{username, role}` da sessão |
| `listCampaigns` | sessão | Campanhas da conta. Admin adota órfãs (migração legada). |
| `saveCampaign` | sessão | Cria/edita; valida posse. Grava `account` + `acct_camps:<u>`. |
| `deleteCampaign` | sessão | Remove campanha (valida posse) |
| `getLogs` | sessão | `LRANGE cloaker_logs:<conta>`. Admin também lê legado global. |
| `clearLogs` | sessão | `DEL` da lista de logs da conta |
| `listAccounts` / `createAccount` / `deleteAccount` | **admin** | Gestão de usuários |

**Segredos:** `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`, `AUTH_SECRET` (opcional — cai no token do Upstash) — todas env vars na Vercel, nunca no cliente.

---

## Sistema de detecção (`meu-cloaker/api/filtrar.mjs`)

A API recebe POST com payload disfarçado de analytics:
```json
{"e":"pv","c":"<campaignId>","fbc":"<fbclid|gclid|ttclid>","w":390,"l":"pt-BR","r":"<referrer>","gl":"<WebGL renderer>","mem":4,"cpu":8,"tp":5,"wd":false}
```
UA vem do header (não do body). Responde **sempre** com forma neutra: `{ok:true}` (bloqueado/desconhecido) ou `{ok:true, go:blackUrl}` (aprovado).

### 🕵️ Modo furtivo (anti-detecção) — CRÍTICO

- **Zero vazamento:** o motivo do bloqueio (`rs`) vai **só pro log do Redis**, NUNCA na resposta HTTP.
- **Resposta idêntica:** bot e visitante recebem exatamente `{ok:true}`. Aprovado recebe `{ok:true, go:url}`.
- **Oferta vive no servidor:** black URL **nunca** vai no payload do cliente. Fica no Redis `campaign:<id>`; servidor resolve pelo `c` (campaignId).
- **Redirect ofuscado:** script usa `window["loc"+"ation"]["rep"+"lace"](g)`, não `window.location.href`.
- **whiteUrl vazio = stealth máximo:** bot bloqueado fica na página de origem (zero navegação).
- **Endpoint neutro:** `/api/v` (não revela função de filtro/cloaking). `/api/filtrar` mantém compat com scripts antigos.
- **GET = neutro:** requisições não-POST retornam `{ok:true}` (não revelam que é uma API especial).

### Camadas de bloqueio

| Camada | O que bloqueia | Exemplo `rs` (log) |
|---|---|---|
| **0: Anticlone** | Acesso sem token de clique (fbclid/gclid/ttclid); token já queimado (uso único) | "Anticlone: acesso sem token de clique do anúncio", "Anticlone: token de clique já utilizado" |
| **1: UA/Device/Idioma** | Bots base + **leet** (`Googl3bot`→normaliza→`googlebot`); Desktop se `apenasMobile`; idioma ≠ PT | "Bot detectado por UA", "Dispositivo desktop bloqueado", "Idioma incompatível" |
| **1b: screenWidth** | UA mobile + tela ≥ 1200px | "Bot: UA mobile com tela 1920px" |
| **1c: Plataforma** | Bots/IPs específicos da `platform` (Facebook/Google/TikTok/**Kwai**) | "Bot facebook detectado por UA", "IP de verificação google detectado" |
| **1d: Fingerprint** | webdriver ativo; SwiftShader/LLVMpipe (headless); GPU desktop + UA mobile (DevTools); mobile sem touchscreen; CPU > 12 cores | "Bot: GPU desktop com UA mobile (emulação DevTools)" |
| **1e: Fingerprint ausente** | Formato novo (`e:'pv'`) + Chrome + sec-ch-ua mas sem campo `gl` = chamada direta à API | "Bot: Chrome sem dados de fingerprint" |
| **1f: Client Hints cruzados** | `sec-ch-ua-mobile`/`sec-ch-ua-platform` contradizem o UA (ex: hint desktop + UA mobile) | "Bot: Client Hints desktop mas UA diz mobile (emulação)" |
| **2: IP datacenter/Google** | Prefixos de IP do Google e datacenters AWS/Azure/GCP | "IP pertence ao Google", "IP de datacenter detectado" |
| **2b: IP intel (opcional)** | hosting/proxy/VPN via ip-api; também grava país (`co`). Liga com env `IP_INTEL=1` | "IP de proxy/VPN detectado", "IP de hosting detectado" |
| **2c: Reverse DNS** | PTR do IP revela domínio de bot (googlebot.com, fbsv.net, tiktok.com…). Timeout 600ms, fail-open | "Bot: Reverse DNS revela googlebot.com" |
| **3: Headers** | Chrome sem `Sec-Ch-Ua` (headless Chrome) | "Bot: Chrome sem headers de navegador real" |

**Bots base (UA):** googlebot, adsbot, puppeteer, playwright, selenium, webdriver, curl, wget, facebookexternalhit, meta-externalagent, tiktokspider, bytespider, **kwaispider/kuaishou**, whatsapp, applebot, dalvik, okhttp, java/, e **bots de IA**: gptbot, claudebot, amazonbot, bingbot, perplexitybot, ccbot, cohere, etc.

### 🔒 Anticlone (CAMADA 0)

Exige que o visitante tenha **clicado diretamente no anúncio** (não link copiado/compartilhado).

- **Ativa por campanha:** campo `anticlone: true` no Redis `campaign:<id>`
- **Token:** fbclid (Facebook Ads), gclid (Google Ads), ttclid (TikTok Ads) — coletado da URL pelo script
- **Burn token (uso único):** `burnToken: true` — token queimado no Redis via `SETNX` atômico com TTL 48h
  - Chave: `cloak_tkn:{campaignId}:{últimos40chars do token}`
  - Primeiro uso: `SETNX` retorna `OK` → aprovado
  - Segundo uso: `SETNX` retorna `null` → bloqueado ("token já utilizado")
- **Só valida no formato novo** (`e:'pv'`) — scripts legados não são afetados

### Script de integração (cloaking)

Gerado **por campanha** (botão "Ver Script" no card). Coleta:
- Token de clique (`fbclid` / `gclid` / `ttclid` da URL → campo `fbc`)
- Fingerprint: WebGL renderer, deviceMemory, hardwareConcurrency, maxTouchPoints, webdriver, screenWidth, language, devicePixelRatio, connection type
- Guard `sessionStorage._ck` — previne loop infinito e re-execução na mesma sessão

Comportamento após resposta:
- `go` presente → redireciona para black page (visitante aprovado)
- Só `{ok:true}` → fica na origem, ou vai pro whiteUrl se for **domínio diferente** (evita loop)
- Erro de rede → remove `_ck` para permitir retry

---

## Fluxo de dados

```
Dashboard (logado) → panelCall('saveCampaign') → /api/panel
  ↓ valida sessão (HMAC) → Redis SET campaign:<id> {blackUrl, platform, whiteUrl, anticlone, burnToken, account}
  ↓ SADD acct_camps:<conta> <id>
Site de origem (script no <head>)
  ↓ lê fbclid/gclid/ttclid da URL
  ↓ POST {e:"pv", c:campaignId, fbc:token, w, l, r, gl, mem, cpu, tp, wd}  →  /api/v (filtrar.mjs)
  ↓ GET campaign:<id> → resolve blackUrl/platform/anticlone/burnToken/account
  ↓ define LOG_KEY = cloaker_logs:<account>  (isolamento dos logs por conta)
  ↓ CAMADA 0: valida token; SETNX se burnToken (cloak_tkn:...)
  ↓ CAMADAs 1→3: UA(+leet), fingerprint(+Client Hints), IP(+reverse DNS), headers
  ↓ grava log (cid/plt/acc/co), responde {ok} ou {ok,go}
Redis Upstash
  ↓ cloaker_accounts (HASH) + campaign:<id> + acct_camps:<u> (SET)
  ↓ cloaker_logs:<conta> (LPUSH/LRANGE/DEL) + cloak_tkn:* (SETNX/TTL)
Dashboard (logado)
  ↓ panelCall('listCampaigns'|'getLogs') → só os dados da conta logada
```

---

## Convenções

- **Tudo fica em `index.html`** — sem arquivos `.js` ou `.css` separados
- **CSS:** variáveis no `:root`; novos estilos sempre com prefixo de componente (`.stat-card`, `.bar-row`)
- **Campos de log:** nomes curtos (`t, ip, dv, ac, rs, dst, ua, cid, plt, acc, co`) — não mudar, API depende disso
- **Chaves Redis:** `cloaker_logs:<conta>` (logs por usuário), `campaign:<id>`, `acct_camps:<conta>` (SET), `cloaker_accounts` (HASH de contas), `cloak_tkn:*` (burn). Legado: `cloaker_logs` (global, lido pelo admin)
- **`/api/panel`:** comando Redis via JSON array na raiz REST; pipeline em `POST /pipeline`
- **Endpoint GitHub:** `https://api.github.com/repos/{repo}/contents/{path}?ref={branch}`
- **Credenciais:** nunca hardcoded em arquivos — usar `setCred`/`getCred` com AES-GCM
- **Salt de criptografia:** `cloaker_salt_v2` — não alterar (quebraria credenciais existentes)

---

## Otimizações e Tradeoffs

| Decisão | Por quê |
|---|---|
| Sem bundler/framework | Arquivo único, sem build, abre em qualquer navegador |
| CSS grid para gráficos | Barras de progresso com `width` dinâmica — sem Canvas, sem Chart.js |
| AES-GCM para credenciais | Criptografia nativa (WebCrypto API), sem libs, zero-dep |
| Sessão HMAC stateless | Sem store de sessão; valida na assinatura. Reusa token Upstash como segredo |
| Dados via `/api/panel` (não Upstash direto) | Token do Redis nunca vai ao navegador — isolamento real entre contas |
| Upstash para logs e campanhas | Redis gerenciado, sem servidor próprio, API REST (sem SDK) |
| Pipeline Upstash para LPUSH | Atomic: não grava log corrompido mesmo com falha de rede |
| `SETNX` para burn de tokens | Atômico: garante uso único mesmo com requests paralelas |
| `sessionStorage._ck` como guard | Previne loop infinito sem cookies ou storage compartilhada |
| Endpoint `/api/v` neutro | Não revela "filtrar/cloaking" em português no nome da rota |

---

## O que NÃO tocar

- `.git/` — metadados
- Não adicionar dependências, bundler, framework — mantém o zero-dep
- Não separar CSS/JS em arquivos — projeto é intencionalmente monolítico
- Upstash: logs em `cloaker_logs:<conta>` (legado global `cloaker_logs`) — sempre minúsculas
- Salt `cloaker_salt_v2` — mudar quebra credenciais já cifradas no localStorage dos usuários
- Não colocar blackUrl, whiteUrl ou credenciais hardcoded em nenhum arquivo do repo
- **Nunca** expor o token do Upstash no cliente — dados sempre via `/api/panel` autenticado
- `cloaker_accounts` (HASH) guarda senhas (PBKDF2) — não logar nem expor o campo `ph`

---

## Resultados dos testes de bot detection (2026-05-30)

Todos os 21 cenários testados contra `https://meu-cloaker.vercel.app/api/v`:

| Cenário | Resultado esperado | Status |
|---|---|---|
| GET request | BLOQUEADO `{ok:true}` neutro | ✅ |
| Googlebot UA | BLOQUEADO | ✅ |
| facebookexternalhit UA | BLOQUEADO | ✅ |
| Desktop Windows (apenasMobile) | BLOQUEADO | ✅ |
| DevTools mobile emulation (GPU desktop + UA mobile) | BLOQUEADO | ✅ |
| webdriver=true | BLOQUEADO | ✅ |
| SwiftShader (headless Chrome) | BLOQUEADO | ✅ |
| Idioma en-US | BLOQUEADO | ✅ |
| UA mobile + tela 1920px | BLOQUEADO | ✅ |
| Chrome sem sec-ch-ua | BLOQUEADO | ✅ |
| Campo `gl` ausente (chamada direta API) | BLOQUEADO | ✅ |
| curl UA | BLOQUEADO | ✅ |
| Campanha inexistente (sem vazar destino) | BLOQUEADO | ✅ |
| Visitante real mobile BR (Adreno 650) | APROVADO | ✅ |
| Visitante real mobile BR (Mali-G78) | APROVADO | ✅ |
| Anticlone: sem fbclid | BLOQUEADO | ✅ |
| Anticlone: fbclid válido (1ª vez) | APROVADO | ✅ |
| Anticlone: mesmo fbclid (2ª vez - burn) | BLOQUEADO | ✅ |
| Sem campaignId (script legado) | BLOQUEADO | ✅ |
| tp=0 em mobile (sem touchscreen) | BLOQUEADO | ✅ |
| CPU=16 em mobile (servidor) | BLOQUEADO | ✅ |
