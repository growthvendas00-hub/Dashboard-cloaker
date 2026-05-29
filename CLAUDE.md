# Dashboard-cloaker

## Visão geral

Painel de controle local (HTML puro, zero-dependência) para gerenciar o sistema de cloaking hospedado em `growthvendas00-hub/meu-cloaker` na Vercel. O painel:
- **Edita** o arquivo `api/filtrar.mjs` via GitHub API (link de oferta black)
- **Gera scripts** de integração para sites de origem
- **Monitora** acessos em tempo real com aba Logs e Overview analítico

**Stack:** HTML5 + CSS3 + Vanilla JS (ES2020)  
**Persistência:** `localStorage` (configurações); Upstash Redis (logs via REST API)  
**Deploy:** alterações via GitHub → Vercel redeploy automático

---

## Repositórios

| Repo | Branch | Papel |
|---|---|---|
| `Dashboard-cloaker` | `main` | Este painel — interface do usuário |
| `meu-cloaker` | `main` | API Vercel (`api/filtrar.mjs`) — lógica de detecção |

---

## Arquivos-chave

| Arquivo | Papel |
|---|---|
| `index.html` | **Único arquivo de código.** CSS, HTML, JS em um. Não há imports, bundler ou dependências. |
| `.claude/launch.json` | Config do preview: `npx serve` na porta 3456 |

---

## Estrutura do `index.html`

Três seções:
```
<style>          → variáveis CSS, grid, componentes de cards/tabelas/gráficos
<body>           → sidebar com 5 abas + main
<script>         → toda lógica JS (DOMContentLoaded, event handlers, API calls)
```

### Abas (5 no total)

| Aba | O que faz | Salva em |
|---|---|---|
| **Overview** (padrão) | Stats: total, taxa aprovação, bloqueados; gráficos de Motivos e Dispositivos | — (lê do Redis) |
| **Configuração** | GitHub token, repo, branch, path, Vercel URL | `localStorage`: `gh_token`, `gh_repo`, `gh_branch`, `gh_path`, `vercel_url` |
| **Link Mobile** | Lê `ofertaBlack` atual, edita e faz deploy | GitHub: `api/filtrar.mjs` |
| **Gerar Script** | Monta `<script>` de cloaking com endpoint e whiteUrl | — (copia para clipboard) |
| **Logs** | Tabela de acessos + Limpar dados | Upstash Redis: chave `cloaker_logs` |

### Dados de logs (estrutura)

Cada entrada é JSON com campos curtos:

```json
{
  "t": "2026-05-29T05:00:00Z",    // timestamp ISO
  "ip": "177.12.3.4",             // IP real (x-forwarded-for)
  "dv": "mobile",                 // "mobile" ou "desktop"
  "ac": "approved",               // "approved" ou "blocked"
  "rs": "Visitante aprovado",    // motivo (texto)
  "dst": "https://oferta.com",    // destino final (vazio se bloqueado)
  "ua": "Mozilla/5.0...",         // User-Agent completo
  "cid": "camp_1780...",          // ID da campanha (filtro nos Logs)
  "plt": "facebook"               // plataforma da campanha
}
```

**Compatibilidade:** dashboard também lê campos antigos (`timestamp`, `device`, `action`, `reason`, `destination`) e double-stringify de versões antigas.

### Funções JS principais

| Função | O que faz |
|---|---|
| `loadOverview()` | Busca `lrange` 500 últimos, calcula stats, renderiza gráficos |
| `fetchLogs()` | Busca `lrange` 100 últimos, renderiza tabela com filtro de valores |
| `clearLogs()` | Executa `del` no Redis para limpar toda chave |
| `saveConfig()` | Salva no localStorage (GitHub) |
| `readCurrentFile()` | GET GitHub API, extrai `ofertaBlack` via regex |
| `saveAndDeploy()` | GET SHA, substitui valor, PUT commit no GitHub |
| `generateScript()` | Monta string com `<script>` de cloaking |
| `renderLogs(logs)` | HTML table com formatação de badges |
| `escHtml(s)` | Escapa `&<>"` para evitar XSS |

---

## Sistema de detecção (`meu-cloaker/api/filtrar.mjs`)

A API recebe POST com payload disfarçado de analytics `{e:"pv", c:campaignId, w:screenWidth, l:language, r:referrer}` (UA vem do header). Responde **sempre** com forma neutra: `{ok:true}` (bloqueado/desconhecido) ou `{ok:true, go:blackUrl}` (aprovado). Aceita também nomes antigos (`userAgent`, `campaignId`, `blackUrl`...) para compat.

### 🕵️ Modo furtivo (anti-detecção) — CRÍTICO

Plataformas (Meta/Google/TikTok) detectam cloakers comparando o que o bot vê vs. o usuário. Regras invioláveis:

