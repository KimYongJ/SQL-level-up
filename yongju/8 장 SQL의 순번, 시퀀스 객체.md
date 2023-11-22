# 8장 SQL의 순번, 시퀀스 객체

# 1. 레코드에 순번을 붙이는 방법

- 먼저 사용할 테이블을 생성하고 값을 넣습니다.

```jsx
[ (1)에서 사용할 테이블 ]
create table weights(
student_id char(4) primary key,
weight integer
);

insert into weights values('A100', 50);
insert into weights values('A101', 55);
insert into weights values('A124', 55);
insert into weights values('B343', 60);
insert into weights values('B346', 72);
insert into weights values('C563', 72);
insert into weights values('C345', 72);

[ (2),(3) 에서 사용할 테이블 ]
create table weights2(
class integer not null,
student_id char(4) not null,
weight integer not null,
primary key(class, student_id)
);

insert into weights2 values(1, '100', 50);
insert into weights2 values(1, '101', 55);
insert into weights2 values(1, '102', 56);
insert into weights2 values(2, '100', 60);
insert into weights2 values(2, '101', 72);
insert into weights2 values(2, '102', 73);
insert into weights2 values(2, '103', 73);

[ (4)에서 사용할 테이블 ]
create table weights3(
class integer not null,
student_id char(4) not null,
weight integer not null,
seq integer null,
primary key(class,student_id)
);

insert into weights3 values(1, '100', 50, null);
insert into weights3 values(1, '101', 55, null);
insert into weights3 values(1, '102', 56, null);
insert into weights3 values(2, '100', 60, null);
insert into weights3 values(2, '101', 72, null);
insert into weights3 values(2, '102', 73, null);
insert into weights3 values(2, '103', 73, null);
```

## (1) 기본 키가 한 개의 필드일 경우

- 위에서 생성한 weights 테이블을 student_id를 기준으로 정렬 후 순번을 출력해보는 예시 입니다. 윈도우 함수와, 상관 서브쿼리를 사용해 구현하였습니다.

```jsx
[ 윈도우 함수 사용 ]
select student_id
,      row_number() over(order by student_id) as seq
from weights;

[ 상관 서브쿼리 사용 ]
select student_id
,      ( select count(*)
         from weights w1
         where w1.student_id <= w2.student_id
        ) as seq
from weights w2
order by student_id;
```

## (2) 기본 키가 여러 개의 필드로 구성되는 경우

- 위에서 생성한  weights2 테이블을 class와 student_id를 기준으로 정렬 후 순번을 출력해 보이는 예시입니다. 윈도우 함수와, 상관 서브쿼리를 사용해 구현하였습니다.

```jsx
[ 윈도우 함수 사용 ]
select class, student_id
,      row_number() over(order by class, student_id) as seq
from weights2;

[ 상관 서브쿼리 사용 ]
select class, student_id
,      (
          select count(*)
          from weights2 w2
          where (w2.class, w2.student_id) <= (w1.class, w1.student_id)
       ) as seq
from weights2 w1;
```

- **윈도우 함수 사용 해설 :** 단순히 row_number 함수에 정렬을 추가해주면  됩니다.
- **상관 서브쿼리 사용 해설 :** 다중 필드 비교를 사용합니다.(복합적인 필드를 하나의 값으로 연결해 한꺼번에 비교하는 기능입니다.) 다중 필드 비교는 문자열이나 숫자 상관없이 간단하게 확장 가능합니다.

## (3) 테이블을 그룹으로 분할했을 때 그룹 내부의 레코드에 순번을 붙이는 경우

- 위에서 생성한 weights2 테이블에 대해서 class 별로 그룹마다 구분을 지어 순번을 붙이는 경우를 윈도우 함수와 상관 서브쿼리를 사용해 구현해보겠습니다.

```jsx
[ 윈도우 함수 사용 ]
select class, student_id
,      row_number() over(partition by class order by class, student_id) as seq
from weights2;

[ 상관 서브쿼리 사용 ]
select class, student_id
,      (
          select count(*)
          from weights2 w2
          where w2.class = w1.class
          and   w2.student_id <= w1.student_id
          group by w2.class
       ) as seq
from weights2 w1;
```

## (4)  테이블에 순번을 갱신(update)하는 쿼리 입니다.

- 테이블은 weights3 테이블을 사용하며 테이블에 seq 값을 update하는 구문을 작성해 봅니다.

