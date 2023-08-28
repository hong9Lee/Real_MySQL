## ***인덱스 실행계획***

##

#### ***INDEX***  </summary>


### Explain (실행계획)
MYSQL 옵티마이저가 수립한 실행 계획의 큰 흐름을 보여준다.


##
###### id 컬럼
SELECT 쿼리별로 부여되는 식별자 값이다.  
subquery, join 등을 수행할때 여러개의 id가 생성된다.



###### select_type 컬럼
각 단위 SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 컬럼이다.

`SIMPLE`  
UNION이나 서브쿼리를 사용하지 않는 단순한 SELECDT 쿼리의 경우 SIMPLE이 표시됨.  
쿼리가 복잡하더라도 SIMPLE인 단위 쿼리는 하나만 존재한다.  
일반적으로 제일 바깥 SELECT 쿼리의 select_type이 SIMPLE로 표시됨.

`PRIMARY`  
UNION이나 서브쿼리를 가지는 SELECT 쿼리의 가장 바깥쪽에 있는 단위 쿼리는 PRIMARY로 표시된다.  
PRIMARY인 단위 쿼리는 하나만 존재한다.

`UNION`  
UNION으로 결합하는 단위 SELECT 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 SELECT 쿼리의 select_type은 UNION으로 표시됨.  
첫번째 단위는 UNION되는 쿼리 결과들을 모아서 저장하는 임시 테이블(DERIVED)가 표시됨.

`DEPENDENT UNION`  
UNION이나 UNION ALL로 집합을 결합하는 쿼리에서 표시된다.  
DEPENDENT는 UNION이나 UNION ALL로 결합된 단위 쿼리가 외부 쿼리에 의해 영향을 받는 것을 의미한다.

ex)  
EXPLAIN  
SELECT *
FROM employees e1 WHERE e1.emp_no IN (  
SELECT e2.emp)no FROM employees e2 WHERE e2.first_name='Matt'    
UNION  
SELECT e3.emp_no FROM employees e3 WHERE e3.last_name='Matt'  
);

`UNION RESULT`  
UNION RESULT는 UNION 결과를 담아두는 테이블을 의미한다.  
select_type이 UNION RESULT인 경우 EXPLAIN table 컬럼에는 <union1, 2>와 같은 값이 출력된다.  
이는 explain의 id 1, 2의 조회 결과를 UNION 했다는 것을 의미한다.

##

###### partitions 컬럼

,,,  
PARTITION BY RANGE COLUMNS(hire_date)  
(PARTITION p1986_1990 VALUES LESS THAN ('1990-01-01'),  
PARTITION p1991_1995 VALUES LESS THAN ('1996-01-01'),  
PARTITION p1996_2000 VALUES LESS THAN ('2000-01-01'),  
PARTITION p2001_2005 VALUES LESS THAN ('2006-01-01'));

EXPLAIN  
SELECT *
FROM employees  
WHERE hire_date BETWEEN '1995-11-15' AND '2000-01-15';

위와 같은 경우, explain의 partitions 컬럼에는 p1996_2000, p2001_2005라고 표현되며 type에는 ALL이라고 표기된다.  
왜 풀스캔(ALL)으로 표현될까?  
MYSQL에서 지원하는 파티션은 물리적으로 개별 테이블처럼 별도의 저장공간을 가지기 때문이다.  
이 쿼리의 경우 모든 파티션이 아니라 p1996_2000, p2001_2005 파티션만 풀 스캔한다는 의미이다.


##  


###### type 컬럼
type 컬럼은 MYSQL 서버가 각 테이블의 레코드를 어떤 방식으로 읽었는지를 나타낸다. (인덱스를 사용했는지, 풀스캔을 했는지)
`  
system,  
const,  
eq_ref,  
ref,  
fulltext,  
ref_or_null,  
unique_subquery,  
index_subquery,  
range,  
index_merge,  
index,  
ALL
`

ALL을 제외한 나머지는 모두 인덱스를 사용하여 접근하는 방법이다.  
`하나의 단위 SELECT 쿼리는 단 하나만 사용 가능하다.`  
`index_merge를 제외한 나머지는 모두 인덱스 하나만 사용 가능하다.`

