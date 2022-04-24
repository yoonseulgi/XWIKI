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
  ![ff](https://user-images.githubusercontent.com/89211245/164954549-b817485d-a20e-4ef6-bc63-d691138b4715.PNG)  
  
  - 기존 설정되어 있던 replication이 끊어짐!  
  ※ __참고__  
  - pg_is_in_recovery()를 통해 insert 쿼리를 수행하지 않아도 primary인지 확인 가능함(t: write query O, f: write query X)  
  
## 여기서 부터 replication 3번 이후에 작업과 같게 진행  
### 2. replication slot 생성  
  - 새로운 master에 slot 생성  
    ```sql 
    select * FROM pg_create_physical_replication_slot('repl_slot_01');  -- repl_slot_01 이름의 slot  생성  
    select * from pg_replication_slots; -- slot 확인  
    ```  
  ![slot](https://user-images.githubusercontent.com/89211245/164954765-36252655-a85b-46ca-94fa-a5b17b65df03.PNG)  


### 3. 기존 master db를 standby db로 변경  
  - pg_basebackup 사용  
    ```shell
    su - postgres  
    rm -rf /var/lib/pgsql/14/data     
    pg_basebackup -h 172.23.13.14 -D /var/lib/pgsql/14/data -U replication -R  
    ```
  
  ![new_re](https://user-images.githubusercontent.com/89211245/164955175-60dad866-3982-4e17-b422-c6081f3bd14b.PNG)  
  
  #### 여기서 postgresql 재시작이 안 되는 문제 발생
  - 기존 백업해둔 데이터 파일을 사용하면 재시작 됨  
  - 백업해둔 데이터 파일과 새롭게 백업된 데이터 파일의 차이를 비교해보니 파일 권한이 다름을 발견  
  - 기존 백업 데이터 파일의 사용자 권한은 postgres인데, 새로 백업된 데이터 파일의 사용자 권한은 root로 되어 있었음  
  - 새로 백업된 데이터 파일의 권한을 postgres로 바꿔주니 문제 해결  
    ```shell
    chown postgres.postgres /var/lib/pgsql/14/data 
    ``` 
  
### 4. 환경 변수 바꿔주기  
  - 기존 standby였던 db에서 data 디렉토리를 복사한 것이므로, primary 정보를 수정해야 함 

## Backup & Recovery  
### 방법1. pg_dump 사용  
  #### 1. 데이터베이스, 테이블 등 백업하고 싶은 단위로 백업
  - 데이터베이스 단위로 백업
    ```shell
    pg_dump -h localhost -U postgres -Fc -f postgres.backup postgres
            # -h host  
            # -U 사용자  
            # -Fc 백업 파일 형태  
            # -f 백업 파일 위치  
            # 마지막은 데이터베이스 명  
    ```  

  ※ __참고__  
    - pg_dump는 테이블, 인덱스, 함수 등 작은 단위로도 백업이 가능함  
      
  - 데이터베이스 복구  
      ```shell
      pg_dump -d postgres -U postgres -v -f postgres.backup
      ```
  - 테이블 삭제  
    ![테이블 삭제](https://user-images.githubusercontent.com/89211245/164956852-cf8ad446-04bc-4dfe-bcd0-63f5369da4b7.PNG)  
    
  - 복구 후  
    ![복구후](https://user-images.githubusercontent.com/89211245/164956862-db607665-c769-44e1-a940-8f1411e653ec.PNG)  

  
