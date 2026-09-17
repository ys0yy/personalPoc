혼잡도 계산 코드 및 SQL
1. 혼잡도 계산 기준

빈틈에서는 사용자가 제보한 혼잡도 데이터를 이용하여 카페의 현재 혼잡도를 계산한다.

현재 혼잡도 계산에는 최근 60분 이내의 유효한 제보만 사용한다.

혼잡도는 다음 세 가지 상태로 구분한다.

여유
보통
혼잡

제보가 충분하지 않거나 같은 상태가 가장 많이 나온 경우에는 임의로 혼잡도를 정하지 않고 **정보 부족**으로 처리한다.

계산 기준
항목	기준
제보 시간	최근 60분 이내
최소 제보 수	2건
혼잡도 상태	여유 / 보통 / 혼잡
중복 제보	같은 사용자의 최근 제보 1건만 사용
계산 방법	각 상태별 제보 수 비교
최다 상태	해당 상태를 현재 혼잡도로 표시
동점	정보 부족
제보 부족	정보 부족
2. 혼잡도 계산 과정
제보 데이터 확인
      ↓
해당 카페의 제보만 확인
      ↓
최근 60분 이내 제보 필터링
      ↓
중복 제보 확인
      ↓
유효 제보가 2건 이상인가?
      ↓
   ┌── 아니오 ──→ 정보 부족
   │
   예
   ↓
여유 / 보통 / 혼잡별 개수 계산
      ↓
가장 많이 나온 상태 확인
      ↓
동점인가?
   ├── 예 → 정보 부족
   └── 아니오 → 해당 상태를 현재 혼잡도로 표시
3. Java 혼잡도 계산 코드

혼잡도 계산은 Java에서 CongestionCalculator 클래스로 구현하였다.

최근 60분 이내의 제보만 확인하고, 같은 사용자가 여러 번 제보한 경우 가장 최근 제보를 사용한다. 이후 여유, 보통, 혼잡의 개수를 비교하여 가장 많이 나온 상태를 현재 혼잡도로 결정한다.

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class CongestionCalculator {

    public CongestionResult calculate(
            Long cafeId,
            List<CongestionReport> reports,
            LocalDateTime now
    ) {

        Map<Long, CongestionReport> latestReports = new HashMap<>();

        // 해당 카페의 최근 제보만 확인
        for (CongestionReport report : reports) {

            if (!report.getCafeId().equals(cafeId)) {
                continue;
            }

            // 최근 60분 이내의 제보만 사용
            if (report.getReportedAt().isBefore(now.minusMinutes(60))) {
                continue;
            }

            // 같은 사용자의 제보는 가장 최근 제보만 사용
            CongestionReport previous =
                    latestReports.get(report.getUserId());

            if (previous == null ||
                    report.getReportedAt().isAfter(previous.getReportedAt())) {

                latestReports.put(report.getUserId(), report);
            }
        }

        // 최소 제보 수 확인
        if (latestReports.size() < 2) {
            return new CongestionResult(
                    CongestionStatus.INSUFFICIENT,
                    latestReports.size()
            );
        }

        int relaxed = 0;
        int normal = 0;
        int crowded = 0;

        // 혼잡도별 제보 개수 계산
        for (CongestionReport report : latestReports.values()) {

            switch (report.getStatus()) {
                case RELAXED -> relaxed++;
                case NORMAL -> normal++;
                case CROWDED -> crowded++;
            }
        }

        int max = Math.max(relaxed, Math.max(normal, crowded));

        int maxCount = 0;

        if (relaxed == max) maxCount++;
        if (normal == max) maxCount++;
        if (crowded == max) maxCount++;

        // 가장 많이 나온 상태가 2개 이상이면 동점
        if (maxCount > 1) {
            return new CongestionResult(
                    CongestionStatus.INSUFFICIENT,
                    latestReports.size()
            );
        }

        if (relaxed == max) {
            return new CongestionResult(
                    CongestionStatus.RELAXED,
                    latestReports.size()
            );
        }

        if (normal == max) {
            return new CongestionResult(
                    CongestionStatus.NORMAL,
                    latestReports.size()
            );
        }

        return new CongestionResult(
                CongestionStatus.CROWDED,
                latestReports.size()
        );
    }
}
4. SQL 혼잡도 계산

