# Airflow 2.9.1 Installation:

- 우분투 20.04에서 Airflow 2.9.1을 설치하는 방법에 대한 문서로 파이썬 3.8을 사용
- 앞서 별도로 공유된 ssh 로그인 문서를 참조하여 할당된 EC2 서버로 로그인 (이 때 ubuntu 계정을 사용함).
- VS Code에서 바로 접근하는 방법도 있으며 이 역시 별도 문서를 제공할 예정

## Airflow Python Module Installation

#### 먼저 우분투의 소프트웨어 관리 툴인 apt-get을 업데이트하고 파이썬 3.0 pip을 설치한다.

원래는 apt-get update 이후에 python3-pip을 설치하면 되는데 pyopenssl 관련 충돌이 있어서 이를 먼저 해결하고 python3-pip를 설치

```
sudo apt-get update 
wget https://bootstrap.pypa.io/get-pip.py
sudo python3 get-pip.py
sudo apt-get install -y python3-pip
sudo pip3 install pyopenssl --upgrade
```

#### 다음으로 Airflow 2.0을 설치하고 필요 기타 모듈을 설치한다

```
sudo apt-get install -y libmysqlclient-dev
sudo pip3 install --ignore-installed "apache-airflow[celery,amazon,mysql,postgres]==2.9.1" --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.9.1/constraints-3.8.txt"
sudo pip3 install oauth2client gspread numpy pandas
```

## airflow:airflow 계정 생성

우리가 설치할 airflow 서비스는 ubuntu 사용자가 아닌 airflow 사용자를 기반으로 실행될 예정이며 이를 위해 airflow 계정을 생성한다. airflow 계정의 홈디렉토리는  /var/lib/airflow로 설정한다

```
sudo groupadd airflow
sudo useradd -s /bin/bash airflow -g airflow -d /var/lib/airflow -m
```

## Airflow의 정보가 저장될 데이터베이스로 사용될 Postgres 설치

Airflow는 기본으로 SQLite 데이터베이스를 설치하는데 이는 싱글 쓰레드라 다수의 DAG 혹은 다수의 Task들이 동시에 실행되는 것을 지원하지 못한다. 이를 나중에 Postgres로 교체한다. 

#### Postgres 설치

```
sudo apt-get install -y postgresql postgresql-contrib
```

#### 다음으로 Airflow가 Postgres를 접속할 때 사용할 계정을 Postgres위에서 생성

이를 위해서 postgres 사용자로 재로그인을 하고 postgres shell (psql)을 실행한 다음에 airflow라는 이름의 계정을 생성한다. 마지막에 exit를 실행해서 원래 ubuntu 계정으로 돌아가는 스텝을 잊지 않는다.

```
$ sudo su postgres
$ psql
psql (10.12 (Ubuntu 10.12-0ubuntu0.18.04.1))
Type "help" for help.

postgres=# CREATE USER airflow PASSWORD 'airflow';
CREATE ROLE
postgres=# CREATE DATABASE airflow;
CREATE DATABASE
postgres=# \q
$ exit
```

#### Postgres를 재시작

이 명령은 ubuntu 계정에서만 실행이 가능함을 꼭 기억!

```
sudo service postgresql restart
```


## Airflow 첫 번째 초기화 

#### 앞서 설치된 Airflow를 실행하여 기본환경을 만든다

이 때 SQLite가 기본 데이터베이스로 설정되며 이를 뒤에서 앞서 설치한 Postgres로 변경해주어야 한다.
```
sudo su airflow
$ cd ~/
$ mkdir dags
$ AIRFLOW_HOME=/var/lib/airflow airflow db init
$ ls /var/lib/airflow
airflow.cfg  airflow.db  dags   logs  unittests.cfg
```

#### Airflow 환경 파일(/var/lib/airflow/airflow.cfg)을 편집하여 다음 2가지를 바꾼다

 * "executor"를 SequentialExecutor에서 LocalExecutor로 수정한다
 * DB 연결스트링("sql_alchemy_conn")을 앞서 설치한 Postgres로 바꾼다
   * 이 경우 ID와 PW와 데이터베이스 이름이 모두 airflow를 사용하고 호스트 이름은 localhost를 사용한다
 
```
[core]
...
executor = LocalExecutor
...
[database]
...
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@localhost:5432/airflow
...
```

#### Airflow를 재설정

airflow 사용자로 아래 명령을 수행해야 한다
```
AIRFLOW_HOME=/var/lib/airflow airflow db init
```


## Airflow 웹서버와 스케줄러 실행

Airflow 웹서버와 스케줄러를 백그라운드 서비스로 사용하려면 다음 명령을 따라하여 두 개를 서비스로 등록한다. 
다음 명령들은 <b>ubuntu</b> 계정에서 실행되어야한다. 만일 "[sudo] password for airflow: " 메시지가 나온다면 지금 airflow 계정을 사용하고 있다는 것으로 이 경우 "exit" 명령을 실행해서 ubuntu 계정으로 돌아온다

ubuntu 계정으로 아래를 수행

