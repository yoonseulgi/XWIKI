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
### replication user 생성  
```

```

### slot 생성
- standby가 모든 wal 파일을 수신할 때까지 primary는 wal 파일을 삭제하지 않음  
- primary와 standby의 연결이 끊긴 경우에도 충돌이 발생하지 않도록 wal 파일을 유지함  
    
    