```jsx
[ 윈도우 함수 사용 ]
update weights3
set seq =   (select seq
            from (  select class, student_id,row_number() over(partition by class order by class, student_id) as seq
                    from weights3
                 ) subtable
           where  weights3.class = subtable.class
           and    weights3.student_id = subtable.student_id);

[ 상관 서브 쿼리 사용 ]
update weights3
set seq = (
            select count(*)
            from  weights3 w
            where weights3.class = w.class
            and   weights3.student_id >= w.student_id
          );
```

# 2. 레코드에 순번 붙이기 응용

## (1) 중앙 값 구하기

- 중앙 값이란, 레코드를 정렬했을 때 가운데 있는 레코드를 의미합니다.(평균과는 다른 의미)
- 테이블은 위에서 생성한 weights 테이블을 사용 합니다.

```jsx
[ 중앙 값 구하는 쿼리 ]
select avg(weight)
from (
        select w1.weight
        from weights w1, weights w2
        group by w1.weight
        having sum(case when w2.weight >= w1.weight then 1 else 0 end) >= count(*)/2
        and
        sum(case when w2.weight <= w1.weight then 1 else 0 end) >= count(*)/2
)tmp;
```

- **해설 :** case 식에 표현한 두 개의 특성 함수로 모집합 weights를 상위 집합과 하위 집합으로 분할 합니다.  avg함수는 테이블의 레코드 수가 짝수일 때도 계산이 가능하도록 평균을 계산하기 위해 사용하는 것입니다. 홀수일 때도 위 코드는 동작 합니다.
    - **특성함수 :** 어떤 값이 특정 집합에 포함되는지 판별하는 함수입니다. 포함되면 1, 포함되지 않으면 0을 리턴합니다.
- **위 코드의 단점**
    - **단점1 :** 코드가 복잡해서 무엇을 하는지 한 번에 이해하기 어렵습니다.
    - **단점2 :** 성능이 나쁩니다.

```jsx
[ 개선된 중앙 값 구하는 쿼리 ]
select avg(weight)
from (
        select weight
        ,      row_number() over(order by weight asc, student_id asc) as hight
        ,      row_number() over(order by weight desc,student_id desc) as low
        from weights
     ) tmp
where hight in (low,low-1,low+1);
```

- **해설 :** row_number 함수를 사용해 순번을 각각 붙이는데 이 때,내림차순과 오름차순으로 각각 붙여 놓습니다. 레코드가 홀수라면 hight = low 가 반드시 성립하지만 짝수일 경우 그렇지 않습니다. 그렇기 때문에 where 절에 hight in (low,low-1,low+1); 쿼리가 들어간것입니다. 데이터가 짝수일 때와 홀 수 일 때를 그림으로 표현하면 아래와 같습니다.