#### 웹서버와 스케줄러를 각기 서비스로 등록

sudo vi /etc/systemd/system/airflow-webserver.service

```
[Unit]
Description=Airflow webserver
After=network.target

[Service]
Environment=AIRFLOW_HOME=/var/lib/airflow
User=airflow
Group=airflow
Type=simple
ExecStart=/usr/local/bin/airflow webserver -p 8080
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

sudo vi /etc/systemd/system/airflow-scheduler.service

```
[Unit]
Description=Airflow scheduler
After=network.target

[Service]
Environment=AIRFLOW_HOME=/var/lib/airflow
User=airflow
Group=airflow
Type=simple
ExecStart=/usr/local/bin/airflow scheduler
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

#### 다음 명령을 실행하여 앞서 서비스들을 활성화 한다

```
sudo systemctl daemon-reload
sudo systemctl enable airflow-webserver
sudo systemctl enable airflow-scheduler
```

서비스들을 시작한다:

```
sudo systemctl start airflow-webserver
sudo systemctl start airflow-scheduler
```

서비스들의 상태를 보고 싶다면 다음 명령을 실행한다:

```
sudo systemctl status airflow-webserver
sudo systemctl status airflow-scheduler
```

## 마지막으로 Airflow webserver에 로그인 어카운트를 생성한다

이는 airflow계정에서 실행되어야 한다. password의 값을 적당히 다른 값으로 바꾼다

```
sudo su airflow
AIRFLOW_HOME=/var/lib/airflow airflow users  create --role Admin --username admin --email admin --firstname admin --lastname admin --password admin4321
```

만일 실수로 위의 명령을 ubuntu 어카운트에서 실행했다면 admin 계정을 먼저 지워야한다. 지울 때 아래 명령을 사용한다

```
AIRFLOW_HOME=/var/lib/airflow airflow users delete --username admin
```

그리고나서 본인 서버를 웹브라우저에서 포트번호 8080을 이용해 접근해보면 아래와 같은 로그인 화면이 실행되어야 한다. 예를 들어 본인 EC2 서버의 호스트 이름이 ec2-xxxx.us-west-2.compute.amazonaws.com이라면 http://ec2-xxxx.us-west-2.compute.amazonaws.com:8080/을 웹브라우저에서 방문해본다. 기본적으로 Airflow 연결은 https (SSL)이 아닌데 이를 변경하고 싶다면 https://airflow.apache.org/docs/apache-airflow/1.10.1/security.html?highlight=celery#ssl 참고.


## 이 Github repo를 클론해서 dags 폴더에 있는 DAG들을 /var/lib/airflow/dags로 복사

keeyong/airflow-setup repository에 있는 dags 폴더의 내용을 /var/lib/airflow/dags로 복사한다. 

```
sudo su airflow
cd ~/
git clone https://github.com/keeyong/airflow-setup.git
cp -r airflow-setup/dags/* dags
```

## 이 Github repo의 업데이트가 필요한 경우

```
sudo su airflow
cd ~/airflow-setup
git pull
cd ..
cp -r airflow-setup/dags/* dags
```

그리고나서 Airflow 웹서버를 다시 방문해보면 (이 설치 작업을 한 시점에 따라) DAG들이 몇개 보이고 일부 에러도 몇개 보일 수 있다. 이 에러들은 나중에 하나씩 해결한다.

단 여기서 복사한 DAG들이 웹 서버에 나타나는데 시간이 좀 걸릴 수 있는데 그 이유는 Airflow가 기본적으로 5분 마다 한번씩 dags 폴더를 뒤져서 새로운 DAG이 있는지 보기 때문이다. 이 변수는 dag_dir_list_interval으로 airflow.cfg에서 확인할 수 있으며 기본값은 300초 (5분)이다. 


## Bash 파일(/var/lib/airflow/.bashrc)을 편집

airflow 사용자로 로그인시 (sudo su airflow등) AIRFLOW_HOME 환경변수가 자동설정되게 한다

```
airflow@ip-172-31-54-137:~$ vi ~/.bashrc
```

이 파일의 마지막에 다음 3개의 라인을 추가한다:
 - i를 누르면 편집모드로 들어간다 (ESC 키를 누르면 명령 모드로 들어간다)
 - 파일을 저장하고 나오려면 명령 모드에서 콜론(:)을 누르고 wq!를 입력한다
```
AIRFLOW_HOME=/var/lib/airflow
export AIRFLOW_HOME
cd ~/
```

exit을 실행하여 ubuntu 사용자로 나온 다음에 다시 airflow 사용자로 로그인하면 (sudo su airflow) 자동으로 홈디렉토리로 이동하고 AIRFLOW_HOME 환경변수가 설정되어 있음을 확인할 수 있다.
```
ubuntu@ip-172-31-54-137:~$ sudo su airflow
airflow@ip-172-31-54-137:~/$ pwd
/var/lib/airflow
airflow@ip-172-31-54-137:~/$ echo $AIRFLOW_HOME
/var/lib/airflow
```
