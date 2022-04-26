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
   
