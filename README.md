# Sehyunk 마케팅 대시보드

![interactive](images/interactive.gif)

- https://public.tableau.com/app/profile/.51348188/viz/Sehyunk/1

- 가상의 서비스의 데이터 테이블을 제작

  - '유입 > 회원가입 > 핸드폰인증 > 구매' 로 퍼널 세팅

- 빅쿼리에서 시각화 할 수 있는 데이터로 쿼리작성하여 출력

- 테블로를 이용하여 시각화

  

## 데이터 테이블 제작

- Visit: 서비스 유입 로그. cid로 각 트래픽을 구분하고, utm 파라미터를 통해 유입된 마케팅 채널을 구분. 날짜정보 포함.
- User: 회원가입 로그. 날짜정보 포함.
- Cert_log: 핸드폰 인증 로그. 날짜정보 포함.
- Order: 구매 로그. 구매자와 구매 금액 포함. 날짜정보 포함.



## 빅쿼리로 쿼리작성

- 날짜와 마케팅 채널별로 group by
- 각 퍼널별로 유저수와 총 매출을 집계
- 각 퍼널별로 집계할 대상이 달라 퍼널을 stage컬럼으로 구분하고 따로 집계
- union all을 통해 하나의 테이블로 취합
- marketing-dashboard.csv로 출력

```sql
# visit(유입)퍼널 집계
select 'visit' as stage, # union 후에도 퍼널을 분류할 수 있게 'visit'고정값으로 stage컬럼 생성
		v.date, # 날짜
    v.utm_source, v.utm_medium, v.utm_campaign, # utm 파라미터
    count(v.cid) as values, # 유입 유저수 카운트
    0 as sales # 유입 퍼널에선 매출이 없기 때문에 0 고정값으로 sales 컬럼 생성(union을 위해 컬럼을 맞춰줘야 함)
from `funnel-visualization.sample_data.visit_log` as v
group by v.utm_source, v.utm_medium, v.utm_campaign, v.date # utm 파라미터와 날짜로 group by
union all 
# sign in(회원가입)퍼널 집계
select 'sign in' as stage, v.date,
    v.utm_source, v.utm_medium, v.utm_campaign, 
    count(u.user_id) as values, 0 as sales
# group by할 utm파라미터가 visit 테이블에 있기 때문에 join 필요
from `funnel-visualization.sample_data.visit_log` as v
        left join `funnel-visualization.sample_data.user` as u on v.cid = u.cid
group by v.utm_source, v.utm_medium, v.utm_campaign, v.date
union all 
# cert_phone(핸드폰 인증)퍼널 집계
select 'cert_phone' as stage, v.date,
    v.utm_source, v.utm_medium, v.utm_campaign, 
    count(c.cert_id) as values, 0 as sales
from `funnel-visualization.sample_data.visit_log` as v
        left join `funnel-visualization.sample_data.user` as u on v.cid = u.cid
        left join `funnel-visualization.sample_data.cert_log` as c on u.user_id = c.user_id
group by v.utm_source, v.utm_medium, v.utm_campaign, v.date
union all 
# order(구매)퍼널 집계
select 'order' as stage, v.date,
    v.utm_source, v.utm_medium, v.utm_campaign, 
    count(distinct o.user_id) as values, sum(o.sales) as sales # 유저마다 재구매가 있을 수 있으므로 distinct로 중복없이 user_id를 카운트, sales값의 총합 집계
from `funnel-visualization.sample_data.visit_log` as v
        left join `funnel-visualization.sample_data.user` as u on v.cid = u.cid
        left join `funnel-visualization.sample_data.cert_log` as c on u.user_id = c.user_id
        left join `funnel-visualization.sample_data.order` as o on u.user_id = o.user_id
group by v.utm_source, v.utm_medium, v.utm_campaign, v.date
```



## 태블로로 대시보드 시각화

- 날짜 필터, utm 파라미터 필터 세팅
- 각 퍼널에서 얼마나 전환되는지 시각화하여 볼 수 있음(필터 변경시 조건에 맞게 바뀜)
- 필터링된 조건에 맞춰 유입, 회원가입, 매출량 TOP 콘텐츠 리스트를 볼 수 있음



## 후기

- 쿼리 더 간단하게 짤 수 있을 것 같았는데 계속 씨름하다가 결국 복잡하게 작성
  - '집계조건이 달라서'가 원인
  - SQL더 학습하자
- 유료버전을 사용하면 서버와 직접연결할 수 있다고 함. 일정주기로 데이터가 자동으로 업데이트되면 정말 유용할 듯.(만들어보고 싶다)

## 쿼리 수정
- 같은 테이블을 여러번 join으로 호출하니 깔끔해보이지 않아서 수정
- with 절 활용해서 joined라는 테이블 만들어 활용

```
WITH joined AS (
  select v.date, v.cid, v.utm_source, v.utm_medium, v.utm_campaign, u.user_id, c.cert_id, o.user_id as ordered_id, o.sales
  from `funnel-visualization.sample_data.visit_log` as v
        left join `funnel-visualization.sample_data.user` as u on v.cid = u.cid
        left join `funnel-visualization.sample_data.cert_log` as c on u.user_id = c.user_id
        left join `funnel-visualization.sample_data.order` as o on u.user_id = o.user_id 
)
select 'visit' as stage, j.date,
    j.utm_source, j.utm_medium, j.utm_campaign, 
    count(distinct j.cid) as values, 0 as sales
from joined as j
group by j.utm_source, j.utm_medium, j.utm_campaign, j.date
union all 
select 'sign in' as stage, j.date,
    j.utm_source, j.utm_medium, j.utm_campaign, 
    count(distinct j.user_id) as values, 0 as sales
from joined as j
group by j.utm_source, j.utm_medium, j.utm_campaign, j.date
union all 
select 'cert_phone' as stage, j.date,
    j.utm_source, j.utm_medium, j.utm_campaign, 
    count(distinct j.cert_id) as values, 0 as sales
from joined as j
group by j.utm_source, j.utm_medium, j.utm_campaign, j.date
union all 
select 'order' as stage, j.date,
    j.utm_source, j.utm_medium, j.utm_campaign, 
    count(distinct j.ordered_id) as values, sum(j.sales) as sales
from joined as j
group by j.utm_source, j.utm_medium, j.utm_campaign, j.date
```
