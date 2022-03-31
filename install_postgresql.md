# Install postgreSQL

### replication 설정
- 방법 1) Log-Shipping
    WAL 파일 자체를 전달하는 방식
    ※ WAL(Write-Ahead Logging) 파일 : 데이터베이스의 변경 내용을 로그 파일에 미리 저장한 후 작업을 수행 
- 방법 2) Streaming
    로그 내용을 전달하는 방식  
        
### 1. postgreSQL 설치하기
- 서버 생성하기(개발환경)  
- 서버에 DB 설치하기  
  
#### 나의 작업  
- Install the repository RPM:  
    sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm  

- Install PostgreSQL:  
    sudo yum install -y postgresql14-server  

- Optionally initialize the database and enable automatic start:  
    sudo /usr/pgsql-14/bin/postgresql-14-setup initdb  
    sudo systemctl enable postgresql-14  
    sudo systemctl start postgresql-14  

- postgresql 접속 : sudo -u postgres psql  
- postgresql 나가기 : EXIT  


### 2. optimizing 하기  
- 서비스 환경에 맞도록 DB 최적화시키기  
- 서버가 가동되기 전에 환경 설정에 필요한 파라메타 값을 찾아서 미리 설정해주기  
    why? 두 번 가동하지 않도록..  

#### 나의 작업
- postgresql 환경 파일 경로 : cd /var/lib/pgsql/14/data  
- max_connection : 최대 동시 접속자 설정  
    클라이언트의 연결요청으로 서버를 부하시키고, 서비스 중단까지 이어지는 문제를 방지하기 위해  
        
- shared_buffer : 1GB로 설정
    쿼리 결과를 메모리에 저장해주는 공간(캐시 메모리와 유사한 개념)    
    DB만 돌아가는 환경이면 버퍼의 크기를 넉넉하게 설정해도 되지만, 다양한 프로그램이 돌아가기에 고려해서 설정  
    서버 메모리 기준 25~40%가 적절  
- wal_buffer : commit하기 전, 디스크에 적재되지 않은 데이터 값들을 보관(log data)  
- dead tuple : select 성능 저하를 방지하기 위함
    ex) dead tuple이 (테이블 레코드 수 + 0.2 ) + 50 이 발생한 경우 autovacuum 동작
- orafce 확장 작업    

### 3. replication 설정하기
- __무중단__ 으로 설정하기   
- replication configration 파라메터 미리 찾아서 설정하기  

#### 참고  
- WAL(Write-Ahead Log) 파일?   
    데이터의 변경 사항이 기록된 파일  
    wal log는 세그먼트 파일 세트로 pg_wal 디렉터리에 저장  
    wal 파일의 목적은 wal 파일을 실행함으로 데이터베이스를 처음부터 다시 만들 수 있음.   
          
- replication slot이 필요한 이유?  
    스트리밍 복제를 실행하는 경우 오프라인이거나 연결이 끊어진 경우에도 wal 파일을 유지  
    replication slot은 두가지 유형이 존재  
    * pysical slot   
        * log shipping : wal 파일 자체를 전달   
            wal 파일이 일정 크기만큼 채워질 동안 전달이 일어나지 않음  
        * streaming : wal 파일 저장여부와 관계 없이 로그의 내용을 직접 전달  
            마스터와 스탠바이 서버 연결 네트워크에 문제가 없다면 데이터 유실 걱정이 적음
        select pg_create_physical_replication_slot(''); - slot 생성  
        select pg_drop_replication_slot(''); - slot 삭제  
       
    * logical slot   
    
    
#### 작업 1.
- replication 사용자 - repluser(user name)  
- 스탠바이 서버에서 마스터 서버에 접근할 replication 전용 유저 생성  
    CREATE ROLE repluser WITH REPLICATION PASSWORD 'zhtmzha123!' LOGIN;  
- 스탠바이 서버 pg_hba.conf 파일에서 아래와 같은 유저 정보 추가  
    host replication repluser ip주소(172.23.13.0/24) trust  
    (172.23.13.0으로 들어오는 connection은 수용하겠다는 의미로 해석)  
  
- 스탠바이 서버 /var/lib/pgsql/14/data 파일을 /for_pg_backup 위치에 백업  
    cp -rp /var/lib/pgsql/14/data /for_pg_backup  
    -r : 디렉토리 내 파일이 있어서 디렉토리 복사하기  
    -p : 복사할 디렉토리의 옵션도 그대로 복사  
    __why?__ 마스터 서버의 postgresql.conf 파일이 그대로 복사되기 때문에..  
         
- 마스터 서버의 postgresql.conf 파일에 설정 변수 추가  
    listen_addresses = '*'   인증/권한 관리는 pg_hba.conf 파일에서 진행  
    wal_level = replica  대기 서버에 읽기전용 작업 가능  
    max_wal_senders = 2   WAL 파일을 전송할 수 있는 최대 서버 수  
    wal_keep_segments = 32   마스터 서버 디렉토리에 보관할 최근 WAL 파일의 수  
    max_replication_slots = 32  서버가 지원할 수 있는 복제 슬롯의 최대 개수  
    port = 5432  
    
- 위와 같이 환경변수 설정 후 마스터 서버의 DB 재시작
    systemctl restart postgresql-14
    systemctl status postgresql-14  - postgresql-14.service의 상태를 확인할 수 있음
    
    #### 여기서 문제 발생 
    - systemctl restart postgresql-14 을 수행한 결과 에러 발생
    - systemctl status postgresql-14 실행 결과 아래 사진과 같이 에러 확인    
    - DB server가 active running이면 postmaster.pid 파일도 존재, active failed면 postmaster.opts만 존재  
