<div align="center">
    <h1>Ambiente de Desenvolvimento para Ciência de Dados</h1>
    <p>Um ambiente local completo para ciência de dados com Docker Compose, integrando ferramentas para armazenamento, processamento, orquestração e análise.</p>
    <img src="https://img.shields.io/badge/Docker-0DB7ED?style=flat-square&logo=docker&logoColor=0DB7ED&labelColor=2E2E2E&color=0DB7ED" alt="Docker">
    <img src="https://img.shields.io/badge/PostgreSQL-0DB7ED?style=flat-square&logo=postgresql&logoColor=336791&labelColor=2E2E2E&color=0DB7ED" alt="PostgreSQL">
    <img src="https://img.shields.io/badge/Airflow-0DB7ED?style=flat-square&logo=apache-airflow&logoColor=007A87&labelColor=2E2E2E&color=0DB7ED" alt="Airflow">
    <img src="https://img.shields.io/badge/Spark-0DB7ED?style=flat-square&logo=apache-spark&logoColor=E25A1C&labelColor=2E2E2E&color=0DB7ED" alt="Spark">
    <img src="https://img.shields.io/badge/Kafka-0DB7ED?style=flat-square&logo=apache-kafka&logoColor=000000&labelColor=2E2E2E&color=0DB7ED" alt="Kafka">
    <img src="https://img.shields.io/badge/Jupyter-0DB7ED?style=flat-square&logo=jupyter&logoColor=F37626&labelColor=2E2E2E&color=0DB7ED" alt="Jupyter">
    <img src="https://img.shields.io/badge/MinIO-0DB7ED?style=flat-square&logo=minio&logoColor=C4292F&labelColor=2E2E2E&color=0DB7ED" alt="MinIO">
    <img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="MIT License">
    <img src="https://img.shields.io/badge/Version-1.0-blue?style=flat-square" alt="Version 1.0">
</div>

---

## 📖 Índice

