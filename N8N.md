Nome sugerido: Analytics Core – Today (Provisório)

Objetivo: recolher métricas detalhadas por página, de hora em hora, para acompanhar o tráfego diário.

GA4 Request:

dateRange: today

Métricas:

1-day active users

newUsers

sessions

averageSessionDuration

engagementRate

bounceRate

Dimensão:

pagePath

Output guardado:

Campo	Descrição
date	Data
page_path	Caminho da página
sessions	Sessões totais
unique_users	Utilizadores únicos ativos (1-day)
new_users	Novos utilizadores
engagement_rate	Taxa de engagement (0–1)
avg_session_duration	Duração média (segundos)
bounce_rate	Taxa de rejeição (0–1)
⚙️ Comportamento

Corre de hora em hora → atualiza valores do próprio dia.

Faz upsert (sem duplicar).

Pode haver pequenas variações (aumenta ao longo do dia até estabilizar).

☀️ Fluxo diário

Cria outro igual, mas:

dateRange: yesterday

Corre às 14:00 (Europe/Lisbon)

Este grava os valores finais “oficiais” do dia anterior.

PODE SE FAZER
📈 O QUE PODES FAZER COM ISTO

1️⃣ Top páginas — ranking das mais visitadas.
2️⃣ Gráficos de sessões/visitas por página.
3️⃣ Comparação diária (ontem vs hoje) para cada página.
4️⃣ Cálculo do engagement real e tempo médio em cada secção do site.
5️⃣ Evolução temporal por página (quando acumulares vários dias).