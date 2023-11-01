# 3장 SQL의 조건 분기 실습

<br/>

## 들어가기.

- SQL의 기본 체계는 선언형입니다. SQL 세계에서 주역은 구문이 아니라 ‘식’ 입니다. SQL구문의 각 부분(SELECT, FROM, WHERE, GROUP BY, HAVING, ORDER BY)에 작성하는 것은 모두 식입니다. 열 이름 또는 상수만을 기술하는 경우에도 ‘연산자가 없는 식’이고, 상수만 있다면 ‘변수와 연산자가 없는 식’으로 부를 수 있습니다.
- SQL 세계에서 구문 내부에서는 식을 작성하는 것이고, 구문을 작성하지는 않습니다.
- **절차 지향형 사고방식에서 선언형 세계로 도약하는 것이 곧 SQL 능력 향상의 핵심입니다.**

<br/>

## 1. CASE WHEN 구문을 활용해 집계와 조건분기에 대해 실습해봅니다.

- 실습 조건 : postgreSQL을 활용한 실습, DBeaver 사용

<br/>

## (1) 실습 1

```jsx
[ 테이블 생성 ]
create table population (
location varchar(100), --// 지역명
sex INTEGER,           --// 1은 남자, 2는 여자
pop INTEGER            --// 인구수
);

[ 데이터 삽입 ]
insert into population(location,sex,pop)values('성남',1,60);
insert into population(location,sex,pop)values('성남',2,40);
insert into population(location,sex,pop)values('수원',1,90);
insert into population(location,sex,pop)values('수원',2,100);
insert into population(location,sex,pop)values('광명',1,100);
insert into population(location,sex,pop)values('광명',2,50);
insert into population(location,sex,pop)values('일산',1,100);
insert into population(location,sex,pop)values('일산',2,100);
insert into population(location,sex,pop)values('용인',1,20);
insert into population(location,sex,pop)values('용인',2,200);
```

- 문제 : 지역 별로 남자와 여자의 인구수를 아래 실행 결과와 같이 나오도록 출력하라

```jsx
[ 실행 결과 ]
location | pop_man | pop_won
수원	       90	    100
성남	       60	    40
용인	       20	    200
광명	       100	    50
일산	       100	    100
```

<br/>

```jsx
[ 정답 ]
select	location
,		SUM(case when SEX=1 then pop else 0 END) as POP_MAN
,		SUM(case when SEX=2 then pop else 0 END) as POP_WON
from POPULATION
group by location;
```

- 해설
    - 지역별로 묶기 위해 GROUP BY 구문에 지역(location)을 넣어 줍니다.
    - 그 후 SUM 함수를 통해 POP을 더해주는데 이 때 CASE 구문을 통해 성별(SEX)이 1인 경우와 2인경우를 나눠서 더해줍니다.
    
<br/>

## (2) 실습 2

```jsx
[ 테이블 생성 ]
create table employees(
emp_id integer,
team_id integer,
emp_name varchar(100),
team varchar(100)
);

[ 데이터 삽입 ]
insert into employees(emp_id,team_id,emp_name,team)values(101,1,'조씨','상품기획팀');
insert into employees(emp_id,team_id,emp_name,team)values(101,2,'조씨','개발');
insert into employees(emp_id,team_id,emp_name,team)values(101,3,'조씨','영업');
insert into employees(emp_id,team_id,emp_name,team)values(102,2,'김씨','개발');
insert into employees(emp_id,team_id,emp_name,team)values(103,3,'장씨','영업');
insert into employees(emp_id,team_id,emp_name,team)values(104,1,'백씨','상품기획');
insert into employees(emp_id,team_id,emp_name,team)values(104,2,'백씨','개발');
insert into employees(emp_id,team_id,emp_name,team)values(104,3,'백씨','영업');
insert into employees(emp_id,team_id,emp_name,team)values(104,4,'백씨','관리');
insert into employees(emp_id,team_id,emp_name,team)values(105,1,'하씨','상품기획');
insert into employees(emp_id,team_id,emp_name,team)values(105,2,'하씨','개발');
```

- 문제 : 소속된 팀이 1개라면 해당 직원의 이름과 팀이름을 그대로 출력, 1개 이상일 경우 n개를 겸무라고 출력하며 아래 결과대로 나오도록 출력한다.

```jsx
[ 실행 결과 ]
emp_name |    team
  조씨	    3개를 겸무
  하씨  	2개를 겸무
  백씨	    4개를 겸무
  김씨	      개발
  장씨	      영업
```

<br/>

```jsx
[ 정답 ]
select	emp_name
,		case when count(team_id) = 1 then max(team) else concat(count(team_id),'개를 겸무') end as team
from employees
group by emp_name;
```

- 해설
    - (1) 먼저 emp_name으로 group으로 묶어 줍니다.
    - (2) count 함수로 team_id를 카운팅하고 1일 경우 team을 출력합니다. 이 때 team은 group by 에 들어가지 않아 집계함수로 표시해주어야 합니다. 그래서 max로 감싸 주었고 어차피 팀은 1개이기 때문에 max를 써도 문제 없습니다.
    - (3) 팀을 카운팅한게 1이 아니라면 concat 함수로 카운팅한 숫자와 글자를 합쳐주어 출력해줍니다.
