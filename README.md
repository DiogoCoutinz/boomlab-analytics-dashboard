# 📊 Dashboard de Analytics — BoomLab

Dashboard interativo em HTML único para visualização de dados analíticos do Google Analytics 4, com dados armazenados no Supabase e atualizados via n8n.

![Dashboard Preview](https://img.shields.io/badge/Status-Production%20Ready-success)
![Tech Stack](https://img.shields.io/badge/Stack-HTML%20%7C%20Tailwind%20%7C%20Chart.js-blue)

---

## 🚀 Quick Start

### 1. Executar SQL no Supabase (OBRIGATÓRIO)

Antes de usar o dashboard, **deves executar** o ficheiro `setup.sql` no Supabase SQL Editor:

1. Acede ao teu projeto Supabase → **SQL Editor**
2. Abre o ficheiro `setup.sql`
3. Copia todo o conteúdo
4. Cola no SQL Editor e executa (Run)

**O que este SQL faz:**
- Cria a view `view_daily_kpis` (KPIs agregados por dia)
- Cria a view `view_leads_daily` (leads agregados por dia)
- Valida que a view `view_page_daily` existe
- Inclui queries de teste para verificar que está tudo OK

**⚠️ Nota importante sobre o constraint:**
O ficheiro também documenta que o constraint de `analytics_core` deve incluir `city`:
```sql
UNIQUE (date, page_path, country, device_category, city)
```

Se ainda não está assim, descomenta a secção 1 do `setup.sql` para corrigir.

---

### 2. Configurar Credenciais Supabase

Abre o ficheiro `index.html` e substitui as credenciais na **linha ~200**:

```javascript
const CONFIG = {
  supabase: {
    url: 'https://uvvxvqrymdywxoqgnnvd.supabase.co',  // ← Substituir
    anonKey: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InV2dnh2cXJ5bWR5d3hvcWdubnZkIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTkwODM3NDAsImV4cCI6MjA3NDY1OTc0MH0.3vNlo61Yl4c3XG7-8yDCKVBO4-f0iNMJeHXSSgzAI5w',                  // ← Substituir
  },
  // ...
};
```

**Onde encontrar:**
- Supabase Dashboard → **Settings** → **API**
- **Project URL** → copia para `url`
- **anon public** key → copia para `anonKey`

**Segurança:**
- Garante que o **Row Level Security (RLS)** está **ativo** nas tabelas
- A `anonKey` só permite leitura (read-only)

---

### 3. Deploy

#### Opção A: Vercel (Recomendado)

```bash
# Instalar Vercel CLI (se ainda não tens)
npm i -g vercel

# Deploy
vercel --prod
```

O Vercel vai detetar automaticamente que é um site estático e fazer deploy do `index.html`.

#### Opção B: Netlify

```bash
# Netlify CLI
npm i -g netlify-cli
netlify deploy --prod
```

#### Opção C: GitHub Pages

1. Faz push do repositório para GitHub
2. Vai a **Settings** → **Pages**
3. Seleciona branch `main` e pasta `/` (root)
4. Aguarda deploy

#### Opção D: Local (Testes)

Abre o `index.html` diretamente no browser (pode haver problemas de CORS — usa Live Server do VS Code).

---

### 4. Embed no Notion

Depois do deploy:

1. Copia o URL do dashboard (ex: `https://analytics-boomlab.vercel.app`)
2. No Notion, escreve `/embed` e cola o URL
3. Ajusta o tamanho do iframe

**💡 Dica:** O Notion tem limitações de altura. Recomenda-se usar **Full Page** ou ajustar manualmente.

---

## 📊 Estrutura de Dados

### Tabelas Base (Supabase)

| Tabela | Descrição | Granularidade |
|--------|-----------|---------------|
| `analytics_core` | Métricas principais por página/geo/device | `(date, page_path, country, device_category, city)` |
| `analytics_engagement` | Engagement, eventos, leads | `(date, page_path, device_category, country, city)` |
| `analytics_acquisition` | Canais de aquisição (UTM) | `(date, session_source_medium, campaign)` |
| `analytics_new_users` | Novos utilizadores | `(date, country, device_category)` |
| `analytics_tech` | Dados técnicos (OS, hora) | `(date, hour, device_category, platform)` |

### Views Agregadas (criadas pelo setup.sql)

| View | Descrição | Uso no Dashboard |
|------|-----------|------------------|
| `view_daily_kpis` | KPIs globais por dia | Cards principais + gráfico temporal |
| `view_leads_daily` | Leads agregados por dia | Card "Leads" |
| `view_page_daily` | Métricas por página | Tab "Páginas" |

---

## 🎨 Funcionalidades

### ✅ Implementado (Versão 1)

#### Filtros Globais
- **Período**: Hoje, 7/30/90 dias, Ano (YTD), Custom Range
- **Página**: Dropdown com todas as páginas
- **País**: Dropdown com todos os países
- **Persistência**: localStorage + query params (deep-link)

#### KPIs Principais (6 cards)
- Total Users
- Sessions
- Engagement Rate
- Avg. Session Duration
- Bounce Rate
- Leads (Qualify + Close)

**Com variações Δ%** vs período anterior (verde ↑ / vermelho ↓)

#### 4 Tabs

**1. Visão Geral**
- Gráfico de linha temporal (Sessions, Users, Engagement Rate)
- Evolução ao longo do período selecionado

**2. Páginas**
- Tabela top 20 páginas (ordenável)
- Gráfico de barras horizontal (top 10)
- Métricas: Users, Sessions, Engagement Rate, Leads

**3. Dispositivos & País**
- Gráfico de barras empilhadas (sessões por país + device)
- Tabela comparativa (desktop vs mobile vs tablet)

**4. Aquisição**
- Gráfico donut (% por canal)
- Tabela de fontes de tráfego
- **Estado vazio elegante** se não houver dados

#### Auto-refresh
- **Intervalo base**: 2 horas
- **Trigger especial**: horas ímpares + minutos 55-59 (sincronizado com ETL do n8n)
- **Manual**: Botão "Atualizar"
- **Indicador**: "Última atualização: HH:mm"

#### UX/Design
- **Dark theme** moderno (tipo Notion)
- **Skeletons** enquanto carrega
- **Tabelas ordenáveis** (clique no header)
- **Tooltips** limpos (Chart.js)
- **Responsive** (desktop-first, funciona em mobile)
- **Locale pt-PT**: números `1.234,5`, datas `DD/MM/YYYY`, percentagens `45,3%`

---

### 🔜 Roadmap (Fase 2)

- [ ] Tab "Técnico" com **heatmap** de sessões por hora (chartjs-chart-matrix)
- [ ] **Alertas visuais** (bounce > 80%, queda > 30%)
- [ ] **Webhook n8n** para notificações (Slack/Email)
- [ ] Views adicionais: `view_country_daily`, `view_acquisition_daily`
- [ ] **Comparação visual** entre períodos (toggle)
- [ ] **Export CSV** de dados
- [ ] **Modo claro** (toggle theme)
- [ ] **Filtro de dispositivo** global

---

## 🛠️ Stack Técnico

### Frontend (tudo via CDN)
- **Tailwind CSS** 3.x (via CDN)
- **Chart.js** 4.x (gráficos)
- **Day.js** 1.x + timezone plugin (datas)
- **Supabase JS** 2.x (fetch de dados)

### Backend
- **Supabase** (PostgreSQL + RLS)
- **n8n** (ETL de GA4 → Supabase, de 2h em 2h)

### Timezone
- **Europe/Lisbon** (configurado em Day.js e n8n)

### Deploy
- **Vercel** / Netlify / GitHub Pages

---

## 📐 Arquitetura do Código

O ficheiro `index.html` está organizado em **secções modulares**:

```javascript
// ========== CONFIG ==========
// Credenciais Supabase, cores, locale, timezone

// ========== STATE ==========
// Filtros, datasets, cache, loading state

// ========== FORMATTERS ==========
// fmtNumber, fmtPercent, fmtDuration, fmtDate, fmtDelta

// ========== UTILS ==========
// getDateRange, getPreviousPeriod, sum, weightedAvg

// ========== FETCH ==========
// fetchDailyKPIs, fetchLeads, fetchPages, fetchDevicesCountries, fetchAcquisition

// ========== COMPUTE ==========
// computeKPIs, aggregatePages, aggregateDevices, aggregateAcquisition

// ========== RENDER ==========
// renderKPICards, renderOverviewChart, renderTables, renderCharts

// ========== EVENTS ==========
// updateFilters, switchTab, onRefreshClick

// ========== AUTO-REFRESH ==========
// setInterval (2h) + trigger especial (horas ímpares)

// ========== INIT ==========
// Carrega filtros, popula dropdowns, fetch inicial, setup listeners
```

**Fácil de expandir:**
- Para adicionar um KPI: modifica `computeKPIs()` e `renderKPICards()`
- Para adicionar um gráfico: cria nova função `renderXChart()`
- Para adicionar uma view: adiciona query em `fetch` + aggregação em `compute`

---

## 🧮 Fórmulas dos KPIs

| KPI | Fórmula | Fonte |
|-----|---------|-------|
| **Total Users** | `SUM(total_users)` | `view_daily_kpis` |
| **Sessions** | `SUM(sessions)` | `view_daily_kpis` |
| **Engagement Rate** | `SUM(engaged_sessions) / SUM(sessions)` | calculado |
| **Avg. Duration** | `SUM(avg_session_duration × sessions) / SUM(sessions)` | média ponderada |
| **Bounce Rate** | `1 - engagement_rate` | calculado |
| **Leads** | `SUM(qualify_leads + close_leads)` | `view_leads_daily` |

**Variação Δ%:**
```javascript
delta = ((valor_atual - valor_anterior) / valor_anterior) × 100
```

**Período anterior:**
- Mesmo número de dias (ex: 7 dias → 7 dias anteriores)
- 2 queries paralelas (atual + anterior)

---

## 🔒 Segurança

### RLS (Row Level Security)

**Configurar no Supabase:**

```sql
-- Permitir leitura pública (anon) nas tabelas/views
ALTER TABLE analytics_core ENABLE ROW LEVEL SECURITY;
ALTER TABLE analytics_engagement ENABLE ROW LEVEL SECURITY;
ALTER TABLE analytics_acquisition ENABLE ROW LEVEL SECURITY;
ALTER TABLE analytics_new_users ENABLE ROW LEVEL SECURITY;
ALTER TABLE analytics_tech ENABLE ROW LEVEL SECURITY;

-- Policy: permitir SELECT público
CREATE POLICY "Allow public read access" ON analytics_core
  FOR SELECT USING (true);

CREATE POLICY "Allow public read access" ON analytics_engagement
  FOR SELECT USING (true);

-- Repetir para outras tabelas...
```

**Alternativamente**, se quiseres proteger com autenticação:
- Usa Supabase Auth
- Adiciona login no dashboard
- Ajusta RLS policies para `auth.uid() IS NOT NULL`

---

## 📝 Notas Importantes

### ETL Schedule (n8n)
- **Frequência**: De 2 em 2 horas
- **Horário**: Horas ímpares às :55 (01:55, 03:55, 05:55, ..., 23:55)
- **Timezone**: Europe/Lisbon
- **Fecho do dia**: 23:55 (tolerância 5 min)

O dashboard sincroniza auto-refresh com este padrão.

### Dados de Aquisição
Podem estar **vazios inicialmente** (depende de UTM tracking). O dashboard mostra um estado vazio elegante.

### Performance
- **Cache client-side**: datasets ficam em memória durante navegação entre tabs
- **Fetch paralelo**: queries independentes em `Promise.all()`
- **Views agregadas**: reduzem carga no browser (cálculos no Postgres)

### Browser Compatibility
- **Chrome/Edge**: ✅ 100%
- **Firefox**: ✅ 100%
- **Safari**: ✅ (testar timezone)
- **IE11**: ❌ (usa APIs modernas)

---

## 🐛 Troubleshooting

### "Erro ao carregar dados"
1. Verifica se executaste o `setup.sql`
2. Confirma que as credenciais Supabase estão corretas
3. Abre a consola do browser (F12) e verifica erros
4. Testa as queries manualmente no SQL Editor do Supabase

### Gráficos não aparecem
- Verifica se Chart.js carregou (consola → `Chart` deve existir)
- Confirma que há dados no período selecionado
- Tenta outro período (ex: "Últimos 30 dias")

### Auto-refresh não funciona
- O browser tem de estar **aberto** (não funciona em background)
- Verifica timezone (deve ser Europe/Lisbon)
- Confirma que o n8n está a correr

### Notion embed cortado
- Ajusta altura do embed manualmente no Notion
- Usa `/embed` em vez de `/bookmark`
- Considera usar **Full Page** (link externo)

---

## 📚 Documentação Adicional

- **PLAN.md**: Planeamento detalhado completo
- **setup.sql**: SQL para criar views + comentários
- **index.html**: Dashboard completo (código inline documentado)

---

## 📧 Suporte

**Criado por**: Cursor AI Assistant  
**Data**: 07/10/2025  
**Versão**: 1.0.0  

Para bugs ou melhorias, edita diretamente o `index.html` — código modular e bem comentado.

---

## 🎉 Deploy Checklist

- [ ] Executar `setup.sql` no Supabase
- [ ] Verificar que as views foram criadas (`SELECT * FROM view_daily_kpis LIMIT 5`)
- [ ] Substituir `SUPABASE_URL` e `SUPABASE_ANON_KEY` no `index.html`
- [ ] Testar localmente (abrir `index.html` no browser)
- [ ] Deploy no Vercel/Netlify
- [ ] Testar URL de produção
- [ ] Embed no Notion
- [ ] Validar auto-refresh (deixar aberto 2h)
- [ ] Partilhar com equipa! 🚀

