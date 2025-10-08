# 📊 Dashboard Analytics BoomLab — Planeamento Final

## 🎯 Objetivo
Dashboard interativo em **HTML único** (Tailwind + Chart.js + Supabase JS) para visualização de dados analíticos do site BoomLab.
- **Deploy**: Vercel
- **Embed**: Notion
- **Auto-refresh**: 2h alinhado com ETL (01:55, 03:55, ..., 23:55)
- **Timezone**: Europe/Lisbon

---

## 📊 Estrutura de Dados

### Tabelas Base (Supabase)

#### 1. `analytics_core`
**Granularidade**: `(date, page_path, country, device_category, city)`

**Constraint correto**:
```sql
UNIQUE (date, page_path, country, device_category, city)  -- COM city
```

**Campos principais**:
- Dimensões: `date, page_path, page_title, country, city, device_category, language`
- Métricas: `total_users, sessions, new_users, average_session_duration, engagement_rate, bounce_rate, screen_pageviews_per_session`

---

#### 2. `analytics_engagement`
**Granularidade**: `(date, page_path, device_category, country, city)`

**Campos principais**:
- Dimensões: `date, page_path, device_category, country, city`
- Métricas: `user_engagement_duration, scrolled_users, event_count, event_count_per_user, engaged_sessions, keyevent_qualify_lead, keyevent_close_convert_lead`

---

#### 3. `analytics_acquisition`
**Granularidade**: `(date, session_source_medium, campaign)`

**Campos principais**:
- Dimensões: `date, session_source_medium, session_channel_group, campaign`
- Métricas: `sessions, total_users, engagement_rate`

**Status**: Pode estar vazia inicialmente (mostrar placeholder elegante)

---

#### 4. `analytics_new_users`
**Granularidade**: `(date, country, device_category)`

**Campos principais**:
- Dimensões: `date, country, device_category`
- Métricas: `new_users`

---

#### 5. `analytics_tech`
**Granularidade**: `(date, hour, device_category, platform)`

**Campos principais**:
- Dimensões: `date, hour, device_category, platform, operating_system, new_vs_returning`
- Métricas: `sessions, avg_session_duration, engaged_sessions, engagement_rate`

**Nota**: Para fase 2 (heatmap)

---

### Views Agregadas (a criar)

#### 1. `view_daily_kpis`
**Propósito**: KPIs globais agregados por dia (evita cálculos client-side complexos)

```sql
CREATE OR REPLACE VIEW view_daily_kpis AS
SELECT
  date,
  SUM(total_users) AS total_users,
  SUM(sessions) AS sessions,
  SUM(engaged_sessions)::float / NULLIF(SUM(sessions), 0) AS engagement_rate,
  SUM(average_session_duration * sessions) / NULLIF(SUM(sessions), 0) AS avg_session_duration,
  1 - (SUM(engaged_sessions)::float / NULLIF(SUM(sessions), 0)) AS bounce_rate
FROM analytics_core
GROUP BY date;
```

**Uso**: Fetch direto para KPIs principais + gráficos de linha temporal

---

#### 2. `view_leads_daily`
**Propósito**: Agregação de leads por dia

```sql
CREATE OR REPLACE VIEW view_leads_daily AS
SELECT
  date,
  SUM(keyevent_qualify_lead) AS qualify_leads,
  SUM(keyevent_close_convert_lead) AS close_leads
FROM analytics_engagement
GROUP BY date;
```

**Uso**: Card de Leads no dashboard

---

#### 3. `view_page_daily` ✅ (já existe)
**Propósito**: Métricas agregadas por página e dia

**Uso**: Tab "Páginas" (ranking, tabela, gráficos)

---

### Views Futuras (fase 2)
- `view_country_daily`: agregação por país + device
- `view_acquisition_daily`: agregação por canal/fonte
- `view_hourly_heatmap`: sessões por hora (para heatmap)

---

## 🧮 Fórmulas dos KPIs

| KPI | Fórmula | Formato | Fonte |
|-----|---------|---------|-------|
| **Total Users** | `SUM(total_users)` | `1.234` | `view_daily_kpis` |
| **Sessions** | `SUM(sessions)` | `5.678` | `view_daily_kpis` |
| **Engagement Rate** | `SUM(engaged_sessions) / SUM(sessions) × 100` | `45,3%` | `view_daily_kpis` |
| **Avg. Session Duration** | `SUM(avg_session_duration × sessions) / SUM(sessions)` | `2:34` (mm:ss) | `view_daily_kpis` |
| **Bounce Rate** | `1 - engagement_rate` | `54,7%` | `view_daily_kpis` |
| **Leads** | `SUM(qualify_leads + close_leads)` | `12` | `view_leads_daily` |

