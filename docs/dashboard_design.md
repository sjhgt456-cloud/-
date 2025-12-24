# API 기반 대시보드 설계

날씨, 환율, 주식, 코인 데이터를 한 화면에서 제공하는 실시간 대시보드를 구축하기 위한 설계 문서다. 데이터 신뢰성, 응답 속도, 가시성을 최우선으로 하고, API 요청 비용을 줄이기 위해 캐싱과 백그라운드 수집 작업을 병행한다.

## 목표
- 한 화면에서 주요 지표(날씨/환율/주식/코인)의 현재 값과 추세를 확인
- 다양한 외부 API를 안전하게 연동하고, 과금/쿨다운 정책을 준수
- 장애 상황에서도 최근 캐시 데이터를 이용한 읽기 서비스 지속

## 기술 스택 제안
- **백엔드**: FastAPI + HTTPX (비동기 수집) + Redis(캐시/락) + PostgreSQL(히스토리)
- **잡 스케줄러**: APScheduler(서비스 내) 혹은 Celery Beat + Worker
- **프런트엔드**: React(Vite) + Chart.js/Recharts + TailwindCSS
- **인프라**: Docker Compose(개발), Kubernetes(HPA + CronJob) or Serverless(람다 + EventBridge)

## 데이터 소스 예시
- **날씨**: Open-Meteo(무료, 키 불필요) / OpenWeather(유료 구간 포함)
- **환율**: exchangerate.host(무료) / 외부 은행 API(국가별)
- **주식**: Alpha Vantage, IEX Cloud, Finnhub (키 필요, 호출 제한 주의)
- **코인**: CoinGecko(무료, rate limit 있음) / Binance Public API

각 소스는 `.env`로 키를 주입하고, 동일 자산군에 대해 추상화된 어댑터를 둔다.

## 아키텍처
1. **Ingestion Worker**: 외부 API 호출, 응답 정규화, Redis & DB 저장.
2. **Aggregation API**: 캐시 우선 조회 후 백필(fetch-on-stale) 옵션.
3. **프런트엔드**: `/api/summary`에서 모든 위젯 데이터를 일괄 받아 렌더.

```text
[External APIs] --> [Ingestion Worker] --> [Redis Cache] --> [API Server] --> [Web UI]
                                           \                         /
                                            ------> [PostgreSQL] ----
```

## 데이터 모델 (정규화된 공용 스키마)
- **WeatherSnapshot**: `location`, `temp_c`, `humidity`, `wind_kph`, `condition`, `observed_at`
- **FxRate**: `base`, `quote`, `rate`, `as_of`
- **MarketPrice**: `symbol`, `asset_type`(`stock|crypto`), `price`, `change_pct`, `volume`, `as_of`

### 통합 응답 예시: `/api/summary`
```json
{
  "weather": {
    "location": "Seoul",
    "temp_c": 21.3,
    "humidity": 55,
    "wind_kph": 12,
    "condition": "Cloudy",
    "observed_at": "2024-05-01T02:00:00Z"
  },
  "fx": [
    { "base": "USD", "quote": "KRW", "rate": 1380.12, "as_of": "2024-05-01T02:00:00Z" },
    { "base": "USD", "quote": "JPY", "rate": 153.25, "as_of": "2024-05-01T02:00:00Z" }
  ],
  "markets": [
    { "symbol": "AAPL", "asset_type": "stock", "price": 170.12, "change_pct": 0.42, "volume": 34000000, "as_of": "2024-05-01T01:59:00Z" },
    { "symbol": "BTC", "asset_type": "crypto", "price": 62000.5, "change_pct": -0.15, "volume": 1200, "as_of": "2024-05-01T01:59:00Z" }
  ]
}
```

## API 엔드포인트 설계
- `GET /api/summary?symbols=AAPL,BTC&fx=USD/KRW,USD/JPY` : 위젯 한 번에 조회, 기본 캐시 TTL 60s.
- `GET /api/weather?location=Seoul` : 위치별 상세 날씨.
- `GET /api/fx?base=USD&quotes=KRW,JPY` : 원하는 통화쌍만 조회.
- `GET /api/markets?symbols=AAPL,TSLA,BTC,ETH` : 주식/코인 혼합 조회.
- `GET /api/health` : 백엔드·데이터소스 헬스 체크.

### 실패 및 폴백 전략
- 각 어댑터 별 `rate-limit` 감지 → 백오프 및 캐시 반환.
- 데이터 누락 시: (1) 마지막 성공 스냅샷 제공, (2) `stale` 플래그로 UI 표시.
- 동일 자산군 동시 호출 시 Redis 분산 락으로 중복 호출 방지.

## 업데이트 주기 제안
- 날씨: 10~15분
- 환율: 5분
- 주식: 장중 30~60초, 장외 5분
- 코인: 15~30초 (API 제한 범위 내)

## 프런트엔드 레이아웃 초안
- **헤더**: 지역/통화 선택, 마지막 업데이트 시각, 새로고침 버튼
- **좌측 컬럼**: 날씨 카드 + 지도/아이콘
- **중앙 그리드**: 환율 테이블, 주식/코인 가격 카드, 차트(24h/1w)
- **우측 패널**: 알림(급등락), 즐겨찾기 심볼 설정

컴포넌트는 `loading`, `stale`, `error` 상태를 명시적으로 표현한다.

## 로컬 개발 절차 예시
1. `cp .env.example .env` 후 API 키 설정
2. `docker compose up -d redis db`로 의존 서비스 실행
3. `uvicorn app.main:app --reload` 또는 `npm run dev`(Vite)으로 클라이언트/서버 실행
4. `pytest` 및 `npm test`로 단위 테스트 수행

## 보안/운영 체크리스트
- 모든 외부 호출에 `timeout`/`retry`/`circuit breaker` 적용 (예: httpx + tenacity)
- 비밀키는 Secret Manager/Parameter Store 활용, 깃에 커밋 금지
- 관측성: OpenTelemetry 트레이싱 + Prometheus 메트릭 + 구조화 로그(JSON)
- 알림: 실패율 상승, 캐시 미스 급증, 스케줄 지연 시 슬랙/이메일 알림
- GDPR/개인정보 민감도 낮음이지만 위치 기반 날씨 조회 시 위치 정보 최소 보관