`system`  
레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블을 참조하는 형태의 접근 방법을 system이라고 한다.  
innoDB 엔진에서는 사용하지 않는다.

`const`  
테이블의 레코드 건수와 관계없이 쿼리가 프라이머리 키나 유니크 키 컬럼을 이용하는 where 조건을 가지고 있으며, 반드시 1건을 반환하는 쿼리의 처리 방식이다.  
다른 DBMS에서는 `유닉스 인덱스 스캔` 이라고도 표현한다.  
또, MYSQL의 옵티마이저가 쿼리를 최적화하는 단계에서 쿼리를 먼저 실행해서 통째로 상수화하는 경우 const라고 표현된다.

`eq_ref`  
여러 테이블이 조인되는 쿼리의 실행 계획에서만 표시된다.  
조인에서 두 번째 이후에 읽는 테이블에서 반드시 1건만 존재한다는 보장이 있어야 사용할 수 있다.

`ref`  
인덱스 종류와 관계없이 동등조건으로 검색할 때는 Ref 접근 방법이 사용된다.  
ref 타입은 반환되는 레코드가 반드시 1건이라는 보장이 없으므로 const나 eq_ref보다는 빠르지 않다.  
하지만, 동등 조건만으로 비교되므로 매우 빠른 레코드 조회 방법의 하나다.

`const`, `eq_ref`, `ref` 세 가지 접근방법 모두 WHERE 조건절에 사용하는 비교 연산자는 동등 비교 연산자여야 한다는 공통점이 있다.  
세 가지 모두 매우 좋은 접근 방법으로 인덱스의 분포도가 나쁘지 않다면 성능상의 문제를 일으키지 않는 접근 방법이다.  
쿼리를 튜닝할때 이 세가지 접근 방법에 대해서는 크게 신경 쓰지 않고 넘어가도 무방하다.

`fulltext`  
MYSQL 서버의 전문 검색 인덱스를 사용해 레코드를 읽는 접근 방법을 의미한다.  
"MATCH(...) AGAINST(...)" 구문을 사용해서 실행하는데, 이때 반드시 해당 테이블에 전문 검색용 인덱스가 준비돼 있어야만 한다.  
인덱스 생성 -> FULLTEXT KEY fx_name (first_name, last_name) WITH PARSER ngram  
검색 -> MATCH(first_name, last_name) AGAINST('Facello' IN BOOLEAN MODE);

`ref_or_null`  
ref 접근방법에 NULL 비교가 추가된 형태다.  
나쁘지 않은 접근 방법이다.

##  
###### key 컬럼
최종 선택된 실행계획에서 사용하는 인덱스이다.  
쿼리 튜닝시, key 컬럼에 의도했던 인덱스가 표시되는지 확인하는 것이 중요하다.  
PRIMARY가 표시되는 경우, 프라이머리 키를 사용한다는 의미이며, 그 이외의 값은 모두 테이블이나 인덱스를 생서할 때 부여했던 고유 이름이다.

###### key_Len 컬럼
쿼리를 처리하기 위해 다중 컬럼으로 구성된 인덱스에서 몇 개의 컬럼까지 사용했는지 알려준다.  
인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려주는 값이다.



##

###### rows 컬럼
옵티마이저 실행 계획의 효율성 판단을 위해 예측했던 레코드 건수를 보여준다.  
rows 컬럼에 표시되는 값은 반환하는 레코드의 예측치가 아니라, 쿼리를 처리하기 위해 얼마나 많은 레코드를 읽고 체크해야 하는지를 의미한다.

##


###### Extra 컬럼
쿼리의 실행 계획에서 성능에 관련된 내용이 표시된다.  
쿼리 실행에 있어 OO을 더 실행하겠다. 라는 내용이기에 NULL이 제일 좋다.




