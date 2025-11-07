Desenvolvi um pipeline local de contingência de marketing (sem VPS), orquestrado com n8n + Docker. Implementei aquisição de chip via API, aquecimento automático e persistência em MySQL, incluindo visualização em dashboard. Usei Firecrawl self-hosted para análise de documentação e referências técnicas. Resultado: custo quase zero, controle total e reprodutibilidade.
# Pipeline de Contingência de Marketing (Local, Docker, n8n)

> **Objetivo:** reduzir custo e aumentar controle rodando tudo **localmente**: aquisição de chip via API para contingência, **aquecimento automático**, persistência em **MySQL** e visualização em **dashboard**.  
> **Stack:** Docker + n8n + MySQL + Adminer + Firecrawl (self-hosted) + Python para utilidades.

---

##  Visão Geral

- **Zero VPS**: executo toda a orquestração local via **Docker** para eliminar custos.
- **n8n**: workflows para:
  - chamar **API de aquisição de chip** (contingência),
  - **aquecimento automático** (gatilhos/esperas),
  - gravação no **MySQL**,
  - envio de eventos para dashboard.
- **Banco**: **MySQL** (tabelas simplificadas de chips, warm-ups e logs).
- **Firecrawl** (self-hosted): estudo de documentação técnica + coleta de referências para automação (versão open source com recursos suficientes para o meu caso).
- **Provas**: screenshots do **n8n** e do **Docker Desktop** incluídas em `/docs`.

<p align="center">
  <img src="docs/n8n-workflow.png" width="820" />
</p>

---

## Arquitetura

```mermaid
flowchart LR
  A[Webhook / Disparo manual] --> B[n8n]
  B -->|HTTP Request| C[API Chip Provider]
  B -->|Wait/Retry| B
  B -->|Insert/Update| D[(MySQL)]
  B -->|Emit Event| E[Dashboard (ex.: Metabase/Grafana)]
  F[Firecrawl (self-hosted)] --> G[(Postgres interno do Firecrawl)]
  F --> H[Docs & Notas técnicas]
