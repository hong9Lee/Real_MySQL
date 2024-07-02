
***UUID***

UUID
- 시점과는 별개로 랜덤한 값 생성
- 상대적으로 긴 문자열

B-Tree 인덱스의 성능 저해 요소
- 정렬되지 않은 키 값 생성 & INSERT
- 길이가 긴 키 값 (PK로 사용 시 모든 secondary index에 영향 미침)
- 일반적으로 UUID 컬럼은 유니크 제약 필요

인덱스가 아무리 크다 하더라도 WorkingSet 크기가 작다면 충분히 빠르게 처리 가능하다.  
UUID 컬럼 인덱스의 경우 전체 인덱스 크기 만큼의 메모리 필요.  

UUID vs BIGINT  
- UUID: 32 char(VARCHAR), 16 bytes(BINARY)  
- BIGINT: 8bytes  

1억건 테이블 (10개 인덱스를 가진 테이블의 PK인 경우)  
- 단일 인덱스 크기: 24GB vs 6GB  
- 전체 인덱스 크기: 264GB vs 66GB  

대체키를 활용해라..!  

내부적으로는 AutoIncrement or Timestamp 기반의 프라이머리 키를 사용.  
외부적으로는 UUID 기반의 유니크 세컨더리 인덱스  
```
CREATE TABLE table(
    id BIGINT NOT NULL AUTO_INCREMENT,
    external_uid CHAR(32) NOT NULL,
    ...
    
    PRIMARY KEY (id),
    UNIQUE INDEX ux_externaluid (external_uid)  
)
```




