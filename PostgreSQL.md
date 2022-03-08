  # PostgreSQL
- 객체-관계형 데이터베이스(ORDBMS)  
- 오픈소스 DBMS  

## 아래 개념 중점으로 보기
high availity / HA / replication / sharding / GP primary segment mirror segment
- __high availity(= HA)__  
    서버, 네트워크, 프로그램 등 정보 시스템이 지속적으로 운용이 가능한 성질  
    2개의 서버를 클러스터로 엮어서 2개의 서버 중 한 서버에서 장애가 발생할 경우 다른 서버가 즉시 업무를 수행하므로 빠른 복구 가능  
- __replication__  
    데이터 백업을 위해 다른 곳으로 데이터를 복사하는 것  
    신뢰성, 성능 개선 등에 영향을 줌  
- __sharding__  
    하나의 큰 데이터베이스 또는 네트워크 시스템을 여러 개의 작은 조각으로 나누어 분산 저장하여 관리하는 것  
    데이터를 구간 별로 나눔으로 노드를 가볍게 하여 트랜잭션의 속도를 향상함  
    
<img src="https://user-images.githubusercontent.com/89211245/156968399-19286bcd-0238-4590-9899-a9ac705dc77e.jpg" width="300" height="300">     
  
## 기능 및 제한  
#### 트랜잭션과 ACID지원 <- 신뢰도와 안정성을 위한 기능
※ __ACID__ : Atomicity(원자성),  Consistency(일관성), Isolation(독립성),  Durability(영속성)  
    원자성: 트랜잭션을 구성하는 모든 연산은 전체 수행되거나 어떠한 연산도 수행되지 않는 속성  
    일관성: 트랜잭션 작업이 수행된 후에도 데이터베이스의 일관성을 유지해야 하는 속성  
    고립성: 트랜잭션 수행 중에 다른 트랜잭션에 영향을 주어서도, 받아서도 안되는 속성  
    지속성: 트랜잭션 수행이 완료되면 결과가 영구적으로 유지되어야 하는 속성(즉, 시스템 장애에도 견딜 수 있음)    
    
#### 진보적인 기능  
- Nested transactions(중첩 트랜잭션), savepoints  
    트랜잭션 내에 다시 트랜잭션을 정의하는 방법  
    가장 바깥쪽 트랜잭션이 커밋될 때까지 내부 트랜잭션이 수행이 되어도 변경 사항이 표시 되지 않음(원자성)  
    savepoins를 특정 시점을 두어 트랜잭션 오류가 발생시 savepoint로 롤백하여 트랜잭션의 전체 롤백을 방지
    
- Point in time recovery(PITR)  
    관리자는 과거 특정 시점의 데이터 or 특정 설정을 복원/복구할 수 있음  
    
- Online/hot backups, Parallel restore  
    __offline backup(= cold backup)__ : 데이터베이스가 오프라인 상태(사용자가 데이터베이스에 쓰기접근을 할 수 없는 상태)에서 백업이 진행됨  
    위와 같은 이유로 연중무휴로 작동되는 시스템에는 사용할 수 없음  
    __online backup(= hot backup)__ : 사용자가 데이터베이스에 접속한 상태에서도 백업 작업을 수행할 수 있음  
    백업 중 사용자가 데이터를 변경하면 데이터의 일관성이 유지되지 않을 수 있음   
    __Parallel restore__ : 여러 대의 장치를 동시에 사용하면서 복원 작업을 수행
    
- Rules system (query rewrite system)  
    사람이 만든 규칙을 적용하여 데이터를 저장, 정렬 및 조작하는 시스템  
    사전에 정의된 규칙을 사용하여 작업을 자동적으로 수행  
    __Q)__ 사용자가 작성한 쿼리를 DBMS가 더 나은 쿼리 실행계획으로 최적화하는 것을 의미하는 것인가요?
    
- B-tree, R-tree, hash, GiST method indexes <- 이와 같이 여러 인덱스를 지원  
    기본적으로 제공되는 인덱스는 __B-tree__ (between ~ in ~, in null, is not null 등을 수행할 때 인덱싱 됨)  
    __has index__ 는 인덱싱된 열의 값을 해시값으로 저장함. 해시 인덱시는 동등 비교만 처리 가능   
    __GiST(Generalized Search Tree)__ : 확장 가능한 데이터 구조, 사용자가 모든 종류의 데이터에 대한 인덱스를 개발 적용하여 조회할 수 있도록 함
    <key, pointer> 쌍을 포함하는 균형 잡힌 트리구조  
    GiST는 정수 키값을 가지는 B-tree와 달리 사용자 정의 클래스를 키값을 가짐  
    따라서 다양한 인덱싱 전략을 구현할 수 있음  
    ex) R-tree는 경계 박스를 키값으로 가짐  
    

    
