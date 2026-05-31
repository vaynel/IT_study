# 장애대응 / Failover

## 핵심 지표

| 지표 | 의미 | 목표 |
|------|------|------|
| **RTO** (Recovery Time Objective) | 장애 발생 후 복구까지 허용 시간 | 짧을수록 좋음 |
| **RPO** (Recovery Point Objective) | 복구 시 허용하는 데이터 손실 시간 | 짧을수록 좋음 |
| **MTTR** (Mean Time To Repair) | 평균 복구 시간 | 줄이는 것이 목표 |
| **MTBF** (Mean Time Between Failures) | 평균 장애 간격 | 늘리는 것이 목표 |

```
장애 발생        복구 완료
   │                │
   ├── RTO ─────────┤  (이 시간 안에 복구해야 함)
   │
   └── RPO ──┤      (이 시점 이전 데이터는 허용)
          최근 백업
```

---

## Failover (자동 전환)

장애 감지 시 **자동으로 트래픽을 정상 노드로 전환**하는 메커니즘.

### 헬스체크 기반 Failover

```
[LB] ──── Health Check ────→ [Server A]  ← 응답 없음
                              [Server B]  ← 정상

→ LB가 Server A를 풀에서 제거, Server B로만 라우팅
```

### DB Failover (Primary-Replica)

```
평상시:  Client → Primary (쓰기/읽기)
                     │ Replication
                  Replica (읽기 전용)

장애 시: Primary 다운 감지
         Replica → Primary로 승격 (자동 or 수동)
         Client → 새 Primary로 연결
```

---

## 헬스체크 설계

### 얕은 헬스체크 (Shallow)
- `/health` 엔드포인트가 200 응답만 확인
- 빠르지만 DB 연결 등 내부 문제는 감지 못 함

### 깊은 헬스체크 (Deep)
- DB 연결, 캐시 연결, 외부 API 등 **의존성까지 확인**
- 실제 서비스 가능 여부를 정확히 판단

```python
@app.get("/health")
def health():
    checks = {
        "db": check_db_connection(),
        "redis": check_redis_connection(),
    }
    status = "ok" if all(checks.values()) else "degraded"
    return {"status": status, "checks": checks}
```

---

## 장애 대응 프로세스

```
1. 감지 (Detection)
   └─ 모니터링 알림, 헬스체크 실패

2. 격리 (Isolation)
   └─ 장애 노드 트래픽 제거, 영향 범위 파악

3. 복구 (Recovery)
   └─ Failover 전환 or 재시작 or 롤백

4. 원인 분석 (Post-mortem)
   └─ 로그 분석, 재발 방지 대책 수립
```

---

## Circuit Breaker 패턴

외부 서비스 장애가 **전체 시스템으로 전파되는 것을 차단**하는 패턴.

```
상태:
CLOSED  → 정상 통신
OPEN    → 장애 감지, 요청 즉시 차단 (빠른 실패)
HALF-OPEN → 일정 시간 후 재시도, 성공하면 CLOSED로 복귀

예시:
결제 API가 5초 이상 응답 없음
→ Circuit OPEN
→ 이후 요청은 즉시 "결제 일시 중단" 반환 (타임아웃 대기 없음)
→ 30초 후 재시도 (HALF-OPEN)
```

---

## 장애 시나리오 대비 (Runbook)

| 장애 유형 | 감지 방법 | 대응 절차 |
|----------|----------|----------|
| 서버 다운 | 헬스체크 실패 | LB에서 제거 → 자동 재시작 |
| DB Primary 다운 | Replication 끊김 | Replica 승격 → 연결 문자열 변경 |
| 메모리 부족 | OOM 알림 | 프로세스 재시작 → 스케일 아웃 검토 |
| 디스크 풀 | 디스크 사용량 알림 | 로그 정리 → 볼륨 확장 |
