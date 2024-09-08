## Project 3 Batch Processing Using Airflow and Spark

### Use case:
### From product need help to integrate data from our dwh to their product via API:

- Top Country Based on User
- Total Film Based on Category

### Prepare Tools:

- Airflow on Local
- VSCode
- Dbeaver
- Register postman

### Dataset:

[Sample Data](https://www.kaggle.com/datasets/kapturovalexander/pagila-postgresql-sample-database)


### Flow

![Deskripsi Gambar](https://imminent-locust-045.notion.site/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F01b24fa3-f906-4bbc-ae9a-eae8c32be7d8%2F32c7269a-4849-4425-bf0d-a7c1fb667885%2Fimage.png?table=block&id=0fc17910-9ab2-4e12-9d21-b0bd32002c60&spaceId=01b24fa3-f906-4bbc-ae9a-eae8c32be7d8&width=2000&userId=&cache=v2)


Noted:

- What is TiDB?
TiDB is an open-source NewSQL database that supports Hybrid Transactional and Analytical Processing workloads.

Step by Step:

- Check connection DB server
    - Postgres
    - TiDB
- Run airflow on your local
    - Create file requirements.txt:

        - pyspark==3.5.2
        - psycopg2-binary==2.9.9
        - mysql-connector-python==9.0.0
        - pandas

    - Build images, Dockerfile:

            FROM apache/airflow:2.10.0
            COPY requirements.txt /
            RUN pip install --no-cache-dir "apache-airflow==${AIRFLOW_VERSION}" -r /requirements.txt

    - docker build -t my-airflow .
    - Create docker compose, docker-compose.yaml:

            version: '3.4'
            services:
                postgres:
                    image: postgres:13
                    container_name: postgres
                    ports:
                    - "5434:5432"
                    healthcheck:
                    test: ["CMD", "pg_isready", "-U", "airflow"]
                    interval: 5s
                    retries: 5
                    env_file:
                    - .env
                    volumes:
                    - postgres_airflow:/var/lib/postgresql/data

                scheduler:
                    image: my-airflow
                    user: "${AIRFLOW_UID}:0"
                    env_file: 
                    - .env
                    volumes:
                    - ./dags:/opt/airflow/dags
                    - ./logs:/opt/airflow/logs
                    - ./plugins:/opt/airflow/plugins
                    - /var/run/docker.sock:/var/run/docker.sock
                    depends_on:
                    postgres:
                        condition: service_healthy
                    airflow-init:
                        condition: service_completed_successfully
                    container_name: airflow-scheduler
                    command: scheduler
                    restart: on-failure
                    ports:
                    - "8793:8793"

                webserver:
                    image: my-airflow
                    user: "${AIRFLOW_UID}:0"
                    env_file: 
                    - .env
                    volumes:
                    - ./dags:/opt/airflow/dags
                    - ./logs:/opt/airflow/logs
                    - ./plugins:/opt/airflow/plugins
                    - /var/run/docker.sock:/var/run/docker.sock
                    depends_on:
                    postgres:
                        condition: service_healthy
                    airflow-init:
                        condition: service_completed_successfully
                    container_name: airflow-webserver
                    restart: always
                    command: webserver
                    ports:
                    - "8080:8080"
                    healthcheck:
                    test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
                    interval: 30s
                    timeout: 30s
                    retries: 5
                
                airflow-init:
                    image: my-airflow
                    user: "${AIRFLOW_UID}:0"
                    env_file: 
                    - .env
                    volumes:
                    - ./dags:/opt/airflow/dags
                    - ./logs:/opt/airflow/logs
                    - ./plugins:/opt/airflow/plugins
                    - /var/run/docker.sock:/var/run/docker.sock
                    container_name: airflow-init
                    entrypoint: /bin/bash
                    command:
                    - -c
                    - |
                        mkdir -p /sources/logs /sources/dags /sources/plugins
                        chown -R "${AIRFLOW_UID}:0" /sources/{logs,dags,plugins}
                        exec /entrypoint airflow version
                volumes:
                postgres_airflow:
                    external: true
    - Set connection on airflow
        - Postgres
        - TiDB
            - [Documentation](https://docs.pingcap.com/tidbcloud/secure-connections-to-serverless-clusters)
    - Extract:
        - Create module connector postgres
        - Create module get data from postgres
    - Transform:
        - Create script for transformation data using spark
    - Load
        - Create module connector 
        - Create load data
