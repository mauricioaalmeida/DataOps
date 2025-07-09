# Exemplo de Pipeline de Análise de Dados de Ações

Este documento detalha a criação de um pipeline de dados de ponta a ponta, utilizando as ferramentas disponíveis no ambiente de desenvolvimento. O objetivo é capturar dados históricos de ações, criar um mecanismo de atualização em tempo real e, por fim, analisar e visualizar esses dados.

## Visão Geral do Pipeline

1.  **Carga Inicial (Batch)**: Um DAG do Airflow buscará 5 anos de dados históricos das ações `BBSA3.SA`, `PETR4.SA`, e `VALE3.SA` usando a API `yfinance` e os salvará em uma tabela no PostgreSQL.
2.  **Atualização (Streaming)**: Um script Python (produtor) consultará a cotação atual das ações e a publicará em um tópico Kafka. Um segundo DAG do Airflow (consumidor) lerá essas mensagens e atualizará os dados no PostgreSQL.
3.  **Análise e Visualização**: Um Jupyter Notebook usará o Spark para se conectar ao PostgreSQL, carregar os dados e gerar análises e gráficos sobre o desempenho das ações.

## Ferramentas Utilizadas

- **Orquestração**: Apache Airflow
- **Mensageria**: Apache Kafka
- **Banco de Dados**: PostgreSQL
- **Processamento**: Apache Spark
- **Análise Interativa**: Jupyter Notebook
- **Monitoramento**: Kafka UI e PgAdmin

---

## Passo 1: Preparação do Ambiente

Antes de criar o pipeline, precisamos instalar as bibliotecas Python necessárias nos contêineres do Airflow e do Jupyter e configurar a conexão do Airflow com o banco.

### 1.1. Dependências Python

As bibliotecas Python necessárias para este exemplo (`pandas`, `yfinance`, `kafka-python`, etc.) são gerenciadas de forma declarativa. Elas estão listadas nos arquivos `docker/airflow/requirements.txt` e `docker/jupyter/requirements.txt`.

Quando você executa `docker-compose up --build`, o Docker constrói imagens personalizadas para os serviços do Airflow e do Jupyter, instalando automaticamente todas as dependências. Isso garante um ambiente consistente e reprodutível, eliminando a necessidade de instalação manual de pacotes.

### 1.2. Configurando a Conexão no Airflow

1.  Acesse a UI do Airflow em `http://localhost:8080`.
2.  Vá para **Admin -> Connections**.
3.  Clique em **+** para adicionar uma nova conexão.
4.  Preencha o formulário com os seguintes dados (se `postgres_default` já não estiver configurado corretamente):
    *   **Connection Id**: `postgres_dataops`
    *   **Connection Type**: `Postgres`
    *   **Host**: `postgres`
    *   **Schema**: `datascience` (o DB criado no `.env`)
    *   **Login**: `postgres` (o usuário do `.env`)
    *   **Password**: A senha que você definiu em `POSTGRES_PASSWORD` no seu arquivo `.env`.
    *   **Port**: `5432`
5.  Clique em **Test**. Se a conexão for bem-sucedida, clique em **Save**. Usaremos o Conn ID `postgres_dataops` nos DAGs.

---

## Passo 2: Carga Inicial de Dados Históricos (Airflow)

Criaremos um DAG para buscar os dados históricos e popular nossa tabela no PostgreSQL.

### 2.1. Código do DAG de Carga Inicial

Crie o arquivo `dag_initial_load.py` dentro do diretório `./data/airflow/dags/` com o seguinte conteúdo:

```python
import yfinance as yf
import pandas as pd
from airflow.decorators import dag, task
from airflow.providers.postgres.hooks.postgres import PostgresHook
from datetime import datetime, timedelta
import json

@dag(
    dag_id='stock_initial_load',
    start_date=datetime(2023, 1, 1),
    schedule_interval=None,
    catchup=False,
    tags=['data-pipeline', 'stocks'],
)
def stock_initial_load_dag():

    @task
    def get_historical_data():
        tickers = ["BBSA3.SA", "PETR4.SA", "VALE3.SA"]
        end_date = datetime.now()
        start_date = end_date - timedelta(days=5*365)
        
        all_data = []
        for ticker in tickers:
            df = yf.download(ticker, start=start_date, end=end_date)
            df.reset_index(inplace=True)
            df['ticker'] = ticker
            all_data.append(df)
            
        combined_df = pd.concat(all_data)
        return json.loads(combined_df.to_json(orient='records', date_format='iso'))

    @task
    def create_table_and_load_data(data):
        hook = PostgresHook(postgres_conn_id='postgres_dataops')
        conn = hook.get_conn()
        cursor = conn.cursor()

        create_table_sql = """
        CREATE TABLE IF NOT EXISTS stock_data (
            "Date" TIMESTAMP,
            "Open" REAL,
            "High" REAL,
            "Low" REAL,
            "Close" REAL,
            "Adj Close" REAL,
            "Volume" BIGINT,
            "ticker" VARCHAR(10),
            PRIMARY KEY ("Date", "ticker")
        );
        """
        cursor.execute(create_table_sql)

        insert_sql = """
        INSERT INTO stock_data ("Date", "Open", "High", "Low", "Close", "Adj Close", "Volume", "ticker")
        VALUES (%(Date)s, %(Open)s, %(High)s, %(Low)s, %(Close)s, %(Adj Close)s, %(Volume)s, %(ticker)s)
        ON CONFLICT ("Date", "ticker") DO NOTHING;
        """
        cursor.executemany(insert_sql, data)
        
        conn.commit()
        cursor.close()
        conn.close()

    historical_data = get_historical_data()
    create_table_and_load_data(historical_data)

stock_initial_load_dag()
```

