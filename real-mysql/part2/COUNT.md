
***COUNT(*) vs COUNT(col)***    
 
조건 없는 COUNT()  
- 실행 계획이 동일하면 성능도 동일한가?    
```
CREATE TABLE counter ( /* INSERT counter VALUES (1,1); 1,000,000 rows */
    fd1 INT NOT NULL,
    fd2 INT
);


Case-1) SELECT COUNT(fd1) from counter; => 0.04 sec
Case-2) SELECT COUNT(*) from counter; => 0.04 sec
Case-3) SELECT COUNT(fd2) from counter; => 4.26 sec
-> counter table의 Primary key를 풀스캔하였다.
```

- Innodb_parallel_read_threads 설정  
mysql 8.0 버전은 조건없는 COUNT() 쿼리에 대해서 병렬 처리 지원.
비활성화 할 경우 쿼리 성능은 달라짐.

```
SET innodb_parrallel_read_threads=1; // 병렬처리 비활성화

Case-1) SELECT COUNT(fd1) from counter; => 4.31 sec
Case-2) SELECT COUNT(*) from counter; => 0.24 sec
Case-3) SELECT COUNT(fd2) from counter; => 4.26 sec
```  

정리  
- COUNT(*)와 COUNT(not_null_column)  
ha_records() 스토리지 엔진 API 사용.    
레코드 건수에 관계없이, 1회만 호출됨.  
COUNT(*) 쿼리는 레코드로부터 컬럼 추출 수행하지 않음.  
innodb_parallel_read_threads>=2의 경우,  
COUNT(not_null_column) 쿼리는 컬럼 추출 수행하지 않음.  

- COUNT(nullable_column)  
ha_index_next() 스토리지 엔진 API 사용.  
레코드 건수만큼 호출됨.


실행 계획  
```
EXPLAIN SELECT COUNT(*) FROM counter;

| id | table  |  type | possible_keys | key  | key_len | rows  |   Extra     |  
| 1  | counter| index |    NULL       | idx1 |    4    | 14460 | USING index |
index를 풀스캔 하고 있다.  
index가 Null인 컬럼도 포함하고 있다.  
Using index -> 커버링 인덱스로 처리되었다.
```  
커버링 인덱스?  
쿼리가 실행될때, 인덱스만 읽어도 쿼리가 처리될 수 있다는 것.  

쿼리의 성능 향상  
- 쿼리가 인덱스를 사용할 수 있도록 튜닝  
- 최소의 레코드만 테이블 데이터를 가져오도록 튜닝  

실행 계획 버그.  
조건 없는 count 쿼리는 실행계획과 다르게 항상 프라이머리 키를 이용하도록 쿼리가 작성되어 있다.  


조건을 가진 COUNT()  
```
Covering Index  

SELECT COUNT(1) FROM counter WHERE ix1='comment';
SELECT COUNT(*) FROM counter WHERE ix1='comment';
SELECT COUNT(ix1) FROM counter WHERE ix1='comment';

| id | table  |  type  | key  | key_len | rows  |   Extra     |  
| 1  | counter|  ref   | idx1 |    4    |  16   | USING index |


```  

```
Non-covering Index  

SELECT count(fd1)  
FROM counter
WHERE ix1='comment';

| id | table  |  type  | key  | key_len | rows  |   Extra     |  
| 1  | counter|  ref   | idx1 |    4    |  16   |   NULL      |

fd1 컬럼이 NotNull인지 확인한 다음 반환해야 한다.  
```

CoveringIndex 성능  
- CoveringIndex 실행 계획이 10배 정도 빠름  

특정 컬럼의 COUNT가 필요한 경우, 쿼리의 조건 또는 주석 표기!  
- SELECT COUNT(*) FROM tab WHERE nullable_col IS NOT NULL
전체 건수가 필요한 경우, COUNT(*) 만 사용 권장!  

