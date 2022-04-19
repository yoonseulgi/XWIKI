### 1. postgresql.conf 파일 수정  
  - archivedir 디렉토리 생성 및 권한부여  
  ```shell
  mkdir /var/lib/pgsql/archivedir
  chown postgres:postgres /var/lib/pgsql/archivedir
  chmod 750 /var/lib/pgsql/archivedir
  ```  
  - postgres가 권한을 가져야 wal 파일을 archive 할 수 있음  
  
### 2. 사용자 생성 및 pg_hba.conf 추가 
  - replication을 위한 사용자 생성 
  ```shell
   sudo -u postgres psql
   create role replication with replication password 'password' login;
   
   # pgpool 사용을 위해 미리 모니터링할 사용자를 생성  
   create role pgpool with password 'pgpool' login;
   grant pg_monitor to pgpool;
   
   # pg_hba.conf 파일에 추가
   host    replication     replication     172.23.13.0/24          scram-sha-256
  ```
  
  
### 3. replication slot 생성  
  - physical_replication_slot 생성  
  ```sql
  select * FROM pg_create_physical_replication_slot('repl_slot_01');
  ```
  
### 4. standby server에만 속하는 속성 재정의
```shell
primary_conninfo = 'user=replication port=5432 host=172.23.13.11'
primary_slot_name = 'repl_slot01'
```
※ __주의__ primary로 부터 backup한 data 파일의 소유자가 누구인지 확인 필요   
