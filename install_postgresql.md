# Install postgreSQL
        
### 1. postgreSQL 설치하기
- 서버 생성하기(개발환경)  
- 서버에 DB 설치하기  
  
#### 나의 작업  
- Install the repository RPM:  
    sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm  
    ※ apt-get : 우분투 환경	yum : 레드헷 리눅스 환경(centos 등..)    
- Install PostgreSQL:  
    sudo yum install -y postgresql14-server  

- Optionally initialize the database and enable automatic start:  
    sudo /usr/pgsql-14/bin/postgresql-14-setup initdb  
    sudo systemctl enable postgresql-14  
    sudo systemctl start postgresql-14  
    ※ initdb : 데이터베이스 클러스터를 생성함  
    		postgres, template1, template2 db가 기본적으로 생성되고, 사용자가 db를 생성할 때 template1 형식 그대로 복사됨  

- postgresql 접속 : sudo -u postgres psql  
- postgresql 나가기 : EXIT  


### 2. optimizing 하기  
- 서비스 환경에 맞도록 DB 최적화시키기  
- 서버가 가동되기 전에 환경 설정에 필요한 파라메타 값을 찾아서 미리 설정해주기  
    why? 두 번 가동하지 않도록..  
- 환경 속성 중 __memory__ > __cpu__ > __storage__ 순으로 중요함   

#### 나의 작업
- postgresql 환경 파일 경로 : cd /var/lib/pgsql/14/data  
- max_connections = 100  
    DB 서버에 최대 동시 연결 수  
- shared_buffers = 1GB  
    서버 메모리 기준 1/4~1/2 설정, 세션들이 공유하여 사용할 메모리 공간    
- effective_cache_size = 3GB  
    단일 쿼리가 사용할 수 있는 디스크 캐시 값 설정  
- maintenance_work_mem = 256MB  
    인덱스 생성, 외래키 등 유지보수에 사용되는 메모리 공간  
- checkpoint_completion_target = 0.9  
    wal segment가 갱신되는 시간. 즉, 데이터 손실을 줄이기 위해 check point를 갱신하는 시간    
- wal_buffers = 16MB  
    log가 기록되는 wal buffer의 크기  
- default_statistics_target = 100  
    특정 쿼리가 최적화되지 않은 경우 옵티마이저가 다른 계획을 선택하다록 하는 변수  
- random_page_cost = 4  
    비순차적으로 저장장치 페이지를 읽어오는 비용에 대한 추정치를 설정  
- effective_io_concurrency = 2  
    db 서버가 동시에 실행할 수 있는 디스크 i/o 개수 지정  
- work_mem = 10485kB  
    저장장치에 기록되기 전에 임시로 저장되는 메모리 크기 지정  
- min_wal_size = 2GB  
    wal size의 최소값  
- max_wal_size = 8GB  
    wal size의 최대값  
- max_worker_processes = 2  
    시스템이 지원할 수 있는 백그라운드 프로세스의 최대 수  
- max_parallel_workers_per_gather = 1  
    노드에서 실행될 수 있는 병렬 작업자의 최대 수  
- max_parallel_workers = 2  
    한 번에 활성화될 수 있는 병렬 작업자의 최대 수  
- max_parallel_maintenance_workers = 1  
    단일 명령어를 실행할 수 있는 병렬 작업자의 최대 수  
    
- 위와 같은 환경변수는 master에서만 설정하고, replication할 때 slave로 복제해야 하기에 slave엔 설정할 필요가 없었음.
    

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
    
- replication 설정
	- 방법 1) Log-Shipping
	    WAL 파일 자체를 전달하는 방식
	    ※ WAL(Write-Ahead Logging) 파일 : 데이터베이스의 변경 내용을 로그 파일에 미리 저장한 후 작업을 수행 
	- 방법 2) Streaming
	    로그 내용을 전달하는 방식  
    
#### 작업 1.
- replication 사용자 - replicaion(user name)  
- 스탠바이 서버에서 마스터 서버에 접근할 replication 전용 유저 생성  
    CREATE ROLE repluser WITH REPLICATION PASSWORD 'password' LOGIN;  
- 스탠바이 서버 pg_hba.conf 파일에서 아래와 같은 유저 정보 추가  
    host replication replication ip주소(172.23.13.0/24) trust  
    (172.23.13.0으로 들어오는 connection은 수용하겠다는 의미로 해석)  
  
