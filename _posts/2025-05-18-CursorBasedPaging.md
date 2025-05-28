---
title: 커서 기반 페이징 (Cursor Based Pagination)
categories:
  - implementation
feature_image: "https://picsum.photos/2560/600?image=872"
---

## 1. 페이지네이션 이란 ?
> 대용량 데이터 처리 방식을 나누다가 나온 주제로 정리를 하기 위해 이 글을 작성합니다.

페이지네이션 (Pagination)은 웹 페이지의 컨텐츠를 여러 페이지로 나누는 것을 얘기합니다. 웹 페이지일 수도 있고, 데이터일 수 있습니다.

---

### 조건 
페이지네이션을 하기 위한 조건은 아래와 같다.
1. 한 페이지에 얼만큼 보여주어야 하는가 <br>**limit** 혹은 **page size**
2. 시작점 <br>**offset**

---

### 구현
페이지 네이션에는 크게 2가지 구현 방식 존재합니다.
1. 오프셋 기반 (offset based pagination)
2. 커서드 기반 (cursor based pagination)

---

## 1. 오프셋 기반 페이징(offset based pagination)이란?
가장 널리 쓰이는 구현 방식, 특정 위치 (시작점, offset)에서 정해진 개수(limit) 만큼 데이터를 조회하는 방식입니다.

### 장점
#### 구현이 간단하다 
1. 관계형베이스 (MySQL, PostgreSQL 등)에서 LIMIT, OFFSET 키워드를 기본 제공한다.
    ```sql 
    SELECT * FROM posts
    ORDER BY created_at DESC
    LIMIT 10  //10개씩 가져와
    OFFSET 20; //시작점은 20부터
    ```
   ```sql
   //Oracle 12c 이상에서는 FETCH FIRST, OFFSEt 같은 구문을 지원한다.
   SELECT * FROM posts
   ORDER BY created_at DESC
   OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

   //Oracle 11g 이하에서는 서브쿼로 우회를 해야함;
   //21~30번째 행을 조회
   SELECT * FROM (
      SELECT a.*, ROWNUM rnum FROM (
        SELECT * FROM posts ORDER BY created_at DESC
      ) a WHERE ROWNUM <= 30
   )
   WHERE rnum > 20;
   ```
2. 조금 이따 얘기하겠지만 이전 페이지의 상태를 기억할 필요가 없이, limit와 page만 넘기면 된다.
3. Spring JPA, Django ORM 과 같은 프레임워크에서 .findAll({offset, limit}) 같은 옵션을 주면 간단히 구현이 된다.

### 단점
#### 데이터 변경 문제
오프셋의 문제는 몇개부터 몇개를 잘라오는 방식이기 때문에, 데이터가 중간에 추가가 되거나 삭제가 되면 **중복 문제**가 발생한다.

예를 들어
- A가 1~10번 페이지를 먼저 조회
- B가 1번 글을 삭제를 함
- A가 다음 페이지인 11번~20번을 조회하였지만 1번 글이 삭제되어 10번부터 19번까지 조회
- **10번의 글이 중복으로 조회가 됨.**

#### 대용량의 경우, 성능 문제가 존재
>사실 이 글을 쓰게된 핵심 이유가 여기 있다. <br/>
>>OFFSET은 DB 내부적으로 앞의 데이터를 전부 스캔 (FULL SCAN) 후에 건너띄는 방식이라, OFFSET이 크다면 성능이 떨어질수 있다.<br/>
예를 들어 OFFSET 방식에 의해 1000001번째 ~ 1000010번째 데이터를 조회한다고 하면, 앞의 100만개의 데이터를 FULL 스캔을 해야하는 것이다.

## 2. 커서 기반 페이징 (Cursor Based Pagination)
> 위 단점을 보완할 수 있는게 커서 기반 페이징 방식이다.

커서 기반 페이징(Cursor Based Pagination)은 **어디서 시작할지** 명확히 하는 방식이다.
커서 기반은 정렬의 기준이 되는 값이 필요한데, 이를 **커서 (Cursor)**라고 한다.

