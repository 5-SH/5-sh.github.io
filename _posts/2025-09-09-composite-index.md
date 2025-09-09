---
layout: post
title: 복합 인덱스
date: 2025-09-09 21:10:00 + 0900
categories: [db]
tags: [db, index]
---

# 복합 인덱스

카테고리가 '전자기기'인 상품들 중 가격이 100,000원 이상인 상품을 조회해줘와 같은 다중 조건 쿼리의 성능을 최적화하기 위해 복합 인덱스를 사용한다.   
복합 인덱스는 두 개 이상의 컬럼을 묶어서 하나의 인덱스로 만든다.    
복합 인덱스를 만들 때 **컬럼의 순서**가 매우 중요하다. 어떤 컬럼 순서로 만드냐에 따라 쿼리의 성능이 많이 차이날 수 있다.   

<br/>

## 1. 컬럼의 순서가 중요한 이유

복합 인덱스의 동작 원리는 전화번호부나 국어사전과 같다.    
아래와 같은 스키마를 갖는 items 테이블에 (category, price) 순서를 갖는 복합 인덱스 ```idx_items_category_price```를 만들었다고 가정한다.   

|name|type|condition|
|---|---|---|
|item_id|INT|PRIMARY, KEY AUTO_INCREMENT|
|seller_id|INT|FK, NOT NULL|
|item_name|VARCHAR(255)|NOT NULL|
|category|VARCHAR(100)|NOT NULL|
|price|INT|NOT NULL|
|stock_quantity|INT| NOT NULL|
|registered_date|DATE|NOT NULL|

idx_items_category_price는 아래와 같이 구성된다.   

1. ```category```를 기준으로 먼저 정렬한다.
2. 같은 ```category```내에서는 ```price```를 기준으로 다시 정렬한다.

|category|price|item_id|
|---|---|---|
|도서|18000|21|
|도서|22000|25|
|도서|28000|15|
|도서|30000|4|
|도서|35000|11|
|생활용품|5000|16|
|생활용품|15000|5|
|생활용품|40000|9|
|생활용품|60000|19|
|전자기기|80000|3|
|전자기기|90000|14|
|전자기기|120000|1|
|전자기기|200000|24|
|전자기기|250000|7|
|전자기기|300000|13|
|전자기기|350000|23|
|전자기기|450000|2|
|전자기기|800000|20|
|전자기기|1500000|10|
|패션|45000|18|
|패션|70000|6|
|패션|95000|8|
|패션|180000|17|
|헬스/뷰티|20000|12|
|헬스/뷰티|25000|22|

이 구조에서 아래와 같이 조건에 따라 검색 효율이 달라질 수 있다.   

- ```category```로 검색: 매우 효율적이다. 인덱스에서 ```category``` 항목만으로 빠르게 찾을 수 있다.
- ```category```와 ```price```로 검색: ```category```로 필요한 항목을 찾은 다음 ```price``` 순으로 정렬된 데이터를 탐색하면 된다.
- ```price```만으로 검색: 매우 비효율적이다. ```price```값은 ```category```마다 흩어져 있기 때문에 인덱스 전체를 다 읽어야 한다.   

## 2. 인덱스 왼쪽 접두어 규칙(Index Left-Prefix Rule)

복합 인덱스는 첫 번째 컬럼을 기준으로 정렬된 상태에서만 제 역할을 할 수 있다.   
인덱스를 (A, B, C) 순서로 생성했다면, WHERE 조건을 (A), (A, B), (A, B, C)로 사용했을 때 효율적이다.   
(B), (C), (B, C)와 같이 첫 번째 기준인 A가 빠진 조건으로는 인덱스를 제대로 활용할 수 없다.   

## 3. 복합 인덱스 성공 예

### 3-1. category 사용

복합 인덱스의 첫 번째 컬럼만 WHERE 절에 사용하는 경우이다.   

```sql
EXPLAIN SELECT * FROM items WHERE category = '전자기기';
```

위 쿼리를 실행한 결과는 다음과 같다.

|id|type|key|rows|filtered|Extra|
|---|---|---|---|---|---|
|1|ref|idx_items_category_price|10|100.00|NULL|

- type: ref: category가 '전자기기'인 조건을 만족하는 데이터를 찾기 위해 동등 비교(=)나 JOIN을 사용하고 인덱스를 효율적으로 사용했음을 의미한다. 
  - Key 값 하나를 딱 집어 찾은 경우이다.
- key: idx_items_category_price: 복합 인덱스가 사용되었다.
- rows: 10: 옵티마이저는 '전자기기' 카테고리에 해당하는 상품이 약 10건 있을 것으로 예측하고 그 부분만 탐색한다.

### 3-2. category, price 함께 사용

복합 인덱스의 모든 컬럼을 WHERE 절에 사용하는 경우이다.

```sql
EXPLAIN SELECT * FROM items WHERE category = '전자기기' AND price = 120000;
```

위 쿼리를 실행한 결과는 다음과 같다.   

|id|type|key|rows|filtered|Extra|
|---|---|---|---|---|---|
|1|ref|idx_items_category_price|1|100.00|NULL|

