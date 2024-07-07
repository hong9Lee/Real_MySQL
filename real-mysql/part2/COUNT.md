
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