같은 혼잡도 계산을 SQL에서도 확인할 수 있도록 작성하였다.

먼저 최근 60분 이내의 제보만 조회하고, 같은 사용자가 여러 번 제보한 경우 가장 최근 제보만 남긴다.

그 후 혼잡도 상태별 제보 수를 계산한다.

WITH recent_reports AS (
    SELECT
        cafe_id,
        user_id,
        status,
        reported_at,
        ROW_NUMBER() OVER (
            PARTITION BY cafe_id, user_id
            ORDER BY reported_at DESC
        ) AS rn
    FROM congestion_report
    WHERE cafe_id = 1
      AND reported_at >= CURRENT_TIMESTAMP - INTERVAL '60 minutes'
),

valid_reports AS (
    SELECT
        cafe_id,
        user_id,
        status,
        reported_at
    FROM recent_reports
    WHERE rn = 1
),

status_count AS (
    SELECT
        cafe_id,
        status,
        COUNT(*) AS report_count
    FROM valid_reports
    GROUP BY cafe_id, status
),

max_count AS (
    SELECT
        cafe_id,
        MAX(report_count) AS max_report_count
    FROM status_count
    GROUP BY cafe_id
)

SELECT
    s.cafe_id,
    s.status,
    s.report_count
FROM status_count s
JOIN max_count m
    ON s.cafe_id = m.cafe_id
WHERE s.report_count = m.max_report_count;
최종 혼잡도 판단

SQL 조회 결과를 기준으로 최다 제보 상태가 하나이면 해당 상태를 현재 혼잡도로 사용한다.

최다 제보 상태가 여러 개이면 동점으로 판단하여 정보 부족으로 처리한다.

또한 최근 60분 이내의 유효한 제보가 **2건 미만인 경우에도 정보 부족**으로 처리한다.

5. 테스트 데이터

예를 들어 다음과 같은 제보가 들어왔다고 가정한다.

사용자	혼잡도	제보 시간
user1	여유	최근 10분
user2	보통	최근 20분
user3	여유	최근 30분
user4	혼잡	최근 40분

혼잡도별 개수는 다음과 같다.

여유 : 2
보통 : 1
혼잡 : 1

따라서 가장 많이 제보된 여유를 현재 혼잡도로 결정한다.

현재 혼잡도 : 여유
유효 제보 수 : 4건
6. 예외 상황 테스트
제보가 부족한 경우
여유 : 1
보통 : 0
혼잡 : 0

유효 제보가 2건보다 적기 때문에

현재 혼잡도 : 정보 부족

으로 처리한다.

혼잡도 제보가 동점인 경우
여유 : 2
보통 : 2
혼잡 : 1

여유와 보통의 제보 수가 같기 때문에 특정 상태를 선택하지 않고

현재 혼잡도 : 정보 부족

으로 처리한다.

오래된 제보가 있는 경우
최근 60분
여유 : 2

60분 이전
혼잡 : 5

60분 이전의 제보는 현재 혼잡도 계산에서 제외한다.

따라서 현재 혼잡도는

현재 혼잡도 : 여유

가 된다.

7. Java와 SQL 결과 비교

같은 테스트 데이터를 Java와 SQL에 적용하여 계산 결과를 비교하였다.

테스트 상황	Java 결과	SQL 결과
여유가 가장 많은 경우	여유	여유
보통이 가장 많은 경우	보통	보통
혼잡이 가장 많은 경우	혼잡	혼잡
제보 2건 미만	정보 부족	정보 부족
최다 제보 동점	정보 부족	정보 부족
60분 이전 제보만 존재	정보 부족	정보 부족
오래된 제보 제외 후 여유 최다	여유	여유

Java와 SQL에 동일한 데이터를 넣었을 때 같은 혼잡도 결과가 나오는지 확인하여 혼잡도 계산 기준이 동일하게 적용되는지 확인하였다.