`Using filesort`  
ORDER BY 처리가 인덱스를 사용하지 못할 때만 실행 계획의 Extra 컬럼에 Using filesort 코멘트가 표시된다.  
이는 조회된 레코드를 정렬용 메모리 버퍼에 복사해 퀵 소트 또는 힙 소트 알고리즘을 이용해 정렬을 수행하게 된다는 의미이다.  
MYSQL 옵티마이저는 레코드를 읽어서 **소트 버퍼에 복사하고, 정렬해서 그 결과를 클라이언트에 보낸다.  
이러한 쿼리는 많은 부하를 일으키므로, 쿼리를 튜닝하거나 인덱스를 생성하는 것이 좋다.

** 소트 버퍼  
정렬을 수행하기 위해 별도의 메모리 공간을 할당 받아서 사용하는데, 이 메모리 공간을 소트 버퍼라고 한다.


##
`Using index(커버링 인덱스)`  
데이터 파일을 전혀 읽지 않고 인덱스만 읽어서 쿼리를 모두 처리할 수 있는 경우.  
인덱스를 통해 쿼리를 처리하는 경우 가장 큰 부하를 차지하는 부분은, 인덱스 검색에서 일치하는 키 값들의 레코드를 읽기 위해 데이터 파일을 검색하는 작업이다.

ex)   
select first_name, birth_date  
from employees  
where first_name BETWEEN 'babette' AND 'Gad';

where절에 일치하는 레코드가 5만건이라 할때,  
이 쿼리가 인덱스(Ix_firstname)을 사용한다면 일치하는 레코드 5만여 건을 검색하고, 각 레코드의 birth_date 컬럼의 값을 읽기 위해 각 레코드가 저장된 데이터 페이지를 5만여 번 읽어야 한다.  
이러한 경우 MYSQL 옵티마이저가 인덱스 보다 풀스캔으로 처리하는 것이 더 효율적이라고 판단할 수 있다.

ex)

select first_name
from employees  
where first_name BETWEEN 'babette' AND 'Gad';

이러한 경우 해당 테이블의 first_name 컬럼만 있으면 쿼리를 수행할 수 있다.  
이와 같이 인덱스만으로 처리되는 것을 "커버링 인덱스"라고 한다.


InnoDB의 모든 테이블은 클러스터링 인덱스로 구성되어있다.  
인덱스의 "레코드 주소" 값에 테이블의 프라이머리 키가 저장되어 있다.

ex)  
select emp_no, first_name
from employees  
where first_name BETWEEN 'babette' AND 'Gad';

위의 예제들과 다르게 테이블의 프라이머리 키는 이미 인덱스에 포함돼 있어 데이터 파일을 읽지 않아도 되며, 인덱스를 레인지 스캔으로 처리한다.


##
`Using temporary`    
MYSQL 서버에서 쿼리를 처리하는 동안 중간 결과를 담아 두기 위해 임시 테이블을 사용한다.  
임시 테이블은 메모리상에 생성될 수도, 디스크상에 생성될 수도 있다.

대표적으로 임시 테이블을 생성하는 쿼리.
```
1. FROM 절에 사용된 서브쿼리는 무조건 임시 테이블을 생성함. 이 테이블을 파생 테이블 이라고 부른다.  
2. COUNT(DISTINCT column1)을 포함하는 쿼리도 인덱스를 사용할 수 없는 경우에는 임시 테이블이 만들어진다.  
3. UNION이나 UNION DISTINCT가 사용된 쿼리도 항상 임시 테이블을 사용해 결과를 병합한다. (MYSQL 8.0 부터는 UNION ALL이 사용된 경우에는 임시 테이블을 사용하지 않도록 개선됨)  
4. 인덱스를 사용하지 못하는 정렬 작업 또한 임시 버퍼 공간을 사용하는데, 정렬해야 할 레코드가 많아지면 결국 디스크를 사용한다.
정렬에 사용되는 소트 버퍼 또한 임시 테이블과 같다.  
```


##

`Using Where`  
조인, 필터링, 집한처리,,, 등을 처리하는 MYSQL 엔진 레이어에서 별도의 가공을 통해 필터링(여과) 작업을 처리한 경우에 Extra 컬럼에 Using where 코멘트가 표시된다.


##

`Zero limit`  
데이터 값이 아닌 쿼리 결과값의 메타데이터만 필요한 경우 쿼리의 마지막에 LIMIT 0을 사용하면 Zero limit 메시지가 출력된다.