### Variações (Δ%)
- **Cálculo**: `((valor_atual - valor_anterior) / valor_anterior) × 100`
- **Estratégia**: 2 queries paralelas (período atual + período anterior)
- **Período anterior**: mesma duração (7 dias → 7 dias anteriores)
- **Cor**: 
  - Verde ↑ se positivo (exceto Bounce Rate → inverter)
  - Vermelho ↓ se negativo

---

## 🎨 Layout & UI

### Estrutura HTML

```
┌─────────────────────────────────────────────┐
│  📈 Dashboard de Analytics — BoomLab        │
├─────────────────────────────────────────────┤
│  Filtros: [Período ▼] [Página ▼] [País ▼]  │
├─────────────────────────────────────────────┤
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐  │
│  │Users│ │Sess │ │Eng% │ │Dura │ │Leads│  │ 
│  │ 123 │ │ 456 │ │45,3%│ │2:34 │ │ 12  │  │
│  │ ↑5% │ │ ↓2% │ │ ↑8% │ │ ↑3% │ │ ↑15%│  │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘  │
├─────────────────────────────────────────────┤
│  [Visão Geral] [Páginas] [Dispositivos] [Aquisição] │
├─────────────────────────────────────────────┤
│                                             │
│         [Gráficos e Tabelas]                │
│                                             │
├─────────────────────────────────────────────┤
│  Última atualização: 10:55  [🔄 Atualizar] │
└─────────────────────────────────────────────┘
```

---

### Filtros Globais

#### 1. **Período**
Opções rápidas:
- Hoje
- Últimos 7 dias
- Últimos 30 dias
- Últimos 90 dias
- Ano (YTD)
- Custom Range (2× `<input type="date">`)

**Default**: Últimos 7 dias

#### 2. **Página**
- Dropdown com todas as páginas únicas de `analytics_core.page_path`
- Opção: "Todas as páginas" (default)

#### 3. **País**
- Dropdown com todos os países únicos de `analytics_core.country`
- Opção: "Todos os países" (default)

**Persistência**: `localStorage` + query params

---

### Aplicação de Filtros por Tab

| Tab | Período | Página | País |
|-----|---------|--------|------|
| **Visão Geral** | ✅ | ✅ | ✅ |
| **Páginas** | ✅ | ❌ (ignora) | ✅ |
| **Dispositivos & País** | ✅ | ✅ | ✅ |
| **Aquisição** | ✅ | ✅ | ✅ |

---

## 📑 Tabs / Secções

### 1️⃣ Visão Geral
**Conteúdo**:
- KPIs principais (já mostrados acima)
- **Gráfico de linhas** (últimos dias do período):
  - Linha 1: Sessions
  - Linha 2: Total Users
  - Eixo Y secundário: Engagement Rate (%)

**Dados**: `view_daily_kpis` + `view_leads_daily`

---

### 2️⃣ Páginas
**Conteúdo**:
- **Tabela top páginas** (ordenável):
  
  | Página | Users | Sessions | Eng. Rate | Leads | Δ% |
  |--------|-------|----------|-----------|-------|-----|
  | / | 123 | 456 | 45% | 5 | +8% |
  | /contactos | 89 | 234 | 67% | 12 | +15% |

- **Gráfico de barras horizontais**: Top 10 páginas por sessions

**Dados**: `view_page_daily`

**Nota**: Ignora filtro de página (mostra sempre ranking global)

---

### 3️⃣ Dispositivos & País
**Conteúdo**:
- **Gráfico de barras empilhadas**: Sessões por país (stacked por device: desktop/mobile/tablet)
- **Tabela comparativa**:

  | Device | Sessions | Avg. Duration | Engagement Rate |
  |--------|----------|---------------|-----------------|
  | Desktop | 1.234 | 3:45 | 56,7% |
  | Mobile | 890 | 2:12 | 43,2% |

**Dados**: 
- `analytics_core` (agregado por device_category, country)
- `analytics_new_users` (new users por device/country)

---