- type: ref: 두 개의 컬럼 조건을 모두 만족하는 데이터를 찾기 위해 인덱스를 사용했다.
- rows: 1: 탐색할 행의 수가 1개로 예측된다.

category가 '전자기기'인 섹션 내부는 price 기준으로 정렬되어 있어 원하는 데이터를 빠르게 찾을 수 있다.

### 3-3. 복합 인덱스와 정렬

복합 인덱스는 정렬 작업을 피할 수 있어 매우 효율적이다.   
WHERE 절의 필터링과 ORDER BY의 정렬 방향이 인덱스 순서와 일치하면 데이터베이스는 불필요한 filesort 작업을 생략해 성능을 크게 높일 수 있다.   

```sql
EXPLAIN SELECT * FROM items WHERE category = '전자기기' AND price > 100000 ORDER BY price;
```

위 쿼리를 실행한 결과는 다음과 같다.   

|id|type|key|rows|filtered|Extra|
|---|---|---|---|---|---|
|1|ref|idx_items_category_price|8|100.00|Using index condition|

이 쿼리는 idx_items_category_price 인덱스를 사용해 category가 '전자기기'인 섹션으로 이동하고 색션 내에서 price가 100000을 초과하는 첫 번째 데이터를 찾는다.   
인덱스의 '전자기기' 섹션은 price 순서로 정렬되어 있으므로 그 지점부터 '전자기기' 섹션이 끝날 때 까지 인덱스를 순서대로 읽기만 하면 된다.   
그래서 Extra 컬럼에 Using filesort가 없다.   

위와 같은 이유로 아래와 같이 category, price로 정렬하고 인덱스 순서와 다른 컬럼으로 정렬을 요청하면 Extra 컬럼에 filesort가 발생한다.

```sql
EXPLAIN SELECT * FROM items WHERE category = '전자기기' AND price > 100000 ORDER BY item_name;
```

|id|type|key|rows|filtered|Extra|
|---|---|---|---|---|---|
|1|range|idx_items_category_price|8|100.00|Using index condition; Using filesort|

## 4. 복합 인덱스 대원칙

복합 인덱스를 설계하고 사용할 때는 다음 세 가지 원칙을 반드시 따라야 한다.   

1. 인덱스는 순서대로 사용한다(왼쪽 접두어 규칙)
2. 등호(=) 조건은 앞으로, 범위 조건(<, >)은 뒤로
3. 정렬(ORDER BY)도 인덱스 순서를 따른다


## 5. 복합 인덱스 실패 예

복합 인덱스를 잘못 사용하는 예를 확인하며 복합 인덱스의 대원칙을 설명한다.   

### 5-1. 인덱스 순서 무시

items 테이블에 idx_items_category_price 인덱스만 있을 때,   
복합 인덱스의 첫 번째 컬럼 category를 건너뛰고 두 번째 컬럼 price 만으로 데이터를 검색하면 풀 테이블 스캔이 발생한다.   

```sql
EXPLAIN SELECT * FROM items WHERE price = 80000;
```

|id|type|key|rows|filtered|Extra|
|---|---|---|---|---|---|
|1|ALL|NULL|25|10.00|Using where|

- type: ALL: 풀 테이블 스캔이 발생했다.
- key: NULL: idx_items_category_price 인덱스를 사용하지 않았다.   

데이터베이스 입장에서 idx_items_category_price 인덱스를 사용하더라도 price가 80000인 항목을 찾기 위해 category 마다 조회를 해야한다.   
이는 인덱스 전체를 스캔하는 작업으로 원본 테이블 전체를 스캔하는 것 보다 나을 것이 없다.   

### 5-2. 범위 조건을 먼저 사용

복합 인덱스를 사용할 때, 선행 컬럼에 범위 조건(>, <, BETWEEN, LIKE %)이 사용되면, 그 뒤에 오는 컬럼은 인덱스를 제대로 활용할 수 없다.   

카테고리 이름이 '패션' 이상인 상품들 중에서 가격이 20,000원인 상품을 찾는 쿼리가 있다.   
카테고리의 문자 정렬 순서로 보면 패션 이상인 것은 '패션', '헬스/뷰티' 이다.   

```sql
EXPLAIN SELECT * FROM items WHERE category >= '패션' AND price = 20000;
```

|id|type|key|rows|filtered|Extra|
|---|---|---|---|---|---|
|1|range|idx_items_category_price|6|10.00|Using index condition|

이 쿼리는 idx_items_category_price 인덱스를 사용해 category가 '패션'인 위치를 찾는다.    
그리고 그 위치에서 인덱스의 끝까지 모든 데이터를 스캔하며 price = 20000 조건을 만족하는지 하나하나 검사한다.   
그래서 filtered: 10.00%, Extra: Using index condition이 결과로 나왔다.   

다시 설명하면,
'패션' 섹션은 price 순으로 정렬되어 있고 '헬스/뷰티' 섹션도 price 순으로 정렬되어 있다.   

패션의 price

|price|item_id|
|---|---|
|45000|18|
|70000|6|
|95000|8|
|180000|17|

