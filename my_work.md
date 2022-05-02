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
          # -R replication에 필요한 primary info를 postgresql.auto.conf 파일에 자동 작성  
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
  - __에러1__  
    - 기존 백업해둔 데이터 파일을 사용하면 재시작 됨  
    - 백업해둔 데이터 파일과 새롭게 백업된 데이터 파일의 차이를 비교해보니 파일 권한이 다름을 발견  
    - 기존 백업 데이터 파일의 사용자 권한은 postgres인데, 새로 백업된 데이터 파일의 사용자 권한은 root로 되어 있었음  
    - 새로 백업된 데이터 파일의 권한을 postgres로 바꿔주니 문제 해결  
      ```shell
      chown postgres.postgres /var/lib/pgsql/14/data 
      ``` 
  - __에러2__  
    - 기존 db 서비스를 stop하지 않고 백업 받은 데이터 디렉토리 경로로 db 서비스를 시작하여 에러  
    - 이미 5432 포트가 열려있기 때문에 해당 포트에 접근하지 못해 발생하는 문제  
    - 기존 db 서비스가 가동된 data 디렉토리 경로로 db 서비스를 중단해줘야 함  
    
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
    - 증분백업이 불가능  
    - 증분백업이란 변경된 데이터만 백업하는 방식  
      
  - 데이터베이스 복구  
      ```shell
      pg_dump -d postgres -U postgres -v -f postgres.backup
      ```
  - 테이블 삭제  
    ![테이블 삭제](https://user-images.githubusercontent.com/89211245/164956852-cf8ad446-04bc-4dfe-bcd0-63f5369da4b7.PNG)  
    
  - 복구 후  
    ![복구후](https://user-images.githubusercontent.com/89211245/164956862-db607665-c769-44e1-a940-8f1411e653ec.PNG)  

### 방법2. wal 파일을 사용한 복구  
- 지정된 시점에 data 스냅샷을 갖는 파일 단위 복구 방법  
※ __스냅샷?__ : 스냅샷(snapshot)은 과거의 한 때 존재하고 유지시킨 컴퓨터 파일과 디렉터리의 모임  
- 모든 wal 파일을 재생하면 drop table, delete 등이 실행된 상태와 같은 상태가 됨  
- 위와 같은 이유로 특정한 시점(돌리고자 하는 시점)까지만 실행(__PITR__)  
- wal 파일은 서버가 삭제하거나 덮어쓰기 등으로 wal 파일이 유지되지 않을 수 있기에 다른 디렉토리에 보관할 필요가 있음 (__archive__)  
#### 1. 데이터 디렉토리 백업  
  - pg_basepackup 사용  
  - wal 파일 다른 디렉토리에 보관  
  ```shell
  pg_basebackup -D /backup/data -Fp
  
  # postgresql.conf
  archive_mode = on
  archive_command = 'test ! -f /var/lib/pgsql/archive/%f && cp %p /var/lib/pgsql/archive/%f'
  ```  
  - __test__ : 파일 존재유무, 타입, 권한을 체크  
    - 긍정이면 0, 부정이면 1 return   
### 여기서 에러 발생 
- __에러 1.__  
  - 백업한 data 디렉토리의 pg_wal을 사용해야 한다고 생각함  
  - 위와 같은 이유로 기존 pg_wal 파일을 사용하지 않고, 백업한 data의 pg_wal을 사용함  
  - 백업한 data의 pg_wal엔 지정한 시점에 정보가 없을 수 있기 때문에 에러가 발생했던 것  
- __에러 2.__  
  - 백업한 data 디렉토리 경로로 db 서버를 실행해야 하는데, 기존 data 디렉토리 경로로 db 서버를 실행함  
  - 위와 같이 실행하면 복구가 되지 않음  
  - 이유는 기존 wal 파일을 실행하면 현재 상태(drop table 등이 실행된 상태)가 유지되기 때문에  
#### 2. pg_wal 파일 복사 
  - db 서버를 위에서 백업한 데이터 디렉토리 경로로 실행할 것이기 때문에 기존 pg_wal 파일을 백업한 데이터 디렉토리로 복사  
  ```shell
  cp -r /var/lib/pgsql/14/data/pg_wal /backup/data/pg_wal
      # -r 디렉토리 하위 파일도 모두 복사 
      # -p 원본 파일의 소유자, 그룹, 권한 등의 정보까지 모두 복사
  ```
#### 3. 백업한 data 디렉토리 기존 db 실행 경로로 복사  
  - 기존 db 서버 실행 경로로 백업한 디렉토리를 복사하지 않으면 db 서버 실행할 때 경로도 함께 명시해주거나 db 실행 경로 수정이 필요함  
  ```shell
  cp -r /backup/data /var/lib/pgsql/14/data
  ```

#### 4. 복구 신호 
  - recovery.signal 파일로 복구 신호
  ```shell
  # data 디렉토리 내에 recovery.signal 파일 생성
  touch recovery.signal 
  ```
  - touch : 내용 없는 파일이 생성  
    
### 여기서 오류 발생  
- recovery_target_time을 지정된 형식에 맞게 입력해야 하나 형식을 다르게 입력함  
- 2022-04-24 18:16:29.651735+09(X)  
- 2022-04-24 18:16:29.651(O)  
#### 5. 원하는 시점 지정  
 - postgresql.conf에 명시  
 - restore_command는 서버가 재생할 wal 파일의 위치  
 - 서버가 wal 파일을 재생하면서 pg_wal 디렉터리로 wal파일을 복사함  
 - recovery_target_time은 복구하고자 하는 시점 명시(복구 프로세스가 중지할 시점)  
   ```shell
   restore_command = 'cp /var/lib/pgsql/archive/%f %p'
   recovery_target_time = '2022-04-24 18:16:29.651'
   ```
- drop table  
  <img width="262" alt="targettime" src="https://user-images.githubusercontent.com/89211245/164980787-8364afc0-22a6-49d7-a141-6fdd0a3cf8eb.png">  
- table 복구  
  <img width="245" alt="삭제된 테이블 살림" src="https://user-images.githubusercontent.com/89211245/164980998-c61eb5c9-f6c8-4b29-a6b3-6e8b12c43e6f.png">  