- Information Schema Multi-Version Concurrency Control (MVCC)  
    DBMS의 동시성제어 방식  
    ※ __동시성 제어__ : 동시에 수행되는 다수의 트랜잭션이 작업을 성공적으로 마칠 수 있도록 트랜잭션의 실행 순서를 제어하는 기법  
    트랜잭션의 직렬성 보장, 공유도최대, 응답 시간 최소, 시스템 활동 최대, 데이터 무결성 및 일관성 보장  
    
    데이터의 업데이트가 필요할 때, 기존 데이터에 덮어쓰는 대신 데이터 항목의 새로운 버전을 만들어 여러 버전으로 저장  
    읽기 트랜잭션은 타임스탬스프나 트랜잭션 ID를 사용해 읽을 데이터베이스 상태를 결정하므로 읽기, 쓰기 트랜재션은 lock 기능 없이 다른 트랜잭션과 격리됨  
    ※ __타임스탬프__ : 트랜잭션을 식별하기 위해 DBMS가 부여하는 유일한 식별자로 트랜잭션 간의 순서를 미리 정의하는 동시성 제어 기법  
    
- Tablespaces  
    실제 데이터를 저장하는 공간(데이터베이스의 물리적인 부분)  
    테이블스페이스 단지 저장공간을 의미할 뿐 논리적 데이터베이스 구조에 영향을 주지 않음    
    __ex)__ 동일한 스키마 내의 다른 오브젝트는 서로 다른 테이블스페이스에 놓일 수 있음  
    -> 스키마 별로 테이블스페이스를 사용하면 관리가 용이함  
    테이블스페이스를 통해 디스크 배치를 통제할 수 있음  
    __ex)__ 빈번히 사용되는 데이터는 빠른 디스크에 저장하고, 거의 사용되지 않는 데이터는 느린 디스크에 저장  
    -> 데이터베이스 객체의 성능 최적화 가능
    
    ```
    select * from pg_tablespace;  
    열 속성  
    spcname - 테이블스페이스 이름  
    spcowner - 테이블스페이스 생성자  
    spcacl - 테이블스페이스 위치(디렉토리 경로)  
    spcoptions - 전급 권한  
    ```    
       
    postgreSQL엔 pg_default, pg_global 테이블 스페이스가 기본적으로 있음  
    사용자가 테이블스페이스를 정의하지 않으면 pg_default를 사용하고, pg_global는 시스템 정보가 저장되어 있음  
    
- Procedural Language(PL/SQL)
    SQL을 확장한 절차적 언어  
    SQL은 비절차적 언어  
    ※ __비절차적 언어__ : 데이터베이스 사용자가 SQL을 사용해 원하는 작업의 결과만 기술하고, 작업이 어떻게 수행될 것인지는 고려하지 않음
    * 종류  
    프로시저 : 리턴 값을 하나 이상 가질 수 있는 프로그램  
    함수 : 리턴 값을 반드시 반환해야 하는 프로그램  
    트리거 : 지정된 이벤트가 발생하면 자동으로 실행  
    패키지 : 하나 이상의 프로시저, 함수, 변수, 예외 등의 묶음을 의미  
    * PL/SQL 구조  
    선언부(declare) : 모든 변수나 상수를 선언하는 부분  
    실행부(excutable) : begin ~ end로 되어있으며, 로직을 기술하는 부분  
    예외처리부(exception) : 실행도중에 에러 발생시 해결하기 위한 명령을 기술하는 부분  
    * 특징  
    인터프리터 언어로 스크립트 생성 및 변경 후 바로 실행이 가능  
    코드 모듈화가 가능  
    변수, 상수 등 식별자 선언가능  
    절차적 언어 구조로 프로그램 작성 가능(if문, loop문 등)  
    에러 처리 가능  
    여러 sql문장을 block으로 묶어 전송하기 때문에 성능 향상을 기대할 수 있음  
 
- Information Schema  
    데이터베이스의 메타정보를 모아둔 데이터베이스
    Information Schema 내의 테이블은 실제 레코드를 가지고 있는 것이 아니라 조회할 때마다 메타 정보를 메모리에서 가져와서 보여줌  

- Database & Column level collation  
     collation 대조, 페이지 순서 조사  
     데이터베이스 수준, 열 수준 정렬을 지원   
- Array, XML, UUID type
    이와 같은 특별한 데이터 타입을 지원함    
    