헬스/뷰티의 price

|price|item_id|
|---|---|
|20000|12|
|25000|22|

두 테이블을 하나로 합치면 정렬된 상태가 아니다.   
그래서 각각의 데이터에 대해 price = 20000 인지 하나씩 확인하는 필터링 작업이 추가된다.   

|price|item_id|
|---|---|
|45000|18|
|70000|6|
|95000|8|
|180000|17|
|20000|12|
|25000|22|

**이처럼 어떤 컬럼에 범위 검색을 사용하는 순간, 그 뒤에 오는 컬럼들은 인덱스의 정렬 효과를 제대로 사용할 수 없게 된다.**   

## 6. 범위 검색은 마지막에 한 번만 사용

복합 인덱스를 설계할 때, **등호(=) 조건을 사용하는 컬럼을 앞에, 범위 조건을 사용하는 컬럼을 뒤에 둔다.**   

예를 들어, 앞서 사용한 쿼리가 자주 사용된다면, 인덱스의 순서를 (price, category)로 변경해서 price=20000 등호 조건을 먼저 처리하도록 해야한다.   

```sql
SELECT * FROM items WHERE category >= '패션' AND price = 20000;
```

price, category 순서의 인덱스를 추가해 실행 계획을 확인하면 아래와 같다.

```sql
CREATE INDEX idx_items_price_category_temp ON items (price, category);
```

|id|type|possible_keys|key|rows|filtered|Extra|
|---|---|---|---|---|---|---|
|1|range|idx_items_category_price, idx_items_category_price_temp|idx_items_category_price_temp|1|100.00|Using index condition|

옵티마이저는 idx_items_category_price, idx_items_category_price_temp 두 인덱스 중 idx_items_category_price_temp가 더 효율적이라고 판단했다.   
그리고 Extra: Using index condition, rows: 1, filtered: 100.00을 통해 인덱스로 하나의 데이터를 찾았고 별도의 필터링 없이 선택된 것을 확인할 수 있다.   

idx_items_category_price_temp 인덱스

|price|category|item_id|
|---|---|---|
|5000|생활용품|16|
|15000|생활용품|5|
|18000|도서|21|
|20000|헬스/뷰티|12|
|22000|도서|25|
|25000|헬스/뷰티|22|
|28000|도서|15|
|30000|도서|4|
|35000|도서|11|
|40000|도서|9|
|...|...|...|

이처럼 **가장 변별력 있는 등호(=) 조건을 먼저 처리해서 작업 범위를 최대한 좁히고 다음에 범위 조건을 처리하는 것이 인덱스 설계의 핵심이다.**   

## 7. IN 절 활용하기

범위 조건 때문에 두 번째 인덱스 컬럼을 사용하지 못하는 문제는 >, < 대신 IN 절을 사용해 해결할 수 있다.   
MySQL 옵티마이저는 IN (...)을 하나의 큰 범위로 취급하지 않고, 여러 개의 동등 비교(=) 조건의 묶음으로 인식한다.   
```category >= '패션'``` 조건은 ```category IN ('패션', '헬스/뷰티')```와 논리적으로 동일하다.   

IN 조건을 사용한 아래 쿼리를 사용하면 

```sql
EXPLAIN SELECT * FROM items WHERE category IN ('패션', '헬스/뷰티') AND price = 20000;
```

실행 결과는 아래와 같다.

|id|type|key|rows|filtered|Extra|
|---|---|---|---|---|---|
|1|range|idx_items_category_price|2|100.00|Using index condition|

범위 연산자를 사용했을 때 예상 rows가 6개 였는데 2개로 줄었다. 인덱스로 찾는 범위가 더 줄었다는 뜻이다.   
그리고 filtered가 10.00%에서 100.00%로 바뀌었다. 이것은 인덱스만을 활용해 원하는 데이터를 100% 다 찾았다는 의미이다.   

<br/>

옵티마이저는 ```WHERE category IN ('패션', '헬스/뷰티')```를 ```WHERE category = '패션' OR category = '헬스/뷰티'```와 동일하게 인식한다.   
그래서 아래와 같이 나누어 실행된다.   

```sql
SELECT * FROM items WHERE category = '패션' AND price = 20000;
UNION ALL
SELECT * FROM items WHERE category = '헬스/뷰티' AND price = 20000;
```

'패션' 카테고리에서 price가 정렬되어 있어 인덱스를 사용해 원하는 데이터를 빠르게 찾고,   
'헬스/뷰티'에서도 price가 정렬되어 있어 인덱스를 사용해 원하는 데이터를 빠르게 찾을 수 있다.   
즉, 옵티마이저는 idx_items_category_price 인덱스를 사용해 ('패션', 20000) 지점으로 한 번, ('헬스/뷰티', 20000) 지점으로 한 번, 총 두 번의 정확한 위치 탐색을 수행한다.   

이 과정에서 price 조건까지 완벽하게 반영되므로 불필요하게 데이터를 읽고 버리는 과정이 사라지고 filtered 컬럼이 100.00%로 표시된다.   