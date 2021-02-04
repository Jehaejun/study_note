# postgresql 날짜분기

```sql
select extract(quarter from to_date('202007', 'YYYYMM'))
```

| date_part |
| --------- |
| 3.0       |



**예시)**

```sql
SELECT T.quarter, T.TRSC_AMT, T.SUM_TRSC_AMT, T.GB
FROM (
        SELECT C.quarter
             , COALESCE(SUM(C.SALE_AMT),0) AS TRSC_AMT
             , SUM(COALESCE(SUM(C.SALE_AMT),0)) OVER() SUM_TRSC_AMT
             , 'S' AS GB
        FROM (
                SELECT A.TRSC_YM, SUBSTRING(TRSC_YM, 1, 4) || '_' || extract(quarter from to_date(A.TRSC_YM, 'YYYYMM')) AS quarter, A.SALE_AMT
                  FROM SELR_MNLY_SUMR A JOIN RPRT_BZAQ_INFM B ON B.USE_INTT_ID  = A.USE_INTT_ID
                                                              AND B.BZAQ_KEY    = A.BZAQ_KEY::TEXT
                                                              AND B.BZAQ_GRP_CD = 'S'
                 WHERE A.USE_INTT_ID = 'UTLZ_1710311402890'
                   AND A.TRSC_YM BETWEEN '201902' AND '201907'
                   AND A.SALE_CNT > 0
        )C
        GROUP BY C.quarter
        UNION
        SELECT C.quarter
             , COALESCE(SUM(C.BUY_AMT),0) AS TRSC_AMT
             , SUM(COALESCE(SUM(C.BUY_AMT),0)) OVER() SUM_TRSC_AMT
             , 'B' AS GB
        FROM (
                SELECT A.TRSC_YM, SUBSTRING(TRSC_YM, 1, 4) || '_' || extract(quarter from to_date(A.TRSC_YM, 'YYYYMM')) AS quarter, A.BUY_AMT
                  FROM BUYR_MNLY_SUMR A JOIN RPRT_BZAQ_INFM B ON B.USE_INTT_ID  = A.USE_INTT_ID
                                                              AND B.BZAQ_KEY    = A.BZAQ_KEY::TEXT
                                                              AND B.BZAQ_GRP_CD = 'S'
                 WHERE A.USE_INTT_ID = 'UTLZ_1710311402890'
                   AND A.TRSC_YM BETWEEN '201902' AND '201907'
                   AND A.BUY_CNT > 0
        )C
        GROUP BY C.quarter
) T
```

