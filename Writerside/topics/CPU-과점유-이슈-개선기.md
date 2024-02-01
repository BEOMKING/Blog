# CPU 과점유 이슈 개선기

## 문제 상황과 원인

고객사 서버(B2B)에서 배치 업데이트 쿼리 스케줄러로 인해 과도한 CPU 점유가 다른 프로세스에 영향을 주는 상황이 발생했다.

스케줄러는 5분 주기로 반복되고 있었으며 쿼리는 거대한 조인과 업데이트를 수행하고 있었다.

서버를 확인하니 업데이트가 끝나지 않은 상태에서 또 다시 업데이트가 쌓이며 CPU 점유율이 1200%까지 올라가는 현상이 발생한 것을 확인할 수 있었다.

문제가 되는 테이블 스키마와 사용되는 쿼리를 비슷하게 재현해보면 다음과 같다.

**조회 대상 테이블**

```sql
CREATE TABLE document_log
(
    id             VARCHAR,
    transaction_id VARCHAR,
    partition_date TIMESTAMP,
    log_date       BIGINT,
    upload_status  SMALLINT,
    PRIMARY KEY (id, partition_date)
) PARTITION BY RANGE (partition_date);

CREATE TABLE file_log
(
    id             VARCHAR,
    transaction_id VARCHAR,
    partition_date TIMESTAMP,
    log_date       BIGINT,
    PRIMARY KEY (id, transaction_id, partition_date)
) PARTITION BY RANGE (partition_date);
```

기존 테이블 스키마는 위와 같다. 두 테이블 모두 레인지 파티션 테이블이며 파티션 키는 id, partition_date이다.

**업데이트 쿼리**

```sql
UPDATE document_log
SET upload_status = 1
WHERE upload_status != 1
  AND transaction_id IN
      (SELECT transaction_id
       FROM file_log
       WHERE process_time >= ?);
```

업데이트 쿼리는 위와 같다. 원본 쿼리는 더 복잡하지만 예시는 간단하게 작성했다.

**조회 쿼리**

```sql
SELECT *
FROM document_log
WHERE log_date BETWEEN ? AND ?
-- 조건 생략
ORDER BY log_date DESC
LIMIT 200 OFFSET 0;
```

업데이트된 데이터는 주로 위와 같은 조회 쿼리로 사용된다.

## 문제 해결 방안

맨 처음 들었던 생각은 그러면 업데이트 쿼리를 최적화하면 되지 않을까? 였다.

하지만 업데이트 대상이 필드는 이만큼의 리소스를 사용할 정도로 중요하지도 많이 사용되지도 않는 기능이라고 판단했고

근본적으로 부하가 큰 업데이트 쿼리와 짧은 스케줄링에 의한 지속적인 부하를 해결하는 것이 중요하다고 생각해 다른 방법을 찾아보기로 했다.

두 번째로 생각한 방법이 배치 업데이트 쿼리 스케줄링을 제거하고, 조회 요청을 날릴 때 해당 필드의 값을 가져올 수 있도록 조인하는 방식을 생각했다.

이 방법의 장점은 UI 작업이 필요하지 않고, 사용자는 기능의 변경없이 문제를 해결된다는 점, 부하를 주는 원인 자체가 사라진다는 점이고
    
단점은 조인에 의해 조회 성능이 느려질 수 있다는 점이었다.

따라서 내가 해결할 또 다른 문제는 조회 성능을 개선하는 것이었다.

## 문제 해결

조회 성능을 개선하기에 앞서 기존 조회 성능을 측정해보았다.

조회 쿼리 성능 테스트는 다음과 같은 환경에서 진행했으며 6월 데이터를 조회하는 쿼리를 측정했다.

6월 document_log 총 709만 개의 데이터, 검색 범위내 450만

6월 file_log 총 1231만 개의 데이터, 검색 범위내 1060만

검색 결과 데이터 150만개 중 200개를 조회하는 쿼리를 측정했다.

측정 결과 7000ms ~ 9000ms 정도의 시간이 걸렸다. (구체적인 조건문을 추가하면 더 빠른 결과를 얻을 수 있었다.)

아래는 쿼리 실행 계획이다.

