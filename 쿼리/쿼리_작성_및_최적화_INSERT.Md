# ***쿼리 작성 및 최적화***  

## `INSERT` 
많은 INSERT 문장이 동시에 실행되는 경우 INSERT 문장 자체보다는 테이블의 구조가 성능에 더 큰 영향을 미친다.  

#### `INSERT IGNORE`  
레코드의 프라이머리 키나 유니크 인덱스 컬럼의 값이 이미 테이블에 존재하는 레코드와 중복되는 경우,  
그리고 저장하는 레코드의 컬럼이 테이블의 컬럼과 호환되지 않는 경우 모두 무시하고 다음 레코드를 처리한다.  
MYSQL 내부적으로 에러를 경고 수준의 메시지로 바꾸고 나머지 레코드의 INSERT를 진행한다.  


#### `INSERT...ON DUPLICATE KEY UPDATE`  
유니크 인덱스의 중복이 발생하면 UPDATE 문장의 역할을 수행하게 해준다.  
REPLACE도 유사해 보이지만 DELETE 와 UPDATE의 조합이다.  

일별로 집계되는 통계 데이터를 INSERT 할때 사용 예제
```
INSERT INTO daily_statistic 
    SELECT DATE(visited_at), 'VISIT', COUNT(*)
    FROM access_log
    GROUP BY DATE(visited_at)
    ON DUPLICATE KEY UPDATE stat_value=stat_value + VALUES(stat_value);
```

## `LOAD DATA`  
MYSQL 엔진과 스토리지 엔진의 호출 횟수를 초과하고 스토리지 엔진이 직접 데이터를 적재하기 때문에 일반 INSERT 보다 매우 빠르다.
하지만 단점은 단일 스레드, 단일 트랜잭션 실행 방식이다.  

테이블에 여러 인덱스가 있다면 레코드를 INSERT 하고 인덱스에도 키 값을 INSERT 해야한다.  
테이블에 레코드가 INSERT 될수록 인덱스의 크기도 커진다.  
단일 스레드로 동작하기 때문에 시간이 지날수록 INSERT 속도가 현저히 떨어진다.  

또한, 단일 트랜잭션 이기 때문에 언두 로그가 삭제되지 못하고 유지돼야 한다.  
언두 로그가 많이 쌓이면 레코드를 읽는 쿼리들이 필요한 레코드를 찾는데 더 많은 오버헤드가 발생한다.  
가능하다면 적재할 데이터 파일을 하나보다는 여러 개의 파일로 준비해서 동시에 여러 트랜잭션으로 나뉘어 실행되게 하는 것이 좋다.  
테이블간 복사 작업이라면 LOAD DATA 보다는 INSERT...SELECT... 문장으로 WHERE 절에서 데이터를 부분적으로 잘라서 INSERT 하는 것이 좋다.  
프라이머리 키 값을 기준으로 데이터를 잘라서 여러 개의 스레드로 실행하기가 용이하기 때문이다.  
