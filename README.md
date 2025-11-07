Sistema de Contingência de Marketing – Aquisição de Chips via API (Local, Docker, n8n, sms-activate)

Objetivo: reduzir custo e aumentar o controle operacional rodando 100% local: aquisição de chip via API sms-activate, aquecimento automático, persistência em MySQL e visualização em dashboard.
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

 Banco de Dados (schema resumido)
-- sql/schema.sql
CREATE TABLE IF NOT EXISTS chips (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  provider_id VARCHAR(64) NOT NULL,
  msisdn VARCHAR(32) UNIQUE NOT NULL,
  status ENUM('new','warming','ready','failed') DEFAULT 'new',
  meta JSON NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS warmups (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  chip_id BIGINT NOT NULL,
  step VARCHAR(64) NOT NULL,
  status ENUM('queued','running','done','error') DEFAULT 'queued',
  detail TEXT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (chip_id) REFERENCES chips(id)
);

CREATE TABLE IF NOT EXISTS logs (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  level ENUM('INFO','WARN','ERROR') NOT NULL,
  source VARCHAR(64) NOT NULL,
  message TEXT NOT NULL,
  payload JSON NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

Fluxo do n8n (resumo)

Trigger (Webhook ou manual).

HTTP Request → API de aquisição de chip.

IF/validações + Wait/retries → aquecimento automático.

Execute Query (MySQL) → insere/atualiza chips e warmups.

Emite eventos/metadados para dashboard (em processo de construção)

Logs de cada etapa gravados em logs.

Exporte seu workflow e salve em workflows/aquisicao_chip_e_aquecimento.json.

Subindo localmente (Docker)
1) .env

Crie a partir do exemplo:

# .env.example
N8N_ENCRYPTION_KEY=troque-por-uma-chave-segura

MYSQL_ROOT_PASSWORD=troque-root
MYSQL_DATABASE=contingencia
MYSQL_USER=contingencia_user
MYSQL_PASSWORD=troque-user

# API do provider (não commitar credenciais reais)
CHIP_API_BASE=https://api.seu-provedor.com
CHIP_API_KEY=sua-chave

2) docker-compose.yml 
version: "3.8"
services:
  n8n:
    image: n8nio/n8n:latest
    ports: ["5678:5678"]
    env_file: [.env]
    environment:
      - DB_TYPE=mysqldb
      - DB_MYSQLDB_HOST=mysql
      - DB_MYSQLDB_PORT=3306
      - DB_MYSQLDB_DATABASE=${MYSQL_DATABASE}
      - DB_MYSQLDB_USER=${MYSQL_USER}
      - DB_MYSQLDB_PASSWORD=${MYSQL_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    depends_on: [mysql]
    volumes: [ "n8n_data:/home/node/.n8n" ]

  mysql:
    image: mysql:8
    command: --default-authentication-plugin=mysql_native_password
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    ports: ["3306:3306"]
    volumes:
      - mysql_data:/var/lib/mysql
      - ./sql/schema.sql:/docker-entrypoint-initdb.d/01-schema.sql

  adminer:
    image: adminer:latest
    ports: ["8080:8080"]
    depends_on: [mysql]

  firecrawl:
    image: mendableai/firecrawl:latest
    ports: ["3002:3002"]
    environment: [ "NODE_ENV=production" ]

volumes:
  n8n_data:
  mysql_data:

3) Subir
docker compose up -d


n8n: http://localhost:5678

Adminer: http://localhost:8080 (Server: mysql)

Firecrawl: http://localhost:3002

Populando dados 
# utils/seed.py
import os, json
import mysql.connector as mc

conn = mc.connect(
  host=os.getenv("MYSQL_HOST","localhost"),
  user=os.getenv("MYSQL_USER","contingencia_user"),
  password=os.getenv("MYSQL_PASSWORD","troque-user"),
  database=os.getenv("MYSQL_DATABASE","contingencia"),
  port=int(os.getenv("MYSQL_PORT","3306"))
)
cur = conn.cursor()
cur.execute("INSERT INTO logs(level,source,message,payload) VALUES(%s,%s,%s,%s)",
            ("INFO","seed","warmup step", json.dumps({"step": 1})))
conn.commit()
print("ok")

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


Autoria
Marina de Alcantara Andrade
Automação local com n8n + Docker, integração via APIs, persistência em MySQL e análise técnica com Firecrawl self-hosted.