![에러](https://user-images.githubusercontent.com/89211245/160507457-c8661507-5c0f-4250-81ab-86b25831af09.PNG)  
        
    #### 에러를 해결하기 위한 작업   
    1. systemctl stop postgresql-14 -> systemctl start postgresql-14  
    2. initdb.log에서 다음과 같은 명령어를 수행하여 db 서버 수행가능하다는 문구 확인 후 실행해봄  
        su postgres  - 서버 프로세스의 소유자로 변경 후 실행  
        /usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data/ -l logfile start    
        -> pg_ctl: could not start server 결과 나옴..  
        
    3. postgresql.conf 에서 재시작시 변경해야 할 환경변수 변경  
![화면설정](https://user-images.githubusercontent.com/89211245/160515160-45013f67-96df-4943-a310-c47849de125e.PNG)  
         
    4. optimizing을 위한 속성과 replication을 위한 속성을 나누어 수행  
      __replication을 위한 속성을 변경할 때 에러 발생 확인__  

### 에러 원인 파악
- postgresql.conf에서 replication을 위한 속성을 변경하면 에러가 발생한다고 생각했으나, pg_hba.conf를 설정하면 에러가 발생!  
- pg_hba.conf에 추가했던 주소들

``` text
    local   all   all                    trust
    host    all   all   172.23.13.0/24   trust
```

- 인증 방법을 trust로 설정하면 에러가 발생하는 것으로 예측
- local의 인증은 peer로, host의 인증은 md5로 수정하니 위와 같은 에러 발생 X

### 나의 작업  
- Master server  
    - DB에 replication을 위한 user 생성 - user name: replication, PW = password  
    - postgresql.conf에 replication을 위한 속성 설정  
```text
        listen_addresses = '*'  
        wal_level = hot_standby  
        max_wal_senders = 2       // stand_by server 개수보다 크게 설정  
        max_connections = 100     // stand_by server도 connection으로 분류되기 때문에 max_wal_senders 보다 큰 값으로 설정  
        max_replication_slot = 2  // 본 test에서는 slot 1개를 사용하지만, 여유를 위해 2로 설정함, max_replication_slot 값이 설정되어야 slot을 생성할 수 있음  
```
  
  
    - pg_hba.conf 에서 master server로 request가 올 ip주소 기입  
```text
        ex) host    replication    user_name    ip주소    md5  
        md5는 인증을 위한 method를 기입   
        
        host    replication    replication    172.23.13.0/24    md5
```

    - 위에서 명시한 ip주소로 요청이 들어오도록 방화벽 설정해주어야 함  
        - test 1. 방화벽 설정 없이 수행 : 방화벽 설정 없이도 수행 O     
        - test 2. 방화벽 설정 후 수행  
        
    - slot 만들기  
```text
sudo -u postgres psql # db postgres user로 접속  
SELECT * FROM pg_create_physical_replication_slot('repl_slot_01');  
```

- Stand_by server 
    - systemctl stop postgresql-14
    - master server의 data 디렉터리 그대로 백업하기  
```text
        su - postgres  
        rm -rf /var/lib/pgsql/14/data    // 기존 stand_by의 data 디렉터리 삭제   
        pg_basebackup -h 172.23.13.11 -D /var/lib/pgsql/14/data -U replication -R  
            -h: master server host  
            -U: replication user  
            -D: standby server 상의 data 디렉터리 저장 위치  
            -R: 복제를 위한 standby DB 설정을 자동으루 수행 postgresql.auto.conf에 자동 설정  
```
  
  
    - systemctl start postgresql-14  
    
        
    
- replication 됐는지 확인하기  
    - master server에서 select * from pg_stat_replication; 수행해보기  
    ![replinfo](https://user-images.githubusercontent.com/89211245/160772216-3291aa85-6aa2-4c15-825b-a7fdd2ca35bf.PNG)  
      
    - master server에서 db 변경하고, standby server에 적용되었는지 확인해보기  
    - ps -axf | grep postgres  
    ![svr1](https://user-images.githubusercontent.com/89211245/160772271-d758bdd7-f303-40a8-a4f6-b819e42c8391.PNG)      
    
  
        
### 4. HA test
- 마스터 죽이고 다시 살리기  
- 서버가 가동된 이후로는 꺼지지 않도록 하고, 최대한 빠르게 구동되도록 하기 
    vip 사용하기, 접속 정보 바꾸기(master -> slave) 등  
            
#### 참고
- Fail over: master가 죽었을 때, slave를 master로 전환  
- Fail back: slave를 통해 master를 복구 즉, master와 slave의 전환 X  
- warm standby server : master server의 장애로 stnadby server가 master로 승격할 때까지 연결할 수 없는 서버 - hot standby server : read-only 쿼리 요청은 받을 수 있는 서버   

#### 나의 작업
- promote_trigger_file
- pg_promote()
            
### ------- 여기까지 진행 후 덕우과장님께 말씀 드리기 ------
        
### 5. 백업 및 장애 복구하기
- drop table 후 테이블 복구
    덕우과장님께서 데이터를 계속 붙이시다가 랜덤한 테이블을 삭제할 계획
- 복구할 수 있는 단계까지 준비  

* 페이로드  
* 페이백 
* Full GC :  
    GC - 가비지 컬렉션: 사용하지 않는 메모리를 자동으로 수거하는 기능  
    heap 영역에서 GC가 발생했을 때  
    

## __질문__  
- replication되기 전엔 log에 기록되는데, replication 후엔 왜 log에 기록이 안됨?  
- table 추구, record 추가 및 삭제 등 테스트를 해보았는데.. standby server에 적용 됨  
- 만약 log가 기록이 되지 않는다면.. 복구를 어케 하나? 
- streaming replication 방법을 사용해서 wal 블록 사이즈만큼 안차더라도 레코드별로 전송되도록 설정했는데.. 