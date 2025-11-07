Sistema de Contingência de Marketing – Aquisição de Chips via API (Local, Docker, n8n, sms-activate)
Objetivo: reduzir custo em édia de 7,00 para 0,60 centavos e aumentar o controle operacional rodando 100% local: aquisição de chip via API sms-activate, aquecimento automático, persistência em MySQL e visualização em dashboard.
Stack: Docker + n8n + MySQL + (Adminer/Metabase/Grafana) + Firecrawl self-hosted + scripts em Python.



Visão Geral
Zero VPS: execução local em Docker para eliminar custos recorrentes.
n8n: orquestra o fluxo (aquisição → validação → aquecimento → persistência → eventos para dashboard).
Banco SQL: guarda chips, passos de aquecimento e logs.
Firecrawl local: leitura/parse de documentação e páginas para apoiar automações (recursos suficientes da versão open-source).
Escalável: se no futuro precisar, basta apontar o docker-compose para um servidor/VPS.


Arquitetura (alto nível)
flowchart LR
  TRG[Webhook/Trigger] --> N8N[n8n]
  N8N -->|HTTP Request| API[(API Chip Provider)]
  N8N -->|Insert/Update| DB[(MySQL)]
  N8N -->|Emit Event| DASH[Dashboard (Metabase/Grafana)]
  FCR[Firecrawl (self-hosted)] --> DOCS[Notas Técnicas/Referências]


Tecnologias
Docker / Docker Compose
n8n (automação/orquestração)
MySQL 8 (+ Adminer opcional para UI)
Metabase ou Grafana  para dashboards
Firecrawl self-hosted (coleta de docs/repos)
Python (scripts utilitários)



Estrutura do repositório
.
├─ docker-compose.yml
├─ .env.example
├─ sql/
│  └─ schema.sql
├─ workflows/
│  └─ aquisicao_chip_e_aquecimento.json   # export do n8n
├─ utils/
│  └─ seed.py
├─ docs/
│  ├─ n8n-workflow.png
│  └─ docker-desktop.png
└─ README.md

Subir
docker compose up -d
n8n: http://localhost:5678
Adminer: http://localhost:8080 (Server: mysql)
Firecrawl: http://localhost:3002

Evidências (docs)
docs/n8n-workflow.png – visão do fluxo
<img width="1352" height="672" alt="image" src="https://github.com/user-attachments/assets/5d8fa6e9-2ece-4141-8eea-7dd9f92a725f" />

docs/docker-desktop.png – containers em execução
<img width="1365" height="698" alt="image" src="https://github.com/user-attachments/assets/c17f869c-d483-47e2-937d-e91fb0f2ad5f" />


Por que local (sem VPS)?
Sem mensalidades desnecessárias.
Controle total sobre dados e latência.
Subirei para VPS apenas se a operação exigir multiusuário/24×7.


Segurança
Nunca comitar chaves/segredos (.env no .gitignore).
Limitar portas expostas se publicar em rede.
Rotacionar N8N_ENCRYPTION_KEY se necessário.



Marina de Alcantara Andrade
Automação local com n8n + Docker, integração via APIs, persistência em MySQL e análise técnica com Firecrawl self-hosted.
