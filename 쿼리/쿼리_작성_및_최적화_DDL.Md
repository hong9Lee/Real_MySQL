# ***쿼리 작성 및 최적화***  

## `스키마 조작(DDL)`  

#### 온라인 DDL 알고리즘  
온라인 DDL은 스키마를 변경하는 작업 도중에도 다른 커넥션에서 해당 테이블의 데이터를 변경하거나 조회하는 작업을 가능하게 해준다.  
ALGORITHM과 LOCK 옵션을 사용한다.  

1. ALGORITHM=INSTANCE  
테이블 데이터는 전혀 변경하지 않고, 메타데이터만 변경하고 작업을 완료한다.  
스키마 변경 도중 테이블의 읽고 쓰기는 대기하게 되지만 스키마 변경 시간이 매우 짧기 때문에 다른 커넥션의 쿼리 처리에는 크게 영향을 미치지 않는다.  


2. ALGORITHM=INPLACE
임시 테이블로 데이터를 복사하지 않고 스키마 변경을 실행한다.  
스키마 변경 중에도 테이블의 읽기와 쓰기 모두 가능하다.  
변경 최초 시작 시점과 마지막 종료 시점에는 테이블의 읽고 쓰기가 불가능하다.  

   
3. ALGORITHM=COPY  
테이블의 레코드를 모두 임시 테이블로 복사한 후 최종적으로 임시 테이블을 RENAME 해서 스키마 변경을 완료한다.  
테이블 읽기만 가능하다.  

`LOCK`  
INSTANCE의 경우 메타데이터만 변경하므로 시간이 짧아 LOCK을 사용할 수 없다.  
INPALCE나 COPY 알고리즘을 사용하는 경우 3가지 방법이 있다.  

NONE: 아무런 잠금을 걸지 않음.  
SHARED: 읽기 잠금을 걸고 스키마 변경을 실행하기 때문에 스키마 변경 중 읽기는 가능하지만 쓰기는 불가함.  
EXCLUSIVE: 쓰기 잠금을 걸고 스키마 변경을 실행하기 때문에 테이블의 읽고 쓰기가 불가함.  


## `컬럼 변경`  
MYSQL 8.0 부터 테이블의 컬럼 추가 작업은 대부분 INPLACE 알고리즘을 사용하는 온라인 DDL로 처리가 가능하다.  
컬럼을 테이블의 제일 마지막 컬럼으로 추가하는 경우에는 INSTANT 알고리즘으로 즉시 추가된다.  

테이블의 중간에 새로운 컬럼을 추가하는 경우, 테이블의 리빌드가 필요하여 INSTANT 알고리즘으로는 처리가 불가능하며 INPLACE 알고리즘으로 처리되어야 한다.  

## `컬럼 삭제`  
컬럼을 삭제하는 작업은 항상 테이블의 리빌드를 필요로 하기 때문에 INPLACE 알고리즘으로만 삭제가 가능하다.  

## `컬럼 변경`  
- 컬럼 이름만 변경하는 경우 실제 데이터 리빌드 작업은 필요하지 않아 INPLACE를 사용해도 INSTANT 알고리즘과 같이 빠르게 완료된다.  
- 컬럼의 타입을 변경하는 경우 COPY 알고리즘이 필요하며 스키마 변경 도중에는 테이블의 쓰기 작업은 불가하다.  
- 컬럼 타입의 길이를 늘리는 경우
  InnoDB 에서 varchar 컬럼의 길이를 관리할 때, DB 내부적으로 0~255 까지는 1바이트를 사용하고 256 바이트부터는 2바이트를 사용하는데  
  varchar 컬럼의 길이를 변경하면서 이 내부 바이트 길이가 변경되는 경우에는 INSTANT 알고리즘을 사용할 수 없다.  
  즉,  
  VARCHAR(10) 에서 VARCHAR(255)로 변경하는 경우는 INSTANT를 사용할 수 있으나  
  VARCHAR(10) 에서 VARCHAR(300) 으로 변경하는 경우에는 INSTANT 를 사용할 수 없다.  
  이는 VARCHAR(300)에서 VARCHAR(10)으로 변경하는 반대 케이스도 마찬가지이다.
  0~255 bytes 사이에서의 변경, 혹은 256bytes 부터 그 뒤로의 변경은 INPLACE 알고리즘,  
  0~255bytes 에서 256bytes 이상의 사이즈로 변경때는 COPY 알고리즘을 사용하면 된다.
- 컬럼 타입의 길이를 줄이는 경우 COPY를 사용하며 LOCK은 SHARED로 사용되어야 한다.


## `인덱스 이름 변경`  
인덱스 이름 변경하는 작업은 INPLACE 알고리즘을 사용하지만 실제 테이블 리빌드를 필요로 하지 않는다.  
```
ALTER TABLE employees
ADD INDEX index_new (first_name, last_name)
ALGORITHM=INPLACE, LOCK=NONE;

ALTER TABLE employees
DROP INDEX ix_firstname,
RENAME INDEX index_new TO ix_firstname,
ALGORITHM=INPLACE, LOCK=NONE;
```

## `인덱스 가시성 변경`  
인덱스의 가시성이란 MYSQL 서버가 쿼리 실행할 때 해당 인덱스를 사용할 수 있게 할지 말지를 결정하는 것이다.  
```
ALTER TABLE employees ALTER INDEX ix_firstname INVISIBLE;
```

## `인덱스 삭제`  
세컨더리 인덱스 삭제 작업은 INPLACE 알고리즘을 사용하지만 실제 테이블 리빌드를 필요로 하지 않는다.  
하지만, 프라이머리 키 삭제 작업은 모든 세컨더리 인덱스의 리프 노드에 저장된 프라이머리 키 값을 삭제해야 하기 때문에 임시 테이블로 레코드를 복사해서 테이블을 재구축 해야한다.  
프라이머리 키 삭제는 COPY 알고리즘을 사용해야하며, 키 삭제 도중 레코드 쓰기는 불가능한 SHARED 모드의 잠금이 필요하다는 것을 확인할 수 있다.  
```
ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=COPY, LOCK=SHARED;
ALTER TABLE employees DROP INDEX ux_empno, ALGORITHM=INPLACE, LOCK=NONE;
```








