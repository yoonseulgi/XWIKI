# SQLtuning

### 성능 개선을 위한 기본 지식  
- 실행계획 :  
  -  ex) index를 탔는지 확인  
- 옵티마이저 :  
- IO :  


※  unique index : 컬럼에 중복된 값이 없는 경우 사용하는 인덱스  
  ex) PK 컬럼사용  
  
※ oracle 행마다 식별자가 있음(rowid)  
   인덱스는 row id 를 저장하고 있음  
   
※ table 일부에 index를 검  
   table access와 index가 함께 오는데.. 만약 table access가 발생하지 않으면 성능이 좋아지겠죠?  
   
※ index는 너무 비싼 자원이기에 최소한의 개수로 사용해주어야 함  
 
 - hash join : 대량 데이터 사용할 때 성능 좋음  
    동등 조건일 대만 사용  
    범위 조건일 경우 merge join 사용  
 - nested loop join  
 - merge join  
   
# 수업 내용 정리  
### SQL 실행과정  
- 구문분석, 실행 계획 생성 -> 하드 파싱  
  - 실행계획 재사용 -> 소프트 파싱  
  - 하드 파싱, 소프트 파싱 바인드 변수 값이 결정하진 않음  
  - CBO기반으로 옵티마이저 실행하므로 통계 정보가 생성되어 있어야 함(I/O..)  
- index 실행시 table access가 동반하는데, table access 횟수는 적을수록 성능이 좋음  
  - index는 row id를 가지고 있음  
  - index는 고비용 기능이고, insert, delete, update의 비용이 비싸짐  
  - External Suppressing, Internal Suppressing(type 변화) 등이 일어나면 index 사용 X  
  - single block i/o를 사용함  
- 복합 index  
  - '='조건 컬럼을 인덱스 조건 제일 앞에 둬라  
- join  
  - nl join -> oltp에서 가장 많이 사용됨, 선행집합이 중요  
  - hash join -> 대용량, =조건 일 때, 선행집합이 중요  
  - merge sort  -> hash 조인 사용 못할 때  
- 페이징
  - inline view  
  - rownum '1' 값을 무조건 포함하고 있어야 함(top n sql)  
  - 범위 지정할 때 inline view 2번 감싸기  
