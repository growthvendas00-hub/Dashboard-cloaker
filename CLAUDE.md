# Dashboard-cloaker

## Visão geral

Painel de controle local (HTML puro, zero-dependência) para gerenciar o sistema de cloaking hospedado em `growthvendas00-hub/meu-cloaker` na Vercel. O painel:
- **Gerencia campanhas** (criar, editar, deletar) com armazenamento no Redis
- **Gera scripts** de integração por campanha para sites de origem
- **Monitora** acessos em tempo real com Logs + filtros
- **Protege credenciais** com criptografia AES-256 (PBKDF2 + AES-GCM)

**Stack:** HTML5 + CSS3 + Vanilla JS (ES2020)  
**Persistência:** `localStorage` encriptado (credenciais + campanhas cache); Upstash Redis (logs + campanhas via REST API)  
**Deploy:** alterações em `meu-cloaker` via GitHub → Vercel redeploy automático

---

## Repositórios

| Repo | Branch | Papel |
|---|---|---|
| `Dashboard-cloaker` | `main` | Este painel — interface do usuário |
| `meu-cloaker` | `main` | API Vercel (`api/filtrar.mjs`) — lógica de detecção |
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
| **Home** (padrão) | Cards de campanhas com stats ao vivo; botão Ver Script por campanha | Redis: `cloaker_logs` (leitura), `campaign:<id>` (CRUD) |
| **Nova Campanha** | Formulário: nome, plataforma, blackUrl, whiteUrl, anticlone, burnToken | Redis SET + `localStorage` cache |
| **Logs** | Tabela de acessos + filtros (data, campanha, ação) + Limpar | Upstash Redis: chave `cloaker_logs` |
| **Configurações** | Upstash URL/Token, GitHub token/repo/branch/path, Vercel URL + proteção AES-256 | `localStorage` (encriptado se senha mestra definida) |

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
  "plt": "facebook"                 // plataforma da campanha
}
```

**Compatibilidade:** dashboard também lê campos antigos (`timestamp`, `device`, `action`, `reason`, `destination`) e double-stringify de versões antigas.

### Funções JS principais

| Função | O que faz |
|---|---|
| `buildScript(camp)` | Gera `<script>` de cloaking com endpoint `/api/v`, coleta fbc/gclid/ttclid, fingerprint WebGL |
| `saveCampaign()` | Lê form, chama `redisSetCampaign()`, salva em `localStorage`, re-renderiza cards |
| `redisSetCampaign(camp)` | PUT para Upstash `/set/campaign:<id>` com JSON de blackUrl/platform/whiteUrl/anticlone/burnToken |
| `renderCard(camp)` | Card HTML com badge de plataforma, badge anticlone, stats do Redis, botões Editar/Script/Deletar |
| `showScript(id)` | Abre modal com script gerado; botão copiar |
| `toggleBurnOption()` | Mostra/esconde opção "Queimar token" dependendo do checkbox anticlone |
| `fetchLogs()` | Busca `lrange` 100 últimos, renderiza tabela com filtros |
| `clearLogs()` | Executa `del` no Redis para limpar toda a chave de logs |
| `loadOverview()` / stats por campanha | Busca 500 logs, filtra por `cid`, calcula aprovados/bloqueados |
| `setCred(key,val)` / `getCred(key)` | Encrypt/decrypt com AES-GCM se senha mestra definida |
| `escHtml(s)` | Escapa `&<>"` para evitar XSS |

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
| **1: UA/Device/Idioma** | Bots base (27 keywords); Desktop se `apenasMobile`; idioma ≠ PT | "Bot detectado por UA", "Dispositivo desktop bloqueado", "Idioma incompatível" |
| **1b: screenWidth** | UA mobile + tela ≥ 1200px | "Bot: UA mobile com tela 1920px" |
| **1c: Plataforma** | Bots/IPs específicos da `platform` da campanha (Facebook/Google/TikTok) | "Bot facebook detectado por UA", "IP de verificação google detectado" |
| **1d: Fingerprint** | webdriver ativo; SwiftShader/LLVMpipe (headless); GPU desktop + UA mobile (DevTools); mobile sem touchscreen; CPU > 12 cores | "Bot: GPU desktop com UA mobile (emulação DevTools)", "Bot: headless Chrome detectado" |
| **1e: Fingerprint ausente** | Formato novo (`e:'pv'`) + Chrome + sec-ch-ua mas sem campo `gl` = chamada direta à API | "Bot: Chrome sem dados de fingerprint" |
| **2: IP datacenter/Google** | Prefixos de IP do Google e datacenters AWS/Azure/GCP | "IP pertence ao Google", "IP de datacenter detectado" |
| **2b: IP intel (opcional)** | hosting/proxy/VPN via ip-api em tempo real. Liga com env `IP_INTEL=1` | "IP de proxy/VPN detectado", "IP de hosting detectado" |
| **3: Headers** | Chrome sem `Sec-Ch-Ua` (headless Chrome) | "Bot: Chrome sem headers de navegador real" |

**Bots base (UA):** googlebot, adsbot, puppeteer, playwright, selenium, webdriver, curl, wget, facebookexternalhit, facebookbot, meta-externalagent, tiktokspider, bytespider, bytedance, whatsapp, applebot, dalvik, okhttp, java/, etc.

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
Dashboard → saveCampaign() → Redis SET campaign:<id> {blackUrl, platform, whiteUrl, anticlone, burnToken}
Site de origem (script no <head>)
  ↓ lê fbclid/gclid/ttclid da URL
  ↓ POST {e:"pv", c:campaignId, fbc:token, w, l, r, gl, mem, cpu, tp, wd}
Vercel API (filtrar.mjs / /api/v)
  ↓ GET campaign:<id> no Redis → resolve blackUrl/platform/anticlone/burnToken
  ↓ CAMADA 0: valida token; SETNX se burnToken (Redis cloak_tkn:...)
  ↓ CAMADAs 1→3: UA, fingerprint, IP, headers
  ↓ grava log (com cid/plt), responde {ok} ou {ok,go}
Redis Upstash
  ↓ cloaker_logs (LPUSH/LRANGE/DEL) + campaign:<id> (SET/GET) + cloak_tkn:* (SETNX/TTL)
Dashboard
  ↓ Home (cards com stats por campanha), Logs (filtros data/campanha/ação)
```

---

## Convenções

- **Tudo fica em `index.html`** — sem arquivos `.js` ou `.css` separados
- **CSS:** variáveis no `:root`; novos estilos sempre com prefixo de componente (`.stat-card`, `.bar-row`)
- **Campos de log:** nomes curtos (`t, ip, dv, ac, rs, dst, ua, cid, plt`) — não mudar, API depende disso
- **Redis:** chave sempre `cloaker_logs`, operações via REST pipeline (`POST /pipeline`)
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
- Upstash: chave de lista sempre `cloaker_logs` (em minúsculas)
- Salt `cloaker_salt_v2` — mudar quebra credenciais já cifradas no localStorage dos usuários
- Não colocar blackUrl, whiteUrl ou credenciais hardcoded em nenhum arquivo do repo

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