- [Visão Geral](#visão-geral)
- [Propósito de Cada Aplicação](#propósito-de-cada-aplicação)
- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Instalação](#instalação)
- [Uso](#uso)
- [Estrutura de Diretórios](#estrutura-de-diretórios)
- [Solução de Problemas](#solução-de-problemas)
- [Contribuição](#contribuição)
- [Licença](#licença)
- [Documentação C4](#documentação-c4)

---

## 📖 Visão Geral

Este projeto configura um ambiente de desenvolvimento para ciência de dados utilizando **Docker Compose**, com ferramentas integradas para armazenamento, processamento, orquestração, mensagens e análise interativa. Todos os dados são persistidos em um volume unificado (`./data`) no host, com uma pasta compartilhada (`./data/shared`) e um volume Delta Lake (`./data/delta_lake`) para a arquitetura medalhão (Bronze, Silver, Gold). O armazenamento centralizado é gerenciado por **MinIO**, com conexões seguras via credenciais no arquivo `.env`.

---

## 🚀 Propósito de Cada Aplicação

- **PostgreSQL** 🗄️: Banco relacional para dados estruturados (e.g., resultados, metadados).
- **PgAdmin** 🖥️: Interface web para gerenciar bancos PostgreSQL.
- **Apache Airflow** ⏰: Orquestração de pipelines de dados (DAGs), integrado ao PostgreSQL.
- **Apache Spark** ⚡️: Computação distribuída para big data (Master, Worker, History Server).
- **Apache Kafka** 📨: Streaming de dados em tempo real.
- **Zookeeper** 🦒: Coordenação para Kafka.
- **Kafka UI** 📊: Monitoramento de tópicos e mensagens do Kafka.
- **Jupyter Notebook** 📓: Análise interativa com Python e R.
- **MinIO** ☁️: Armazenamento S3-compatível com conexões seguras para backup e compartilhamento.

---

## 🛠️ Arquitetura

### Componentes
- **Rede**: `ds-network` conecta todos os serviços.
- **Persistência**:
  - Volume unificado: `./data` (e.g., `./data/postgres`, `./data/airflow`).
  - Pasta compartilhada: `./data/shared` (acessível por Airflow, Spark, Jupyter).
  - Delta Lake: `./data/delta_lake/{bronze,silver,gold}` para armazenamento versionado.
  - MinIO: `./data/minio` para armazenamento S3, com credenciais seguras no `.env`.
- **Portas Expostas**:
  - PostgreSQL: `5432`
  - PgAdmin: `5050`
  - Airflow: `8080`
  - Spark Master: `7077`, `8081`
  - Spark History: `18080`
  - Kafka: `9092`
  - Zookeeper: `2181`
  - Kafka UI: `8082`
  - Jupyter: `8888`
  - MinIO: `9000` (API), `9001` (Console)

### Diagrama
```
[PostgreSQL:5432] <-> [PgAdmin:5050]
    |                [Airflow:8080] <-> [Shared Folder] <-> [Delta Lake]
[Kafka:9092] <-> [Zookeeper:2181] <-> [Kafka UI:8082]
[Spark Master:7077,8081] <-> [Spark Worker] <-> [Shared Folder] <-> [Delta Lake]
    |                        [Spark History:18080]
[Jupyter:8888] <-> [Shared Folder] <-> [Delta Lake]
[MinIO:9000,9001] <-> [All Services via S3 API]
```

---

## 📋 Pré-requisitos

- **Docker** 🐳: Versão 20.10 ou superior.
- **Docker Compose** 🛠️: Versão 1.29 ou superior.
- **Sistema Operacional**: Linux, macOS ou Windows (com WSL2).
- **Recursos**: 8GB RAM, 4 CPUs, 20GB disco.

---

## 🏗️ Instalação

1. **Clone o Repositório**
   ```bash
   git clone <url-do-repositório>
   cd <nome-do-repositório>
   ```

2. **Crie o Arquivo `.env`**
   Crie um arquivo `.env` com:
   ```plaintext
   POSTGRES_USER=postgres
   POSTGRES_PASSWORD=your_secure_password
   POSTGRES_DB=datascience
   PGADMIN_DEFAULT_EMAIL=admin@admin.com
   PGADMIN_DEFAULT_PASSWORD=admin
   AIRFLOW__CORE__EXECUTOR=LocalExecutor
   AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://postgres:your_secure_password@postgres:5432/datascience
   AIRFLOW__CORE__LOAD_EXAMPLES=false
   AIRFLOW_ADMIN_PASSWORD=admin
   JUPYTER_ENABLE_LAB=yes
   JUPYTER_TOKEN=datascience
   MINIO_ROOT_USER=minioadmin
   MINIO_ROOT_PASSWORD=minioadmin123
   ```
   Substitua `your_secure_password` por uma senha segura.

3. **Inicie os Containers**
   ```bash
   docker-compose up -d
   ```

4. **Inicialize o Delta Lake**
   ```bash
   mkdir -p ./data/delta_lake/{bronze,silver,gold}
   ```

5. **Verifique o Status**
   ```bash
   docker-compose ps
   ```

---

## 🎮 Uso

### Acessando os Serviços
- **PostgreSQL** 🗄️: `localhost:5432` (use credenciais do `.env`).
- **PgAdmin** 🖥️: `http://localhost:5050` (`PGADMIN_DEFAULT_EMAIL`, `PGADMIN_DEFAULT_PASSWORD`).
- **Airflow** ⏰: `http://localhost:8080` (usuário: `admin`, senha: `AIRFLOW_ADMIN_PASSWORD`).
- **Spark Master** ⚡️: `http://localhost:8081`.
- **Spark History** 📜: `http://localhost:18080`.
- **Kafka** 📨: `localhost:9092`.
- **Kafka UI** 📊: `http://localhost:8082`.
- **Jupyter** 📓: `http://localhost:8888` (token: `JUPYTER_TOKEN`).
- **MinIO** ☁️: API em `localhost:9000`, console em `http://localhost:9001` (`MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`).

### Exemplos
- **Airflow DAG**:
  Crie em `./data/airflow/dags/example_dag.py`:
  ```python
  from airflow import DAG
  from airflow.operators.python import PythonOperator
  from datetime import datetime

  def print_hello():
      print("Hello, Airflow!")

  with DAG('hello_world', start_date=datetime(2025, 1, 1), schedule_interval='@daily') as dag:
      task = PythonOperator(task_id='print_hello', python_callable=print_hello)
  ```

- **Spark com Delta Lake**:
  ```bash
  docker-compose exec spark-master bash
  ```
  ```python
  from pyspark.sql import SparkSession
  spark = SparkSession.builder.appName("DeltaLake").getOrCreate()
  df = spark.read.csv("s3a://mybucket/data.csv")
  df.write.format("delta").mode("overwrite").save("/opt/bitnami/spark/delta_lake/bronze/table")
  ```

- **MinIO com Python**:
  ```python
  from minio import Minio
  client = Minio("minio:9000", access_key="minioadmin", secret_key="minioadmin123", secure=False)
  client.make_bucket("mybucket")
  ```

- **Backup**:
  ```bash
  tar -czf backup_data.tar.gz ./data
  ```

---

## 📂 Estrutura de Diretórios

- `./data/`:
  - `postgres/`: Dados PostgreSQL.
  - `pgadmin/`: Configurações PgAdmin.
  - `airflow/{dags,logs,plugins}/`: Airflow.
  - `spark/{data,logs}/`: Spark.
  - `kafka/`: Kafka.
  - `zookeeper/`: Zookeeper.
  - `notebooks/`: Jupyter notebooks.
  - `scripts/`: Jupyter scripts.
  - `shared/`: Pasta compartilhada.
  - `delta_lake/{bronze,silver,gold}/`: Delta Lake.
  - `minio/`: Armazenamento MinIO.

---

## 🛠️ Solução de Problemas

- **Container não inicia**: Verifique logs (`docker-compose logs <serviço>`), portas (`netstat -tuln`), permissões (`sudo chown -R $USER:$USER ./data`).
- **Airflow não conecta ao PostgreSQL**: Confirme `AIRFLOW__DATABASE__SQL_ALCHEMY_CONN`.
- **MinIO não acessível**: Verifique credenciais no `.env` e portas `9000`/`9001`.
- **Delta Lake erros**: Confirme `delta-spark` nos containers Spark/Jupyter.

---

## 🤝 Contribuição

1. Fork o repositório.
2. Crie uma branch: `git checkout -b feature/nova-funcionalidade`.
3. Commit: `git commit -m "Descrição"`.
4. Push: `git push origin feature/nova-funcionalidade`.
5. Abra um pull request.

---

## 📜 Licença

Licença MIT.

---

## 📚 Documentação C4

Consulte a [Documentação C4](c4_documentation.md) para uma visão detalhada da arquitetura no modelo C4 (Contexto, Contêiner, Componente).

---

<div align="center">
    <h3>Desenvolvido por Mauricio A. Almeida</h3>
    <a href="https://github.com/mauricioaalmeida"><img src="https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=FFFFFF" alt="GitHub"></a>
    <a href="https://linkedin.com/in/mauricioaalmeida"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=FFFFFF" alt="LinkedIn"></a>
</div>