- **Zero vazamento:** o motivo do bloqueio (`rs`) vai **só pro log do Redis**, NUNCA na resposta HTTP. Bot e visitante recebem resposta idêntica (`{ok:true}`).
- **Oferta vive no servidor:** a black URL **nunca** vai no payload do cliente. Fica no Redis (`campaign:<id>` → `{blackUrl, platform, whiteUrl}`); o servidor resolve pelo `c` (campaignId) e só devolve `go` a quem é aprovado. Bot nunca vê a oferta.
- **Redirect ofuscado:** o script usa `window["loc"+"ation"].replace(g)`, não `window.location.href = url` literal.
- **whiteUrl vazio = stealth máximo:** bot bloqueado fica na página de origem (zero navegação). Só redireciona pro white se whiteUrl for definido.

### Camadas de bloqueio

| Camada | O que bloqueia | Exemplo `rs` (log) |
|---|---|---|
| **1: UA/Device/Idioma** | Bots base; Desktop se `apenasMobile`; idioma ≠ PT | "Bot detectado por UA", "Dispositivo desktop bloqueado", "Idioma incompatível" |
| **1b: screenWidth** | UA mobile + tela ≥ 1200px | "Bot: UA mobile com tela 1920px" |
| **1c: Plataforma** | Bots/IPs específicos da `platform` da campanha (Facebook/Google/TikTok) | "Bot facebook detectado por UA", "IP de verificação google detectado" |
| **2: IP (prefixos)** | Google IPs, Datacenters fixos | "IP pertence ao Google", "IP de Datacenter detectado" |
| **2b: IP intel (opcional)** | hosting/proxy/VPN via ip-api em tempo real. Liga com env `IP_INTEL=1` (grátis) ou `IP_INTEL_KEY=...` (pago) | "IP de proxy/VPN detectado", "IP de hosting/datacenter detectado" |
| **3: Headers** | Chrome sem `Sec-Ch-Ua` (headless) | "Bot: Chrome sem headers de navegador real" |

**Bots base (UA):** googlebot, adsbot, puppeteer, playwright, selenium, webdriver, curl, wget, facebookexternalhit, facebookbot, meta-externalagent, tiktokspider, bytespider, bytedance, whatsapp, applebot, dalvik, okhttp, java/, etc.
**Bots por plataforma:** facebook (instagram, fbav/, google-inspectiontool...), google (storebot-google, googleother, yandex...), tiktok (tiktok/, musical_ly, baiduspider...).

### Script de integração (cloaking)

Gerado **por campanha** (na aba Nova Campanha / botão "Ver Script"). Faz POST com payload de analytics, recebe `{ok, go?}`:
- `go` presente → redireciona (visitante aprovado)
- só `{ok:true}` → fica na origem ou vai pro whiteUrl se definido (bloqueado — sem revelar)

---

## Fluxo de dados

```
Dashboard → salva campanha → Redis SET campaign:<id> {blackUrl, platform, whiteUrl}
Site de origem (script injetado por campanha)
  ↓ POST {e:"pv", c:campaignId, w, l, r}   (SEM a oferta)
Vercel API (filtrar.mjs)
  ↓ GET campaign:<id> no Redis → resolve blackUrl/platform
  ↓ Camadas 1→3, grava log (com cid/plt), responde {ok} ou {ok,go}
Redis Upstash
  ↓ cloaker_logs (LPUSH/LRANGE/DEL) + campaign:<id> (SET/GET/DEL)
Dashboard
  ↓ Home (cards por campanha), Logs (filtro data/campanha/ação)
```

---

## Convenções

- **Tudo fica em `index.html`** — sem arquivos `.js` ou `.css` separados
- **CSS:** variáveis no `:root`; novos estilos sempre com prefixo de componente (`.stat-card`, `.bar-row`)
- **Campos de log:** nomes curtos (`t, ip, dv, ac, rs, dst, ua`) — não mudar, API depende disso
- **Redis:** chave sempre `cloaker_logs`, operações via REST pipeline (`POST /pipeline`)
- **Endpoint GitHub:** `https://api.github.com/repos/{repo}/contents/{path}?ref={branch}`

---

## Otimizações e Tradeoffs

| Decisão | Por quê |
|---|---|
| Sem bundler/framework | Arquivo único, sem build, abre em qualquer navegador |
| CSS grid para gráficos | Barras de progresso com `width` dinâmica — sem Canvas, sem Chart.js |
| `localStorage` para credenciais | Rápido, local, persiste entre aberturas (não está na nuvem) |
| Upstash para logs | Redis gerenciado, sem servidor próprio, API REST (sem SDK) |
| Pipeline Upstash para LPUSH | Atomic: não grava log corrompido mesmo com falha de rede |

---

## O que NÃO tocar

- `.git/` — metadados
- Não adicionar dependências, bundler, framework — mantém o zero-dep
- Não separar CSS/JS em arquivos — projeto é intencionalmente monolítico
- Upstash: chave de lista sempre `cloaker_logs` (em minúsculas)
