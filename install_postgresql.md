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


### 4. HA test
- 마스터 죽이고 다시 살리기  
- 서버가 가동된 이후로는 꺼지지 않도록 하고, 최대한 빠르게 구동되도록 하기 
    vip 사용하기, 접속 정보 바꾸기(master -> slave) 등  
            
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