### 4️⃣ Aquisição
**Conteúdo**:
- **Gráfico donut**: % de sessões por `session_channel_group`
- **Tabela de fontes**:

  | Canal / Fonte | Sessions | Users | Eng. Rate |
  |---------------|----------|-------|-----------|
  | Organic Search | 456 | 234 | 67% |
  | Direct | 123 | 89 | 45% |

**Dados**: `analytics_acquisition`

**Estado vazio**: 
```html
<div class="text-center py-12 text-gray-400">
  <svg>...</svg>
  <p>Sem dados de aquisição ainda</p>
</div>
```

---

## 🔄 Auto-refresh

### Timing
- **Intervalo base**: 2 horas (setInterval)
- **Trigger especial**: Se `minuto ∈ [55-59]` E `hora ímpar` → refresh imediato
- **Manual**: Botão "Atualizar dados"

### Timezone
- **Supabase/n8n**: Europe/Lisbon
- **Browser**: Converte automaticamente para local (usando Day.js com timezone plugin)

### Implementação
```javascript
// Auto-refresh a cada 2h
setInterval(() => fetchAllData(), 2 * 60 * 60 * 1000);

// Check especial a cada minuto
setInterval(() => {
  const now = dayjs().tz('Europe/Lisbon');
  const minute = now.minute();
  const hour = now.hour();
  
  // Horas ímpares: 1, 3, 5, ..., 23
  // Minutos: 55-59
  if (hour % 2 === 1 && minute >= 55 && minute <= 59) {
    if (!lastRefreshInWindow) {
      fetchAllData();
      lastRefreshInWindow = true;
    }
  } else {
    lastRefreshInWindow = false;
  }
}, 60 * 1000);
```

---

## 🎨 Design / Estilo

### Tema
- **Dark mode** moderno (tipo Notion/DataStudio)
- **Background**: `#1a1a1a` ou similar
- **Cards**: `#2a2a2a` com sombra suave
- **Texto**: `#e5e5e5` (primário), `#9ca3af` (secundário)
- **Accent**: azul/verde para positivos, vermelho para negativos

### Componentes
- **KPI Cards**: hover effect, tooltip com fórmula
- **Tabs**: underline ativo, transição suave
- **Gráficos**: Chart.js com tema dark, tooltips customizados
- **Tabelas**: sticky header, hover row, ordenação visual (↑↓)

### Skeletons
Shimmer/pulse enquanto carrega (Tailwind `animate-pulse`)

---

## 📦 Stack Técnico

### CDNs (no `<head>`)
```html
<!-- Tailwind CSS -->
<script src="https://cdn.tailwindcss.com"></script>

<!-- Chart.js -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@4"></script>

<!-- Day.js + timezone plugin -->
<script src="https://cdn.jsdelivr.net/npm/dayjs@1/dayjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/dayjs@1/plugin/timezone.js"></script>
<script src="https://cdn.jsdelivr.net/npm/dayjs@1/plugin/utc.js"></script>

<!-- Supabase JS -->
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

### Configuração Supabase
```javascript
const SUPABASE_URL = 'https://uvvxvqrymdywxoqgnnvd.supabase.co';
const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InV2dnh2cXJ5bWR5d3hvcWdubnZkIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTkwODM3NDAsImV4cCI6MjA3NDY1OTc0MH0.3vNlo61Yl4c3XG7-8yDCKVBO4-f0iNMJeHXSSgzAI5w';
const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

**RLS**: ON (read-only, anon key segura)

---

## 🔧 Estrutura do Código (index.html)