![사진1](https://github.com/KimYongJ/SQL-level-up/assets/106525587/18f383cd-61e4-46ae-977a-ed569501dd1a)

- 위 쿼리보다 성능적으로 더 개선된 쿼리를 짤 수 있습니다.

```jsx
[ 성능적으로 더 개선된 중앙 값 구하는 쿼리 ]
select avg(weight)
from (
       select weight
       ,      2*row_number() over(order by weight) - count(*) over() as diff
       from weights
     )tmp
where diff between 0 and 2;
```

- 위 코드를 이해하기 위해 아래 예시를 봅니다.

```jsx
[ 위 코드의 from 안에 있는 코드의 실행 결과 ]
select weight
,      2*row_number() over(order by weight) as row
,      count(*) over() as cnt
,      2*row_number() over(order by weight) - count(*) over() as diff
from weights;
```

- **실행결과**

| weight | row | cnt | diff |
| --- | --- | --- | --- |
| 50 | 2 | 7 | -5 |
| 55 | 4 | 7 | -3 |
| 55 | 6 | 7 | -1 |
| 60 | 8 | 7 | 1 |
| 72 | 10 | 7 | 3 |
| 72 | 12 | 7 | 5 |
| 72 | 14 | 7 | 7 |
- **해설 :** 순번 번호에 곱하기 2를하고 총 카운트 갯수를 빼면 중앙 값의 diff값은 홀수 일때는 무조건 1이나오며 짝수일 때는 숫자 0과 2로 2개가 나옵니다. 그렇기에 **where 조건에 diff의 범위를 0,1,2로 한다면 무조건 중앙 값을 구할 수 있게 됩니다.**

## (2) 순번을 사용한 테이블 분할

### 1) **단절 구간 찾기 연습 문제**

- 아래 테이블과 같이 1번부터 12번까지 숫자가 드문 드문 있습니다. 실무에서 이 상황은 레스토랑이나 영화관 좌석 예매시 있는 상황입니다.  비어있는 숫자는 예매가 완료된 좌석 이라 생각하고, **예매 완료된 자석의 범위를 구하는 쿼리를 만들어 주세요.**

```jsx
[ 테이블 생성 및 데이터 삽입 ]
create table numbers(
num integer primary key
);

insert into numbers values(1);
insert into numbers values(3);
insert into numbers values(4);
insert into numbers values(7);
insert into numbers values(8);
insert into numbers values(9);
insert into numbers values(12);
```

- **아래와 같이 출력 되도록 쿼리를 만들어 주세요 . 아래 테이블은 출력 결과 예시 입니다.**

| gap_start | ~ | gap_end |
| --- | --- | --- |
| 2 | ~ | 2 |
| 5 | ~ | 6 |
| 10 | ~ | 11 |

### **정답 코드 1.**

```jsx
select n1.num+1 as gap_start
,      '~'
,      min(n2.num)-1 as gap_end
from numbers n1
inner join numbers n2 -- // n1테이블에 n2를 추가 한다.
on n2.num > n1.num -- // n2를 추가할 때 n1의num보다 큰 것만 추가한다.(해당 조인 결과 테이블은 밑에 테이블로 그려놓음)
group by n1.num -- // n1의num을 기준으로 묶는다.
having n1.num+1 < min(n2.num); -- // 그룹으로 묶인 n1과 min으로 찾은 n2의 최소 num값의 차이가 2이상인 것만 추려낸다.
```

- **해설**
    - n2.num을 사용해 ‘특정 레코드의 값(n1.num) 보다 큰 숫자의 집합’을 조건으로 지정했습니다. (on n2.num > n1.num 부분.) 해당 집합을 표로 나타내면 아래와 같습니다. (길이가 길어 옆으로 넓혔습니다.)

| n1.num | n2.num |  | n1.num | n2.num |
| --- | --- | --- | --- | --- |
| 1 | 3 |  | 4 | 7 |
| 1 | 4 |  | 4 | 8 |
| 1 | 7 |  | 4 | 9 |
| 1 | 8 |  | 4 | 12 |
| 1 | 9 |  | 7 | 8 |
| 1 | 12 |  | 7 | 9 |
| 3 | 4 |  | 7 | 12 |
| 3 | 7 |   | 8 | 9 |
| 3 | 8 |  | 8 | 12 |
| 3 | 9 |  | 9 | 12 |
| 3 | 12 |  |  |  |

위와 같이 테이블이 생성됬을 때 **단절 구간을 찾기 위해서는, n1.num을 기준으로 group by를 하고, n2.num의 min 함수로 구한 컬럼 값과 group by로 묶은 n1.num의 차이가 2이상이면 단절 구간이 있는 것입니다.** 예를 들어 n1.num이 1인 구간들을 살펴보면 n2.num이 3~12까지 입니다. 이때 n1.num을 기준으로 그룹으로 묶고 n2.num의 min을 찾으면 n1.num은 1이고, n2.num은 3이 됩니다. 이 때 n1.num과 n2.num의 차이는 2입니다. 그럼 단절이 있는 것이지요. n1.num이 3인 구간은 n2.num의 min함수 연산 값이 4이기 때문에 단절 구간이 없습니다. n1.num이 4인 구간을 보면 n2.num의 min값이 7입니다. 즉 n1.num이 4, n2.num이 7이 되는 것이지요, 그럼 5~6이 단절구간이 됩니다.  n1.num이 각각 7, 8, 9일 때도 마찬가지로 같은 연산을 통해 단절구간을 구할 수 있습니다. 

### **정답 코드 2.**

```jsx
select num+1 as gap_start
,      '~'
,      (num + diff -1) as gap_end
from(
      select num
      ,      max(num) over(order by num rows between 1 following and 1 following) - num as diff
      from numbers
    )tmp
where diff<> 1;
```

- **해설 :** 이 쿼리에서 포인트는 윈도우 함수로 현재 레코드의 바로 다음 레코드를 구한 후 두 레코드의 숫자 차이를 diff 필드에 저장한 후 연산한다는 것입니다. 원리는 똑같 습니다. 숫자 차이가 1이 아닌 것들(where diff<> 1부분.)만 추려내면 갭을 구할 수 있습니다.  from 안에 select 구문을 출력해보면 아래와 같습니다.

| num | diff |
| --- | --- |
| 1 | 2 |
| 3 | 1 |
| 4 | 3 |
| 7 | 1 |
| 8 | 1 |
| 9 | 3 |
| 12 |  |
- 출력 결과를 보면 알겠지만 바로 다음 숫자와의 차이가 diff컬럼에 담겨 있습니다. diff가 1이 아닌 것만 출력하면 갭을 구할 수 있습니다.

### 2) 예매 안된 구간 찾기 연습 문제

- 이번에는 빈 구간이 아니라 **예매가 안된 구간을 찾는 문제**입니다. **실무에서는 인원수에 맞게 자리를 예약하고 싶은 경우 덩어리를 구할 때 사용 할 수 있습니다.** 테이블은 위에서 생성한 numbers 테이블을 사용합니다.

### 정답 코드 1.

```jsx
select min(num) as low
,      '~'
,      max(num) as hight
from (
       select n1.num
       ,      count(n2.num) - n1.num as gp
       from numbers n1 
       inner join numbers n2
       on n2.num <= n1.num
       group by n1.num
)N
group by gp
order by low;
```

- **출력 결과**

| low | ~ | hight |
| --- | --- | --- |
| 1 | ~ | 1 |
| 3 | ~ | 4 |
| 7 | ~ | 9 |
| 12 | ~ | 12 |
- **해설 :** 위 코드에서 from절 안에 있는 쿼리를 실행 해보면 아래와 같은 결과가 나옵니다.

| num | gp |
| --- | --- |
| 1 | 0 |
| 3 | -1 |
| 4 | -1 |
| 7 | -3 |
| 8 | -3 |
| 9 | -3 |
| 12 | -5 |

이런 결과를 바탕으로 gp를 group by해서 그룹으로 묶은 후, min, max함수를 활용해 num을 구하면 해당 구간이 나오게 됩니다.

# 3. 시퀀스 객체. IDENTITY 필드

- 표준 SQL에는 순번을 다루는 기능으로 시퀀셜 객체, IDENTITY 필드가 있습니다. 하지만 IDENTITY 필드는 오라클이 지원하지 않고, 시퀀스 객체는 MySQL이 지원하지 않습니다. 그렇기에 되도록 사용하지 않아야 합니다.

## (1) 시퀀스 객체

```jsx
[ 시퀀스 객체 정의 예시 ]
create sequence testseq
start with 1 -- //초기 값
increment by 1 -- // 증가 값
maxvalue 100000 -- // 최댓 값
minvalue 1 -- // 최솟값
cycle; -- // 최댓값에 도달했을 때 순환 유무
```

- **시퀀스 객체의 문제점**
    - 표준화가 늦어서 구현에 따라 구문이 달라 이식성이 없고, 사용할 수 없는 구현도 있다.
    - 시스템에서 자동으로 생성되는 값이므로 실제 엔티티 속성이 아니다.
    - 성능적인 문제를 일으킨다.
    

## (2) IDENTITY 필드

- IDENTITY 필드는 ‘자동 순번 필드’라고 합니다. 테이블의 필드로 정의하고, 테이블에 INSERT가 발생할 때 자동으로 순번을 붙여 줍니다. 시퀀스 객체는 테이블과 독립적이어서 여러 테이블에서 사용할 수 있으나 IDENTITY 필드는 특정한 테이블과 연결됩니다. 성능적으로 개선할 방법이 제한적이라 IDENTITY 필드 사용시 이점이 거의 없다고 할 수 있습니다.

# 마무리

- SQL에서 초기에 배제했던 절차 지향형이 윈도우 함수라는 형태로 부활
- 윈도우 함수를 사용하면 코드를 간결하게 기술할 수 있으므로 가독성 향상
- 윈도우 함수는 결합 또는 테이블 접근을 줄이므로 성능 향상을 가져옴
- 시퀀스 객체 또는 IDENTITY 필드는 성능 문제를 일으키는 원인이 되므로 사용할 때 주의가 필요