- Auto-increment (sequences)  
    해당 속성이 컬럼에 지정되면 자동으로 1씩 증가하여 순서를 부여  
    
- Asynchronous replication
    비동기식은 동기식보다 복잡하지만 결과가 출력되는 시간동안 다른 작업을 할 수 있기에 자원을 효율적으로 사용할 수 있음   
    master db server와 slave db server의 동기화방식으로 __동기__ , __비동기__ 이 있음  
    ※ __replication__   
    사용자가 늘어남에 따라 DB가 쿼리를 처리하기에 힘든 상황이 되면서 DB를 늘림  
        
    <img src="https://user-images.githubusercontent.com/89211245/156983506-a7d40831-aeba-4be0-b8f1-3e9961ee4002.jpg" width="300" height="300">  
                  

    * 효과  
    데이터 요청을 slave로 분산함으로 병목현상 해결, 트랜잭션 집중에 따른 성능저하 해결(scale out solution)  
    서비스에 영향을 주지 않고 정보분석 가능(slave에서 정보 분석을 실행)  
        
    __Q)__ __DB단 복제__ , __스토리지단 복제__    
    db replication은 서버를 늘려서(= 증설 or scale out) 수행 및 스토리지 replication은 저장장치를 늘려서 수행하는 것이 맞나요? __NO__  
    
    * __DB level__  
        
    __그림 1__ ↓
    <img src="https://user-images.githubusercontent.com/89211245/157142168-adf9c97f-afdf-4806-acdb-8943158e86b6.jpg" width="200" height="300">  
    
    그림 1과 같이 같은 서버에 master db와 slave db를 설치할 수 있음  
    그러나 서버에 장애가 생기면 master db도 날라가기 때문에 일반적으로 그림 2와 같은 방법으로 사용됨
    ※ master db의 포트 번호 != slave db의 포트 번호
        
    __그림 2__ ↓
    <img src="https://user-images.githubusercontent.com/89211245/157142163-9c064d54-5015-4237-91cc-516be552c63e.jpg" width="300" height="300">
    
    GP도 그림2와 같은 방법을 사용함   
    primary segment = master  
    mirror segment = slave  
    
    * __storage level__   
        
    <img src="https://user-images.githubusercontent.com/89211245/157145102-0af0692e-0a5a-40d3-9861-f6dd68a2b287.jpg" width="200" heigth="200">  
    
    스토리지 replication은 스토리지를 증설하기 보다 스토리지의 영역을 나눠 replication을 수행함
    ex) 1PB면 500TB 영역을 replication 용도로 사용  

## 참고
``` text
- 파이버 채널(Fibre channel) : 스토리지 네트워킹에 쓰이는 네트워크 기술  
- 스토리지의 변화  
    memory + ssd + hdd 등 여러 종류의 기억장치를 조합함  
    memory엔 매우 빈번하게 사용되는 데이터, ssd는 빈번하게 사용되는 데이터, hdd는 빈번하게 사용되지 않는 데이터를 저장하여 보다 효율적으로 사용  
- 스토리지 replication이나 백업 등을 수행시 데이터를 복사하는 동안 변화된 데이터는 스토리지에 곧바로 적용하지 않음(= 프리징)
- DAS vs NAS vs SAN
1) DAS(Directed Attached Storage) : 네트워크를 거치지 않고 직접 연결되는 저장장치  
2) NAS(Network Attached Storage) : 네트워크에 연결된 파일 수준의 저장장치, 다른 서버에서도 접근 가능
3) SAN(Storage Area Network) : 대규모 네트워크 사용자를 위해 저장장치를 데이터 서버와 연결한 고속 스토리지 
```

    
- LIMIT/OFFSET
    limit: 출력할 행의 수
    offset: 몇 번째 row부터 출력할 것인지  

    
    
## 내부 구조
- PostgreSQL의 프로세스 구조   
    client는 인터페이스 라이브러리(jdbc, obdc 등)을 통해 서버에게 연결을 요청함 -> postmaster 프로세스가 서버와 연결을 중계 -> 클라이언트와 서버가 연결을 통해 질의를 수행  
        
- 서버 내부의 쿼리 수행 과장  
    쿼리구문 분석 -> 쿼리의미 분석 -> 서버에 정의된 룰에 따라 재정의 -> 실행 가능한 여러 쿼리 실행계획 중 최적화된 실행계획 선택 -> 실행 -> 사용자에게 결과 반환 
        
<img src="https://user-images.githubusercontent.com/89211245/157173519-5b1bdb30-15b0-41ab-8380-9b10d1b7f23e.jpg" width="200" height="200">  

    
    