- 스탠바이 서버 /var/lib/pgsql/14/data 파일을 /for_pg_backup 위치에 백업  
    cp -rp /var/lib/pgsql/14/data /for_pg_backup  
    -r : 디렉토리 내 파일이 있어서 디렉토리 복사하기  
    -p : 복사할 디렉토리의 옵션도 그대로 복사  
    __why?__ 마스터 서버의 postgresql.conf 파일이 그대로 복사되기 때문에..  
         
- 마스터 서버의 postgresql.conf 파일에 설정 변수 추가  
    listen_addresses = '*'   인증/권한 관리는 pg_hba.conf 파일에서 진행  
    wal_level = replica  대기 서버에 읽기전용 작업 가능  
    max_wal_senders = 2   WAL 파일을 전송할 수 있는 최대 서버 수, standby 서버가 1개 이기에 2로 설정  
    wal_keep_segments = 32   마스터 서버 디렉토리에 보관할 최근 WAL 파일의 수  
    max_replication_slots = 32  서버가 지원할 수 있는 복제 슬롯의 최대 개수  
    port = 5432  
    
- 위와 같이 환경변수 설정 후 마스터 서버의 DB 재시작
    systemctl restart postgresql-14
    systemctl status postgresql-14  // postgresql-14.service의 상태를 확인할 수 있음
    
    #### 여기서 문제 발생 
    - systemctl restart postgresql-14 을 수행한 결과 에러 발생
    - systemctl status postgresql-14 실행 결과 아래 사진과 같이 에러 확인    
    - DB server가 active running이면 postmaster.pid 파일도 존재, active failed면 postmaster.opts만 존재  
![에러](https://user-images.githubusercontent.com/89211245/160507457-c8661507-5c0f-4250-81ab-86b25831af09.PNG)  
        
    #### 에러를 해결하기 위한 작업   
    1) systemctl stop postgresql-14 -> systemctl start postgresql-14  
    2) initdb.log에서 다음과 같은 명령어를 수행하여 db 서버 수행가능하다는 문구 확인 후 실행해봄  
        su postgres  - 서버 프로세스의 소유자로 변경 후 실행  
        /usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data/ -l logfile start    
        -> pg_ctl: could not start server 결과 나옴..  
        
    3) postgresql.conf 에서 재시작시 변경해야 할 환경변수 변경  
![화면설정](https://user-images.githubusercontent.com/89211245/160515160-45013f67-96df-4943-a310-c47849de125e.PNG)  
	
	변경시 재시작을 잘못 해석함  
	
    4) optimizing을 위한 속성과 replication을 위한 속성을 나누어 수행  
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
- 인증 방법을 trust(비밀번호 없이 사용자로 접근)로 설정하려면 사전에 ssh key 설정이 필요  

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
        - 방화벽 설정이라는 것이 pg_hba.conf에서 설정을 변경하는 것을 의미?
          __NO!__  ph_hba.conf는 ip 설정하는 파일  
	  방화벽 -> 서버에서 막기 -> ip 테이블 -> pg_hba.conf  
        
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
- warm standby server : master server의 장애로 stnadby server가 master로 승격할 때까지 연결할 수 없는 서버 
- hot standby server : read-only 쿼리 요청은 받을 수 있는 서버   
- pgpool
- watchdog : pgpool 노드가 살아있는지 지속적으로 체크하고, 죽은 노드를 살아있는 노드로 vip를 셋팅하는 역할  
  pgpool을 설치하면 내장되어 있는 기능  

- recovery를 위한 속성을 자동으로 설정하는 명령어
  sudo -u postgres pg_basebackup -h 10.34.96.3 -D /var/lib/postgresql/13/main -U replication -v -P --wal-method=stream --write-recovery-conf
- replication만으로 slave -> master 승격시킬 경우, slave는 read-only이기 때문에 master에 insert하는 query가 입력됐을 때 runtime error가 발생  
※ pg_basebackup은 online 으로 진행  
※ ip주소 확인 - ip addr

#### failover 방법  
1. repmgr : vip 생성으로 auto-failover  
2. pgpool : db server와 db client 사이 미들웨어로 auto-failover 
3. promot_trigger_file : 수동으로 slave를 master로 승격  

