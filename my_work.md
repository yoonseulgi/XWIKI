## replication
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
  
### 4. master의 data 디렉토리를 복사
  - pg_basebackup 사용  
  ```shell
  su - postgres  
  rm -rf /var/lib/pgsql/14/data    // 기존 stand_by의 data 디렉터리 삭제   
  pg_basebackup -h 172.23.13.11 -D /var/lib/pgsql/14/data -U replication -R
  ```
  ※ __주의__  
  - 사용자를 postgres로 변경하지 않고 pg_basepback을 수행하면 data 디렉토리의 권한이 root로 설정됨.  
  - 그럼 db 실행이 안 됨.  
  
### 5. standby server에만 속하는 속성 재정의
```shell
primary_conninfo = 'user=replication port=5432 host=172.23.13.11'
primary_slot_name = 'repl_slot01'
```
※ __주의__  
- primary로 부터 backup한 data 파일의 소유자가 누구인지 확인 필요   

## HA(master <-> standby)  
- replication 거는 작업과 유사  
### 1. standby db를 master db로 승격
  - 기존 standby db에서 수행  
  ```shell
  select pg_promote();
  ```
  ※ __참고__  
  - pg_is_in_recovery()를 통해 insert 쿼리를 수행하지 않아도 primary인지 확인 가능함(t: write query O, f: write query X)  
  
### 2. 기존 master db를 standby db로 변경  
  - pg_basebackup 사용  
  ```shell
  su - postgres  
  rm -rf /var/lib/pgsql/14/data     
  pg_basebackup -h 172.23.13.14 -D /var/lib/pgsql/14/data -U replication -R  
  ```
  ※ __참고__  
  - pg_rewind  
  
### 3. 환경 변수 바꿔주기  
  - 기존 standby였던 db에서 data 디렉토리를 복사한 것임으로, primary 정보를 수정해야 함 
  