### 2.2. Entendendo o DAG

- **`get_historical_data`**: Esta tarefa usa `yfinance` para baixar os dados dos últimos 5 anos. Ela combina os dados de todos os tickers em um único DataFrame Pandas e o converte para JSON para ser passado para a próxima tarefa via XComs.
- **`create_table_and_load_data`**: Esta tarefa se conecta ao PostgreSQL usando o `PostgresHook`. Primeiro, ela garante que a tabela `stock_data` exista. Em seguida, ela insere os registros recebidos. A lógica `ON CONFLICT DO NOTHING` evita a duplicação de dados.

Após salvar o arquivo, o DAG `stock_initial_load` aparecerá na UI do Airflow. Ative-o e dispare uma execução manual para popular o banco de dados.

### 2.3. Verificando os Dados no PgAdmin

1.  Acesse o PgAdmin em `http://localhost:5050`.
2.  Faça login e adicione um novo servidor se necessário (hostname `postgres`).
3.  Navegue até `datascience -> Schemas -> public -> Tables`. Você deverá ver a tabela `stock_data`.
4.  Clique com o botão direito e selecione `View/Edit Data` para ver os registros.

---

## Passo 3: Pipeline de Atualização com Kafka

Agora, vamos simular atualizações em tempo real. Um script produtor enviará novas cotações para o Kafka, e um DAG consumidor as processará.

### 3.1. Script do Produtor Kafka

Crie o arquivo `stock_producer.py` no diretório `./data/scripts/`:

```python
import yfinance as yf
from kafka import KafkaProducer
import json
import time

producer = KafkaProducer(
    bootstrap_servers='kafka:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

tickers = ["BBSA3.SA", "PETR4.SA", "VALE3.SA"]

print("Iniciando produtor de cotações...")
while True:
    for ticker in tickers:
        try:
            data = yf.Ticker(ticker).history(period="1d")
            if not data.empty:
                latest_data = data.iloc[-1].to_dict()
                latest_data['ticker'] = ticker
                latest_data['Date'] = data.index[-1].isoformat()
                print(f"Enviando: {latest_data}")
                producer.send('stock_updates', value=latest_data)
        except Exception as e:
            print(f"Erro ao buscar dados para {ticker}: {e}")
    time.sleep(15) # Espera 15 segundos antes da próxima rodada
```

Para executar o produtor, abra um novo terminal e execute:
```bash
docker-compose exec jupyter python /home/jovyan/scripts/stock_producer.py
```
Este script ficará em execução, enviando uma nova cotação a cada 15 segundos para o tópico `stock_updates`.

### 3.2. Monitorando o Tópico no Kafka UI

Acesse a Kafka UI em `http://localhost:8082`. Você verá o tópico `stock_updates`. Clique nele e vá para a aba `Messages` para ver os dados chegando.

### 3.3. DAG Consumidor Kafka

Crie o arquivo `dag_stream_update.py` em `./data/airflow/dags/`.

```python
from airflow.decorators import dag, task
from airflow.providers.postgres.hooks.postgres import PostgresHook
from datetime import datetime
from kafka import KafkaConsumer
import json

@dag(
    dag_id='stock_stream_update',
    start_date=datetime(2023, 1, 1),
    schedule_interval='* * * * *', # Executa a cada minuto
    catchup=False,
    max_active_runs=1,
    tags=['data-pipeline', 'stocks', 'streaming'],
)
def stock_stream_update_dag():

    @task
    def consume_and_update_stock_data():
        consumer = KafkaConsumer(
            'stock_updates',
            bootstrap_servers='kafka:9092',
            auto_offset_reset='earliest',
            group_id='airflow_stock_consumer',
            consumer_timeout_ms=5000, # Para o consumidor após 5s se não houver mensagens
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )

        messages = list(consumer)
        if not messages:
            print("Nenhuma mensagem nova no tópico.")
            return

        hook = PostgresHook(postgres_conn_id='postgres_dataops')
        conn = hook.get_conn()
        cursor = conn.cursor()

        # Lógica de UPSERT (Update or Insert)
        upsert_sql = """
        INSERT INTO stock_data ("Date", "Open", "High", "Low", "Close", "Adj Close", "Volume", "ticker")
        VALUES (%(Date)s, %(Open)s, %(High)s, %(Low)s, %(Close)s, %(Adj Close)s, %(Volume)s, %(ticker)s)
        ON CONFLICT ("Date", "ticker") DO UPDATE SET
            "Open" = EXCLUDED."Open",
            "High" = EXCLUDED."High",
            "Low" = EXCLUDED."Low",
            "Close" = EXCLUDED."Close",
            "Adj Close" = EXCLUDED."Adj Close",
            "Volume" = EXCLUDED."Volume";
        """
        
        for msg in messages:
            cursor.execute(upsert_sql, msg.value)

        conn.commit()
        cursor.close()
        conn.close()
        print(f"{len(messages)} mensagens processadas.")

    consume_and_update_stock_data()

stock_stream_update_dag()
```

