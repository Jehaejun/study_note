# Bulk Upsert

PostgreSQL 9.5에서 ON CONFLICT 가 도입된 후 Upsert (Insert or Update) 문을 쓸수 있게 되었지만, 복수행을 bulk로 보내는건 불가했습니다.

ON CONFLICT 를 사용하지 않아도, CTE 로 로직을 만들어 Bulk Upsert 와 같은 결과를 내는 것이 가능합니다.

```SQL
WITH
-- write the new values
n(ip,visits,clicks) AS (
  VALUES ('192.168.1.1',2,12),
         ('192.168.1.2',6,18),
         ('192.168.1.3',3,4)
),
-- update existing rows
upsert AS (
  UPDATE page_views o
  SET visits=n.visits, clicks=n.clicks
  FROM n WHERE o.ip = n.ip
  RETURNING o.ip
)
-- insert missing rows
INSERT INTO page_views (ip,visits,clicks)
SELECT n.ip, n.visits, n.clicks FROM n
WHERE n.ip NOT IN (
  SELECT ip FROM upsert
)
```

위 sql 출처 (Faster data updates with CartoDB — CARTO Blog)

**주의점**

select문과 update, insert 문과의 차이점은 select문은 복수행을 한번에 취득한다는 점이고, update, insert 문은 row by row로 결과를 취득한다는 것입니다. 때문에 위의 UPDATE문에서 n테이블을 사용했을 때, 마치 for loop 도는 것과 같은 액션이 가능합니다.

**예시)**

```sql
WITH P AS (
   SELECT 'D' AS PROD_GRP, PROD_CD, BIZ_NO, BANK_PROD_CD FROM PROD_LDGR
),
UPSERT AS (
   UPDATE VOUC_PROD_LDGR_2021 as PL
      SET BANK_PROD_CD=P.BANK_PROD_CD
     FROM P
    WHERE PL.PROD_GRP=P.PROD_GRP AND PL.PROD_CD=P.PROD_CD AND PL.BIZ_NO=P.BIZ_NO
RETURNING *
)
INSERT INTO VOUC_PROD_LDGR_2021 (PROD_GRP, PROD_CD, BIZ_NO, BANK_PROD_CD, reg_dtm, regr_id)
SELECT P.PROD_GRP, P.PROD_CD, P.BIZ_NO, P.BANK_PROD_CD,  TO_CHAR(NOW(), 'YYYYMMDDHH24MISS'), 'test_ttttt'
  FROM P
 WHERE NOT EXISTS (SELECT * FROM UPSERT)
```

해당 쿼리 실행시 다음과 같은 오류를 접하였습니다.

```
SQL Error [XX000]: ERROR: failed to find conversion function from unknown to text
```

문자열 리터럴의 (아직 알려지지 않은) 데이터 유형을 선언하기 위해 명시적 캐스트가 필요함을 의미합니다.

https://stackoverflow.com/questions/18073901/failed-to-find-conversion-function-from-unknown-to-text



**해결)**

```SQL
WITH P AS (
   SELECT 'D'::text AS PROD_GRP, PROD_CD::text, BIZ_NO::text, BANK_PROD_CD::text FROM PROD_LDGR
),
UPSERT AS (
   UPDATE VOUC_PROD_LDGR_2021 as PL
      SET BANK_PROD_CD=P.BANK_PROD_CD
     FROM P
    WHERE PL.PROD_GRP=P.PROD_GRP AND PL.PROD_CD=P.PROD_CD AND PL.BIZ_NO=P.BIZ_NO
RETURNING *
)
INSERT INTO VOUC_PROD_LDGR_2021 (PROD_GRP, PROD_CD, BIZ_NO, BANK_PROD_CD, reg_dtm, regr_id)
SELECT P.PROD_GRP, P.PROD_CD, P.BIZ_NO, P.BANK_PROD_CD,  TO_CHAR(NOW(), 'YYYYMMDDHH24MISS'), 'test_ttttt'
  FROM P
 WHERE NOT EXISTS (SELECT * FROM UPSERT)
```

