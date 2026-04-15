# StockRedo v1.0.0
> 한국 주식 가상거래 시뮬레이션 플랫폼 <br>
> https://stockredo.duckdns.org

---

## 🏗️ 인프라 & 데이터 파이프라인

### 핵심 서버 스택
<img width="1376" height="768" alt="아키텍" src="https://github.com/user-attachments/assets/56f269ae-5609-47c8-9171-4e8da9a72358" />


### 실시간 가격 수집
- `collector_redis.py` → 평일 08:50~15:40, 약 5초마다 Redis 갱신
- `stock:{code}:price` / `stock:{code}:prev_close` / `stock:{code}:time`

### 데이터 흐름
```
Redis (실시간) ──5분──▶ stock_prices DB (sync_redis_to_db.php)
                                │
                                └──1분──▶ stock_history DB (archiver.py)
                                               │
                                         7일 후 자동 삭제 (MySQL Event Scheduler)
```

### Crontab
| 시각 | 작업 |
|------|------|
| 평일 08:55 | `stock_to_db.py --prices-only` — 매매 기준가 확정 |
| 평일 09:00~18:01 매분 | `archiver.py` — 차트용 1분 스냅샷 저장 |
| 평일 18:02 | `stock_to_db.py` — 재무지표 풀 동기화 (~2분) |
| 평일 18:07 | `save_daily_ranking.php` — 리더보드 순위 박제 |
| 매일 00:00 | 게스트 계정 데이터 자동 삭제 |
| 매주 일요일 03:00 | `update_sectors.py` — 섹터 분류 업데이트 |
| 평일 장중 5분마다 | `sync_redis_to_db.php` — Redis → DB 가격 동기화 |

---

## 📈 주요 기능

### 거래
- 국내 전 종목 가상 매수/매도 (ETF 제외)
- 슬리피지: 한국 호가단위 기준 1틱 / 수수료: 0.015%
- 체결 미리보기 (매수/매도 예상가·수수료·총액)
- 슬라이더 + 수량 직접 입력 동기화

### 차트
- 라인 / 캔들 차트 전환
- 이동평균선 MA5 / MA20 / MA60 / MA120
- 1M / 5M / 30M / 1H / 1D / ALL 범위 선택
- 십자선 (수직+수평) 커서

### 포트폴리오
- 보유 종목 목록 (수량·평균단가·투자금액·수익률)
- 자산 배분 도넛 차트
- 주식 평가 수익률 실시간 갱신

### 트렌드
- 급등 / 급락 / 활발 — Redis 실시간 (1분 단위)
- 갱신 시간 초단위 표시 + 1분마다 자동 갱신
- 티커 배너 (상단 스크롤)

### AI 추천 (STOCKREDO_AI)
- 퀀트 멀티팩터 Z-score 스코어링
  - Value 20% / Quality 25% / Growth 20% / Momentum 25% / Liquidity 10%
- 섹터 중립화 (섹터별 최대 3종목)
- 뉴스 감성 분석 연계
- 노드 그래프 분석 애니메이션
- 하루 3회 자동 갱신 (09:00 / 12:00 / 15:00)

### AI 챗봇
- claude-haiku 기반 투자 어시스턴트
- 포트폴리오·섹터·뉴스 컨텍스트 자동 주입
- 무료 월 10회 / PRO 월 500회
- 대화 히스토리 세션 유지

### 계정
- 구글 / 카카오 소셜 로그인
- 이메일 회원가입 + 인증
- 게스트 로그인 (매일 00:00 자동 삭제)
- 닉네임 변경 (게스트 불가)
- 자산 리셋 (시스템 설정)

### 랭킹
- 수익률 / 총자산 기준 리더보드
- 매일 18:07 순위 박제

### 뉴스
- 연합뉴스 + Google News RSS
- 강세 / 약세 / 중립 감성 분류
- 보유 종목 관련 뉴스 우선 표시

---

## 💰 운영 비용 (월)
| 항목 | 비용 |
|------|------|
| AI 추천 (66회/월) | $0.31 |
| AI 챗봇 (유저 수 비례) | ~$0.40 / 10명 |
| **합계** | **~$0.71~** |

---

## 🗂️ 주요 파일 구조
``` 
/var/www/html/ (총 46개)
├── index.php              # 메인 앱
├── index_script.js        # 프론트엔드 로직
├── index_style.css        # 스타일
├── get_trend.php          # 트렌드 API (Redis 실시간)
├── trend_ai_worker.php    # AI 추천 워커
├── chat_ai.php            # AI 챗봇
├── get_all_prices.php     # 실시간 가격 API
├── db_connect.php         # DB 연결
└── components/            # 차트 등 컴포넌트

/home/ubuntu/ (총 26개)
├── stock_to_db.py         # 전종목 재무지표 수집
├── archiver.py            # 1분 스냅샷
├── sync_redis_to_db.php   # Redis→DB 동기화
├── update_sectors.py      # 섹터 분류
└── collector_redis.py     # 실시간 가격 수집
```
