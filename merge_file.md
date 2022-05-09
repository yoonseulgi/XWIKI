# postgreSQL  
## Install postgreSQL(master, standby server)  
  - postgresql 설치
  ```shell
  sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm  
  sudo yum install -y postgresql14-server  
  ```
  - .bash_profile  
    - 간단하게 사용하고 싶은 postgresql의 명령어를 명시  
    ```text
    [ -f /etc/profile ] && source /etc/profile
    PGDATA=/var/lib/pgsql/14/data
    export PGDATA
    # If you want to customize your settings,
    # Use the file below. This is not overridden
    # by the RPMS.
    [ -f /var/lib/pgsql/.pgsql_profile ] && source /var/lib/pgsql/.pgsql_profile
    alias pgstart='/usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data -l logfile start'
    alias pgrestart='/usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data -l logfile restart'
    alias pgstatus='/usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data -l logfile status'
    alias pgstop='/usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data -l logfile stop'
    ```
  - initdb 설정하기  
    - 한글 정렬을 가능하게 하려면 initdb 설정이 필요  
    ```shell
    # --lc-collate=C 옵션 
    /usr/pgsql-14/bin/initdb -D /var/lib/pgsql/14/data --lc-collate=C -E UTF-8
    ```
    
## opmizing  
  - 서비스 환경에 맞도록 DB 최적화  
  - 환경변수 중요도  
    - memory > cpu > storage  
    - https://pgtune.leopard.in.ua/ 사이트에서 hw 스펙에 맞는 환경변수 참고가능  
    ```shell 
    # /var/lib/pgsql/14/data/postgresql.conf
    
    max_connections = 100
    
    # 메모리
    shared_buffers = 1GB
    maintenance_work_mem = 256MB
    wal_buffers = 16MB
    work_mem = 128MB
    
    # 저장장치
    random_page_cost = 4
    effective_io_concurrency = 2
    min_wal_size = 2GB
    max_wal_size = 8GB
    
    # cpu
    max_worker_processes = 8              # processor * core 개수 추천, 8이 기본 설정 값
    max_parallel_workers_per_gather = 1
    max_parallel_workers = 2
    max_parallel_maintenance_workers = 1
    ```
  - archive 설정  
    - archive 디렉토리 생성  
    ```shell 
    mkdir /var/lib/pgsql/14/archive
    ```
    - 디렉토리 권한 __postgres__ 로 변경   
    ```shell 
    chown -R postgres.postgres /var/lib/pgsql/14/data/archive
    ```
    
## replication  
- Streaming Replication은 지속적으로 WAL XLOG 레코드를 primary 서버들의 변화되는 내용을 standby server들에게 전달 및 적용하는 기능을 제공 
### 1. replication user 생성  
```
# 방법 1. 
createuser -U postgres --replication repuser -P

# 방법 2. DB 접속 후 
CREATE ROLE repuser WITH REPLICATION LOGIN;
```
### 2. replication user 인증 설정  
- pg_hba.conf 파일에 replication user 정보를 입력하여 postgresql에 접속하는 user 인증 설정
```text 
host replication repuser 172.23.13.13/32 trust

# ip 대역으로 사용자 인증(172.23.13.0 대역 사용자는 DB 연결 허용)
host replication repuser 172.23.13.0/24 trust
```
__※ 주의__  
  - 인증 방식을 제대로 지정해주지는 않으면 에러 발생  
  - user 암호가 없다면 trust(인증 방법 없이 DB 접속 허용)  
  - 사용자 암호를 부여했다면 md5, scram-sha-256 등 인증 방식 설정  

### 3. slot 생성(primary server에서 진행)  
- standby가 모든 wal 파일을 수신할 때까지 primary는 wal 파일을 삭제하지 않음  
- primary와 standby의 연결이 끊긴 경우에도 충돌이 발생하지 않도록 wal 파일을 유지함  
```shell
# slot 생성 
postgres=# select * from pg_create_physical_replication_slot('replication_slot');  

# slot 확인  
postgres=# select * from pg_replication_slots;
```

### 4. replication 설정(standby server에서 진행)  
- hot_standby 설정 
```text
hot_standby = on   
```
- primary의 data 디렉토리 백업받기  
```shell
su - postgres
rm -rf /var/lib/pgsql/14/data    # 기존 stand_by의 data 디렉터리 삭제
pg_basebackup -h 172.23.13.12 -D /var/lib/pgsql/14/data -U repuser -R  
         # -h: master server host  
         # -U: replication user  
         # -D: standby server 상의 data 디렉터리 저장 위치  
         # -R: 복제를 위한 standby DB 설정을 자동으루 수행 postgresql.auto.conf에 자동 설정 
```
- (primary 서버 정보 자동설정 옵션(-R)을 주지 않은 경우) primary 정보 standby 환경 변수 파일에 변경   
```
primary_conninfo = 'user=repuser port=5432 host=172.23.13.12'
primary_slot_name = 'replication_slot'
```

### 5. replication 확인
- 방법 1. primary 서버 db에서 insert/delete 등 쿼리 실행하고, standby 서버 db에도 적용되었는지 확인  
- 방법 2. 
```shell 
postgres=# select * from pg_stat_replication;
```

## HA 
- 일반적으로 vip 설정 후 HA test 함  
- 해당 문서에서는 vip 설정 없이 HA test 했음 
- master <-> standby  
### standby server를 새로운 primary server로 승격  
```sql
select pg_promote();
```

### 기존 primary server를 새로운 standby server로 replication 설정  
#### 1. 새로운 primary 서버(기존 standby 서버)에 slot 생성  
  ```
  select * from pg_create_physical_replication_slot('replication_slot');
  ```
  