#### 나의 작업
1. repmgr을 통한 auto-failover  
  - ssh key 등록 : 비밀번호 없이 사용자를 바꿀 수 있도록 하기 위함  
	  ``` shell
	  ssh -keygen -t rsa  
	  # .ssh가 저장될 위치 설정. 엔터입력시 기본 위치로 설정  
	  # 비밀번호 입력. 엔터시 비밀번호 없음
	  ssh-copy-id root@ip주소  
	  ```
  
  - repmgr 설치  
	  ```shell
	  yum -y install repmgr12*
	  ```
  ※ 참고 블로그는 postgresql ver 12을 사용해서 repmgr12를 install 했다고 추측됨  
      repmgr14* 를 사용해보니.. nopackage 라는 경고 문구가 떠서 repmgre12로 설치함  
      postgresql 11 버전부터 postgresql 형태가 많이 달라짐  
      따라서 11~14는 형태가 비슷하다고 생각되어 repmgre12로 설치  
      오류가 나면 repmgr 버전 다르게 해보기  
      
  - postgresql.conf, pg_hba.conf에 repmgr에 필요한 속성 추가  
	  ```shell

		# for repmgr

		local           replication             repmgr                                   peer
		host            replication             repmgr          127.0.0.1/32             md5
		host            replication             repmgr          172.23.13.0/24           md
	  ```  

  - /usr/pgsql-12/bin 위치에 있던 repmgr 을 /usr/pgsql-14/bin 위치로 이동 

### pgpool
- sudo apt-get install pgpool2
    apt-get은 리눅스 패키지를 다운로드 받을 때 사용하는 명령어   
    aws 등 클라우드에선 해당 명령어가 사용되지 않을 수 있음  
    사용되지 않을 경우 __yum__ 을 사용해서 다운로드 받으면 됨  
            
- pgpool 설치(가장 최신 버전으로 다운)  
    ```shell
    rpm pgpool-II-pg10-debuginfo-4.0.18-1pgdg.rhel7.x86_64.rpm  
    yum install pgpool-II-pg10  
    ```  
          
- pgpool.conf 파일 수정  
    - 기존 pgpool.conf 백업  
    ```shell
    cp pgpool.conf pgpool.conf.backup  
    ```  
    
    - 변수 변경
    ```shell
    backend_hostname0 = '172.23.13.11'
                                       # Host name or IP address to connect to for backend 0
    backend_port0 = 5432
                                       # Port number for backend 0
    backend_weight0 = 0.5
                                       # Weight for backend 0 (only in load balancing mode)
    backend_data_directory0 = '/var/lib/pgsql/14/data'
                                       # Data directory for backend 0
    backend_flag0 = 'ALLOW_TO_FAILOVER'
                                       # Controls various backend behavior
                                       # ALLOW_TO_FAILOVER, DISALLOW_TO_FAILOVER
                                       # or ALWAYS_MASTER

    backend_hostname1 = '172.23.13.14'
    backend_port1 = 5432
    backend_weight1 = 0.5
    backend_data_directory1 = '/var/lib/pgsql/14/data'
    backend_flag1 = 'ALLOW_TO_FAILOVER'
    
    
    failover_command = '/etc/pgpool-II/failover.sh %d %P %H %R'
                                       # Executes this command at failover
                                       # Special values:
                                       #   %d = node id
                                       #   %h = host name
                                       #   %p = port number
                                       #   %D = database cluster path
                                       #   %m = new master node id
                                       #   %H = hostname of the new master node
                                       #   %M = old master node id
                                       #   %P = old primary node id
                                       #   %r = new master port number
                                       #   %R = new master database cluster path
                                       #   %% = '%' character


	```  
	
- failover.sh 작성  
    - 쉘 스크립트란? 유닉스 환경 등에서 커맨드를 나열한 파일  
    	언제, 어떤 조건으로 어떤 커맨드를 실행할 것인가 등을 나열한 파일  
    	확장자 .sh  
    - #!/bin/sh 로 파일 작성 시작(쉘 스크립트를 작성한다고 명시)  
    - 쉘 스크립트를 실행하기 위해 __chmod__ -> __./~.sh__ 를 사용   
    - echo로 출력, read로 입력  
    	ex) read NAME  
	    echo "$NAME"  
	    사용자에게 NAME을 입력받은 후 출력  
    - if [조건] then  
      elif [조건] then  
      else  
      if  
    - #!/bin/sh -x : 디버깅 조건  
※ su user -c 명령어 : 사용자 변경과 명령어를 한 번에 입력     
            
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