```sql
SELECT * FROM posts
WHERE created_at < '2024-01-01 12:00:00' //'2024-01-01 12:00:00' 가 커서이다. 해당 시점 이전 글 10개를 조회
ORDER BY created_at DESC
LIMIT 10;
```

### 장점
#### 데이터 중복 문제 해결
> 상황을 가정해보자. A 사용자가 최근 글 10개 1페이지를 본 이후 B 사용자가 글 하나를 삭제 > A 사용자가 2 페이지 요청

```sql
-- 1페이지
SELECT * FROM posts
ORDER BY id DESC
   LIMIT 10;

-- 2페이지 (A는 마지막 글의 id = 1000 을 커서로 기억하고 있음)
SELECT * FROM posts
WHERE id < 1000 //1000이 커서다
ORDER BY id DESC
   LIMIT 10;
```
- 어떤 값을 커서로 쓸건지도 고민해볼 대상이다. 날짜와 같은 값도 cursor가 될수 있지만, 고유성이 보장되고, 인덱싱이 되는 인자면 더욱 유리할 것. 또한 카디널리티가 낮은 (중복도가 높은) 인자가 검색에 유리할 것이다.
- 단 id 의 생성 순서가 보장되어야한다. UUID와 같은 랜덤한 값은 페이징하기 적절치 않다.

#### 성능이 보다 좋다.
> 커서 기반은 무작정으로 앞에서 풀스캔을 하지 않는다.
```sql
-- created_at에 인덱스가 있다면, 아래는 범위 조회로 빠르게 동작
SELECT * FROM posts
WHERE created_at < '2024-12-31 23:59:59'
ORDER BY created_at DESC
LIMIT 10;
```

## 100만건의 유저의 수를 가지고 있는 테이블 조회 성능 비교
![test](/assets/images/screentshot-20250528202554.png)
![test](/assets/images/screentshot-20250528202555.png)

## offset (928ms) 보다 cursor (472ms)로 두배의 성능 차이를 보인다.
  ![test](/assets/images/screentshot-20250528202556.png)
  ![test](/assets/images/screentshot-20250528202557.png)
  ![test](/assets/images/screentshot-20250528202558.png)


### 단점
#### 이전 페이지로의 이동이 어렵다.
- 이전 페이지로 이동을 하려면 커서를 되짚을 방법이 필요하다.
> A가 3페이지를 갔다가 2페이지로 돌아가려한다.<br>

1. 2페이지의 마지막 글을 기억하고
2. 정렬을 반대로하여, Limit 만큼 뽑아낸다.
```sql
-- created_at이 '2024-01-01'인 경우 이전 페이지를 보려면
//2페이지의 마지막 글이 12/30 23:59:59 에 작성되었다 가정하면
SELECT * FROM posts
WHERE created_at > '2024-12-30 23:59:59'
ORDER BY created_at ASC
LIMIT 10;
```

#### 오프셋처럼 특정 페이지 번호로 직접 이동 불가
> 9페이지 주세요 같은 특정 페이지로 이동은 불가하다.


## 3. 언제 cursor 기반이 좋을까?
> SNS 피드, 채팅 로그 등 무한 스크롤 같은 경우, 커서 페이징의 단점을 띄기 어려운 구조이며 <br>
> 실시간성으로 데이터가 자주 변하는 시스템의 경우 적절하다

## 4. 대용량 배치는 어떨까?
- 정산 같은 도메인을 개발하다 보면 하루에 수십 만건에서 수억건에 해당하는 데이터를 조회하는 경우가 있다.
- offset 이전의 데이터를 full scan 하는 offset 방식보다 cursor 방식 쿼리 성능 측면에서 유리하다.
- 또한 데이터의 중복이나 누락의 위험도가 없다.
```sql
-- 초기 커서
SELECT * FROM my_table
WHERE id > 0
ORDER BY id ASC
LIMIT 10000;

-- 이후 반복
SELECT * FROM my_table
WHERE id > :last_processed_id
ORDER BY id ASC
LIMIT 10000;
```

### **단, 반드시 PK의 정렬 기준이 순차적이고 고유해야한다. 그리고 병렬 처리시 범위를 쪼개서 병렬 커서 관리가 필요하다.**