Ative o DAG `stock_stream_update` na UI do Airflow. Você verá as execuções acontecendo a cada minuto, processando os dados que o produtor está enviando.

---

## Passo 4: Análise de Dados com Jupyter e Spark

Finalmente, vamos usar o Jupyter e o Spark para analisar os dados consolidados no PostgreSQL.

### 4.1. Criando o Notebook de Análise

1.  Acesse o Jupyter em `http://localhost:8888` (token: `datascience`).
2.  Crie um novo notebook Python 3 no diretório `work`.
3.  Adicione as seguintes células de código ao seu notebook:

**Célula 1: Importações e Configuração da Sessão Spark**
```python
from pyspark.sql import SparkSession

# O Spark precisa do driver JDBC do PostgreSQL.
# Vamos configurar a sessão para baixar o pacote necessário.
spark = SparkSession.builder \
    .appName("StockAnalysis") \
    .config("spark.driver.extraClassPath", "/home/jovyan/work/postgresql-42.3.1.jar") \
    .config("spark.jars.packages", "org.postgresql:postgresql:42.3.1") \
    .getOrCreate()

print("Sessão Spark criada com sucesso!")
```

**Célula 2: Lendo os Dados do PostgreSQL com Spark**
```python
db_properties = {
    "user": "postgres",
    "password": "your_secure_password", # SUBSTITUA PELA SUA SENHA DO .env
    "driver": "org.postgresql.Driver"
}

jdbc_url = "jdbc:postgresql://postgres:5432/datascience"

df_spark = spark.read.jdbc(url=jdbc_url, table="stock_data", properties=db_properties)

df_spark.printSchema()
df_spark.show(5)
```

**Célula 3: Análise e Visualização**
```python
import matplotlib.pyplot as plt
import seaborn as sns

# Para visualização, é mais fácil trabalhar com Pandas.
# Vamos converter o DataFrame Spark para Pandas.
df_pandas = df_spark.toPandas()

# Converter a coluna de data para o formato datetime do pandas
df_pandas['Date'] = pd.to_datetime(df_pandas['Date'])

# Configurar o gráfico
plt.style.use('seaborn-v0_8-whitegrid')
fig, ax = plt.subplots(figsize=(15, 7))

# Plotar o preço de fechamento para cada ação
sns.lineplot(data=df_pandas, x='Date', y='Close', hue='ticker', ax=ax)

ax.set_title('Evolução do Preço de Fechamento (Últimos 5 Anos)', fontsize=16)
ax.set_xlabel('Data', fontsize=12)
ax.set_ylabel('Preço de Fechamento (R$)', fontsize=12)
ax.legend(title='Ação')

plt.show()
```

### 4.2. Entendendo o Notebook

- **Sessão Spark**: Configuramos a sessão para incluir o pacote do driver JDBC do PostgreSQL, permitindo que o Spark se comunique com o banco.
- **Leitura de Dados**: Usamos `spark.read.jdbc` para carregar a tabela `stock_data` em um DataFrame Spark.
- **Análise e Visualização**: Convertemos o DataFrame Spark para um DataFrame Pandas (`.toPandas()`) para facilitar a plotagem com `matplotlib` e `seaborn`, gerando um gráfico de linhas que compara a evolução do preço de fechamento das três ações.

---

## Conclusão

Parabéns! Você construiu com sucesso um pipeline de dados completo e funcional, que combina processos de batch e streaming para análise de dados financeiros.

Este exemplo prático demonstrou como as diferentes ferramentas do nosso ambiente de desenvolvimento se integram para criar uma solução robusta:

- **Airflow** orquestrou tanto a carga histórica quanto as atualizações contínuas de forma confiável.
- **Kafka** atuou como um buffer de alta performance para os dados em tempo real, desacoplando o produtor do consumidor.
- **PostgreSQL** serviu como um repositório de dados estruturados, acessível e eficiente.
- **Spark** forneceu o poder de processamento para analisar um volume de dados que poderia ser muito maior, diretamente do banco de dados.
- **Jupyter** ofereceu o ambiente interativo ideal para explorar os dados e visualizar os resultados da análise.

Este projeto serve como uma base sólida. A partir daqui, você pode explorar pipelines mais complexos, adicionar etapas de transformação de dados (por exemplo, salvando dados agregados no Delta Lake), desenvolver modelos de machine learning ou integrar novas fontes de dados.