```
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|QUERY PLAN                                                                                                                                                                                                                                  |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|Merge Append  (cost=2.05..2350673.04 rows=871108 width=7943) (actual time=3100.039..8492.949 rows=1506899 loops=1)                                                                                                                          |
|  Sort Key: document_log.log_date DESC                                                                                                                                                                                                      |
|  ->  Index Scan Backward using document_log_2023_01_idx1 on document_log_2023_01 document_log_1  (cost=0.12..8.15 rows=1 width=8657) (actual time=0.026..0.027 rows=0 loops=1)                       |
|        Index Cond: ((log_date < '1687618800000'::bigint) AND (log_date >= '1686754800000'::bigint))                                                                                                                     |
|  ->  Index Scan Backward using document_log_2023_02_idx1 on document_log_2023_02 document_log_2  (cost=0.12..5.90 rows=1 width=8657) (actual time=0.016..0.056 rows=0 loops=1)                       |
|        Index Cond: ((log_date < '1687618800000'::bigint) AND (log_date >= '1686754800000'::bigint))                                                                                                                     |
|  ->  Index Scan Backward using document_log_2023_03_idx1 on document_log_2023_03 document_log_3  (cost=0.12..7.58 rows=1 width=8657) (actual time=0.245..0.277 rows=0 loops=1)                       |
|        Index Cond: ((log_date < '1687618800000'::bigint) AND (log_date >= '1686754800000'::bigint))                                                                                                                     |
|  ->  Index Scan Backward using document_log_2023_04_idx1 on document_log_2023_04 document_log_4  (cost=0.12..8.15 rows=1 width=7975) (actual time=0.011..0.011 rows=0 loops=1)                       |
|        Index Cond: ((log_date < '1687618800000'::bigint) AND (log_date >= '1686754800000'::bigint))                                                                                                                     |
|  ->  Index Scan Backward using document_log_2023_05_idx1 on document_log_2023_05 document_log_5  (cost=0.43..8.46 rows=1 width=7943) (actual time=0.162..0.195 rows=0 loops=1)                       |
|        Index Cond: ((log_date < '1687618800000'::bigint) AND (log_date >= '1686754800000'::bigint))                                                                                                                     |
|  ->  Index Scan Backward using document_log_2023_06_idx1 on document_log_2023_06 document_log_6  (cost=0.43..2333195.91 rows=871101 width=7943) (actual time=3099.546..8408.109 rows=1506899 loops=1)|
|        Index Cond: ((log_date < '1687618800000'::bigint) AND (log_date >= '1686754800000'::bigint) AND (detection_log_show = 1))                                                                                        |
|  ->  Index Scan Backward using document_log_2023_07_idx1 on document_log_2023_07 document_log_7  (cost=0.43..8.46 rows=1 width=7943) (actual time=0.022..0.022 rows=0 loops=1)                       |
|        Index Cond: ((log_date < '1687618800000'::bigint) AND (log_date >= '1686754800000'::bigint))                                                                                                                     |
|  ->  Index Scan Backward using document_log_2023_08_idx1 on document_log_2023_08 document_log_8  (cost=0.12..8.15 rows=1 width=9102) (actual time=0.003..0.003 rows=0 loops=1)                       |
|        Index Cond: ((log_date < '1687618800000'::bigint) AND (log_date >= '1686754800000'::bigint))                                                                                                                     |
|Planning Time: 49.234 ms                                                                                                                                                                                                                    |
|JIT:                                                                                                                                                                                                                                        |
|  Functions: 46                                                                                                                                                                                                                             |
|  Options: Inlining true, Optimization true, Expressions true, Deforming true                                                                                                                                                               |
|  Timing: Generation 55.231 ms, Inlining 14.091 ms, Optimization 1670.979 ms, Emission 1406.869 ms, Total 3147.170 ms                                                                                                                       |
|Execution Time: 8591.004 ms                                                                                                                                                                                                                 |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

실행 계획을 보기 전에 조회 쿼리의 문제점을 눈치챘을 수도 있겠지만(기존 문제가 되는 업데이트 쿼리도 마찬가지) 조회 조건으로 파티션 키를 사용하지 않고 있다.

조회 조건으로는 long 타입의 log_date를 사용하고 있으며, 이는 파티션 키인 partition_date와는 다르다.

파티션 키를 사용하지 않으면 파티션 테이블의 장점을 활용하지 못하고 오히려 성능이 떨어질 수 있다.

실행 계획을 보면 document_log 테이블의 모든 파티션을 조회하고 있는 것을 확인할 수 있다.

따라서 내가 해야할 일은 조회 쿼리에 조인 쿼리를 추가하는 것, 조인 대상이 되는 두 테이블 모두 파티션 키를 사용할 수 있도록 해주는 것이었다.

다행히 파티션 키와 별개로 검색 조건, 조인에 사용되는 인덱스는 잘 타는 것을 확인했다.

여러 시행착오 끝에 최종적으로 다음과 같은 쿼리를 만들었다.

```sql
SELECT *
FROM (SELECT *
      FROM document_log
      WHERE partition_date BETWEEN '2023-06-15' AND '2023-06-25'
        AND log_date BETWEEN 1686754800000 AND 1687618800000) AS dl
         JOIN file_log fl
              ON fl.partition_date BETWEEN '2023-06-15' AND '2023-06-25' AND
                 fl.transation_id = dl.transation_id
GROUP BY * -- 생략
ORDER BY log_date DESC
LIMIT 200 OFFSET 0;
```

위와 같이 프론트에서 들어오는 log_date 파리미터를 이용해 partition_date를 계산하여 파티션 키를 사용할 수 있도록 했다.

파티션 키를 사용하니 조인을 하더라도 성능은 200ms ~ 300ms 정도로 개선되어 기존 방식보다 훨씬 빠른 속도를 보여주었다.

> 물론 불필요한 변환을 하지 말고 파티션 키를 log_date로 바꾸면 해결되는 문제가 아닌가? 라는 의문이 들 수 있다. 
> 하지만 기존에 사용하는 모든 고객사 DB 스키마를 변경해야 하기 때문에 이 방법은 선택하지 않고 추후 개선 사항으로 남겨두었다.

## 마무리

막상 조회 쿼리를 수정하는 부분은 상대적으로 간단했지만 문제에 대한 원인과 해결 방안을 찾는 과정에서 많은 생각을 하고 배울 수 있었다.

또한, 고객사와 우리의 상황을 고려하여 해결 방법을 선택하는 과정 또한 많은 경험이 되었다.