```javascript
// ========== CONFIG ==========
const CONFIG = {
  supabase: { url, key },
  colors: { primary, success, danger, ... },
  locale: 'pt-PT',
  timezone: 'Europe/Lisbon'
};

// ========== FORMATTERS ==========
const fmtNumber = (n) => Intl.NumberFormat('pt-PT').format(n);
const fmtPercent = (n) => `${(n * 100).toLocaleString('pt-PT', {minimumFractionDigits: 1, maximumFractionDigits: 1})}%`;
const fmtDuration = (seconds) => { /* mm:ss */ };
const fmtDate = (date) => dayjs(date).format('DD/MM/YYYY');

// ========== STATE ==========
const state = {
  filters: { period: '7d', page: null, country: null },
  datasets: { kpis: [], leads: [], pages: [], ... },
  cache: new Map(),
  loading: false
};

// ========== UTILS ==========
const getDateRange = (period) => { /* retorna [from, to] */ };
const getPreviousPeriod = (from, to) => { /* período anterior */ };
const calculateDelta = (current, previous) => { /* Δ% */ };

// ========== FETCH ==========
const fetchDailyKPIs = async (from, to, filters) => { /* query view_daily_kpis */ };
const fetchLeads = async (from, to, filters) => { /* query view_leads_daily */ };
const fetchPages = async (from, to, filters) => { /* query view_page_daily */ };
const fetchDevicesCountries = async (...) => { /* query analytics_core */ };
const fetchAcquisition = async (...) => { /* query analytics_acquisition */ };

// ========== COMPUTE ==========
const computeKPIs = (kpisData, leadsData) => {
  return {
    totalUsers: sum(kpisData, 'total_users'),
    sessions: sum(kpisData, 'sessions'),
    engagementRate: weightedAvg(kpisData, 'engagement_rate', 'sessions'),
    avgDuration: weightedAvg(kpisData, 'avg_session_duration', 'sessions'),
    bounceRate: 1 - engagementRate,
    leads: sum(leadsData, 'qualify_leads') + sum(leadsData, 'close_leads')
  };
};

// ========== RENDER ==========
const renderKPICards = (current, previous) => { /* HTML cards com Δ% */ };
const renderLineChart = (data, canvasId) => { /* Chart.js */ };
const renderTable = (data, containerId, columns) => { /* HTML table */ };
const renderBarChart = (data, canvasId) => { /* Chart.js */ };
const renderDonutChart = (data, canvasId) => { /* Chart.js */ };
const renderSkeleton = (containerId) => { /* loading state */ };

// ========== EVENTS ==========
const onFilterChange = () => {
  saveFiltersToLocalStorage();
  updateQueryParams();
  fetchAllData();
};

const onTabSwitch = (tabName) => {
  showTab(tabName);
  renderTabContent(tabName);
};

const onRefreshClick = () => {
  fetchAllData(true); // force refresh
};

// ========== INIT ==========
const init = async () => {
  loadFiltersFromLocalStorage();
  loadFiltersFromQueryParams();
  renderFilterUI();
  await fetchAllData();
  setupAutoRefresh();
};

document.addEventListener('DOMContentLoaded', init);
```

---

## 🗂️ Persistência & Deep-link

### localStorage
```javascript
{
  "filters": {
    "period": "7d",
    "page": "/contactos",
    "country": "Portugal"
  },
  "lastRefresh": "2025-10-07T14:55:00Z"
}
```

### Query Params
```
?period=7d&page=%2Fcontactos&country=Portugal&tab=pages
```

**Prioridade**: Query params > localStorage > defaults

---

## ✅ Checklist de Entrega

### SQL (setup.sql)
- [x] `view_daily_kpis`
- [x] `view_leads_daily`
- [x] Correção do constraint `analytics_core` (documentado)

### HTML (index.html)
- [x] CDNs carregados
- [x] Config Supabase (placeholders)
- [x] Filtros globais funcionais
- [x] 6 KPIs com Δ%
- [x] 4 tabs (Visão Geral, Páginas, Dispositivos, Aquisição)
- [x] Gráficos Chart.js
- [x] Tabelas ordenáveis
- [x] Auto-refresh (2h + trigger especial)
- [x] Skeletons/loading
- [x] localStorage + query params
- [x] Locale pt-PT
- [x] Dark theme Tailwind
- [x] Estado vazio (Aquisição)
- [x] Footer com última atualização

---

## 🚀 Deploy

1. Executar `setup.sql` no Supabase
2. Substituir `SUPABASE_URL` e `SUPABASE_ANON_KEY` no `index.html`
3. Deploy no Vercel:
   ```bash
   vercel --prod
   ```
4. Embed no Notion:
   ```
   /embed https://analytics-boomlab.vercel.app
   ```

---

## 📈 Fase 2 (futuro)

- [ ] Tab "Técnico" com heatmap (chartjs-chart-matrix)
- [ ] Alertas visuais (bounce > 80%, queda > 30%)
- [ ] Webhook n8n para notificações
- [ ] Views adicionais: `view_country_daily`, `view_acquisition_daily`
- [ ] Comparação entre períodos (toggle visual)
- [ ] Export de dados (CSV)
- [ ] Modo claro (toggle theme)

---

**Documento criado**: 2025-10-07  
**Última atualização**: 2025-10-07

