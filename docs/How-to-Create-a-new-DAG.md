1. 먼저 본인 EC2 Airflow에 로그인합니다
2. sudo su airflow 명령을 실행해서 airflow 사용자로 바꿉니다
3. 그러면 /var/lib/airflow/ 디렉토리로 이동되는데 cd dags를 실행해서 dags 폴더로 이동합니다
```
airflow@ip-172-31-xx-xxx:~$ cd dags
airflow@ip-172-31-xx-xxx:~/dags$ pwd
/var/lib/airflow/dags
```

4. 숙제를 위한 DAG 작성을 위해 파이썬 파일을 하나 vi 에디터로 오픈합니다. 다음과 같은 내용으로 파일을 일단 만듭니다
```
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.models import Variable
from airflow.hooks.postgres_hook import PostgresHook

from datetime import datetime
from datetime import timedelta

import requests
import logging
import psycopg2


def get_Redshift_connection(autocommit=False):
    hook = PostgresHook(postgres_conn_id='redshift_dev_db')
    conn = hook.get_conn()
    conn.autocommit = autocommit
    return conn.cursor()


def run(**context):
    # 여기에 내용 채우기
    pass


dag = DAG(
    dag_id = 'MyDAG_ID',
    start_date = datetime(2022,8,18), # 적당히 조절
    schedule_interval = '0 2 * * *',  # 적당히 조절
    max_active_runs = 1,
    catchup = False,
    default_args = {
        'retries': 1,
        'retry_delay': timedelta(minutes=3),
    })

task1 = PythonOperator(
    task_id = 'task1',
    python_callable = run,
    dag = dag)
```

5. 위와 같은 세팅의 DAG(ID가 MyDAG_ID)에서 task(ID가 task1)를 실행하려면 아래와 같이 실행합니다
```
airflow tasks test MyDAG_ID task1 YYYY-MM-DD
```
