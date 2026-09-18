# kbrain-cert — 100명 동시 응시 실행 계획 (Capacity Plan)

**최종 갱신**: 2026-07-14 (Agora 확정 반영)

**목표**
- 응시자 100명이 동시 응시해도 **답안 유실 0건**
- **응답 p95 < 500ms** (문제 페치 · 답안 저장 · 서버 시간 동기)
- 감독 이벤트 유실률 < 0.1%
- 녹화 청크 유실률 < 1% (재시도 포함)
- 시험 종료 순간 최종 제출 폭주(100명 동시) 안전 처리

**규모 가정**: 100명 · 120분 · **회당** 계산 (월 시험 횟수는 담당자 사업에 따라 유동)

**미결정 항목은 §7 참고** (계정 발급 · 계약 필요).

---

## 1. 서비스별 실제 티어 · 예상 비용

### 1.1 Supabase (DB · Auth · Storage · Realtime)

| 항목 | Pro Plan ($25/월) | kbrain-cert 사용량 | 판정 |
|---|---|---|---|
| Direct connections | 200 (기본) | 100명 응시 시 커넥션 100 필요 | 여유. PgBouncer transaction mode 안전용으로만 |
| Realtime concurrent | 500 채널 | 응시자 outbound POST만 · 관리자 폴링 → 채널 <20 | 여유 |
| DB storage | 8 GB | 문제·답안·이벤트 → 회당 ~200MB | 여유 |
| DB egress | 250 GB | 회당 ~2GB | 여유 |
| Storage (파일) | 100 GB | 신분증 30일 보관 · 슬롯 파일 → 회당 5~10GB | 여유 |
| **월 비용** | | | **$25 (약 3.5만원) 고정** |

**Pro 유지 이유**: 자동 백업(PITR) · 안전한 커넥션 풀 · 공식 자격증 서비스 안정성.

### 1.2 Agora (감독관 실시간 화상 관찰) — Daily.co 대체

**공식 요금** (agora.io/en/pricing/video-calling, 2026-07 확인):

| 항목 | 요금 (per 1,000 참여자·분) |
|---|---|
| Audio only | $0.99 |
| **Video HD (720p)** — kbrain-cert 해당 | **$3.99** |
| Video Full HD (1080p) | $8.99 |
| Video 2K | $15.99 |
| SD (480p 이하) | 페이지 미명시 · **실질 HD로 카운트** |
| Cloud Recording | 별도 · **미사용** (우리는 R2 자체 녹화) |
| Cloud Proxy (UDP 차단 회사 네트워크용) | 월 $500 base fee · **초기 미사용** |

**Free tier**: **10,000분/월** (초과 시 서비스 중단 → 카드 등록으로 자동 결제 필요)

**Flexible billing**: pay-as-you-go 기본 (사용량 만큼 카드 결제) + Cloud Proxy 옵션.

**계산 (Video HD 720p 기준)**:

| 항목 | 계산 | 회당 |
|---|---|---|
| 응시자 100명 × 120분 | 12,000참여자·분 × $3.99/1000 | $47.88 |
| 감독관 2명 × 120분 | 240참여자·분 × $3.99/1000 | $0.96 |
| **회당 총합** | | **약 $48.84 = 약 7만원 / 회** |
| **첫 회 (Free tier 활용)** | 12,240 − 10,000 = 초과 2,240분 × $3.99/1000 | **약 $8.94 = 약 1.3만원** |

⚠️ **Screen sharing 카운트 방식 검증 필요** — Agora 공식 페이지에 명시 없음. 만약 웹캠·화면공유 트랙이 각각 카운트되면 요금 2배 (**회당 약 14만원**). Agora sales에 문의 or 실제 사용해 계산서 확인 필요.

**Agora 선택 이유**:
- Daily.co Prime 대비 여전히 저렴 (Daily 10만원 → Agora 7만원, 30% 절감)
- Web SDK(`agora-rtc-sdk-ng`) 성숙 · React 예제 많음
- 한국 리전 지원 (Seoul edge)
- **Free 10,000분/월** → 초기 개발·리허설·첫 회 실운영 부담 크게 줄임

**적용 방법**:
- agora.io Console에서 프로젝트 생성 → App ID + Certificate 발급
- Next.js Server Action에서 RTC 토큰 서명 (App Certificate로)
- 응시자·감독관은 client에서 App ID + 토큰으로 채널 join

**주의**:
- Agora 채팅은 Signaling SDK 별도(RTM). 담당자 감독관↔응시자 채팅은 **Supabase Realtime으로 자체 구현** (Agora RTM 비용 절감)
- 녹화는 Agora Cloud Recording 사용 시 별도 요금 → **kbrain-cert는 클라 MediaRecorder → R2 방식 유지**

### 1.3 Cloudflare R2 (응시 녹화)

| 항목 | 요금 | 계산 (100명·120분·1회) | 회당/월 비용 |
|---|---|---|---|
| 저장 (Standard) | $0.015/GB/월 | 100 × 120분 × 1Mbps(webcam+screen 합) = **90GB · 30일 보관** | **월 $1.35 (약 2천원)** |
| Class A writes (500ms 청크) | $4.50/M | 100 × 2트랙 × 120분 × 120chunks/분 = **2.88M writes** | **회당 $13 (약 2만원)** ⚠️ |
| **Class A writes (5초 청크 · 권장)** | 위와 동일 | writes 1/10 | **회당 $1.30 (약 2천원)** |
| Class B reads | $0.36/M | 관리자·채점자 재생 시. 소량 | 예산 내 |
| Egress | **무료** | S3 대비 큰 이점 | $0 |
| **R2 회당 총합 (5s 청크)** | | | **약 4천원 / 회** (30일 보관) |
| **R2 회당 총합 (500ms 청크)** | | | 약 2.2만원 / 회 |

**결정 필요**: 청크 길이 500ms(실시간 재검토 유리 · 요금 10배) vs 5초(비용·병목 저감 · 실시간성 다소 손실). **kbrain-cert는 사후 검토 위주라 5초 청크 권장.**

### 1.4 Resend (초대·OTP 이메일)

| 플랜 | 월 요금 | 발송 한도 | kbrain-cert 사용량 |
|---|---|---|---|
| **Free** — 권장 | $0 | 3,000통/월 · 도메인 1개 | 100명 × (초대 1 + OTP 1~3 + 결과 알림 1) = **300~500통/회** → Free로 커버 |
| Pro | $20/월 | 50,000통 | 월 여러 회 시험 시 |

**월 시험 1회면 Free plan으로 무료 운영 가능.**

### 1.5 Vercel (프론트엔드)

| 플랜 | 월 요금 |
|---|---|
| Hobby (개인용) | $0 (하지만 **상업 라이선스 위반**) |
| **Pro** | $20/월 (약 3만원) 필수 |

**공식 자격증 시험 서비스는 Vercel Pro 필수.**

### 1.6 월/회당 총 예상

| 항목 | 회당 변동비 | 월 고정비 |
|---|---|---|
| Supabase Pro | — | 3.5만원 |
| **Agora Video HD** | **약 7만원** (Free 소진 후) · 첫 회 약 1.3만원 | — |
| Cloudflare R2 (5s 청크) | 약 4천원 | — |
| Resend Free | 0원 | 0원 |
| Vercel Pro | — | 3만원 |
| **합계** | **약 7.4만원 / 회** | **약 6.5만원 / 월** |

**월 1회 시험 기준**: **회당 총 약 14만원** · 응시자 1인당 **약 1,400원**  
**월 2회 시험 기준**: **회당 총 약 10.7만원** · 응시자 1인당 **약 1,070원**  
**월 4회 시험 기준**: **회당 총 약 9만원** · 응시자 1인당 **약 900원**  
**첫 회 (Free tier 활용)**: **약 8만원**

⚠️ **screen sharing 이중 카운트 시 최악 시나리오**: 회당 21만원 · 1인당 약 2,100원 (Agora sales에 카운트 방식 확인 필수)

---

## 2. 라이브러리 선정 · 성능 벤치마크 목표

### 2.1 프론트엔드 핵심

| 라이브러리 | 버전 | 역할 | 성능 목표 |
|---|---|---|---|
| Next.js | **16.2.10** (설치됨) | App Router · Server Components | TTFB < 300ms · LCP < 2s |
| React | 19.2.4 | UI | — |
| Tailwind CSS | v4 | 스타일 (@theme inline) | 초기 CSS < 30KB gzip |
| TanStack Query | v5 | 서버 상태 · 답안 저장 mutation | 저장 재시도 3회 · 500ms 백오프 |
| Supabase JS | v2 최신 | DB · Auth · Realtime · Storage | 커넥션 pool 자동 |
| face-api.js | 0.22.2 (원본과 동일) | 얼굴 감지 로컬 추론 | **CPU < 15% 지속** · 2.5s 간격 감지 |
| **agora-rtc-sdk-ng** | 최신 | WebRTC 스트림 (Daily.co 대체) | 스트림 초기화 < 3s · Seoul 리전 |
| MediaRecorder API | native | R2 녹화 청크 생성 | 5초 청크당 <5MB webm |

### 2.2 백엔드 · 인프라

| 항목 | 선택 | 이유 |
|---|---|---|
| DB | Postgres (Supabase) | RLS · JSONB · 파티셔닝 |
| 커넥션 풀 | Supabase Pooler (PgBouncer) transaction mode | 100 클라이언트 커넥션도 안전하게 압축 |
| Realtime | Supabase Realtime (관리자만 · 응시자 outbound POST) | 채널 요금 회피 |
| 이벤트 배치 | 클라 5s 윈도우 → bulk insert | 100명 × 초당 2 = 200/s peak → 5s 배치 → 40/s |
| 감독 이벤트 파티션 | monthly + session_id hash | 조회 성능 · 사후 삭제 편의 |
| 파일 업로드 | R2 Presigned URL 직접 업로드 | 서버 개입 최소 |
| **감독관↔응시자 채팅** | Supabase Realtime (Agora RTM 대신 자체 구현) | Agora 채팅 비용 절감 |

### 2.3 성능 벤치마크 목표 (M6 부하검증에서 측정)

| 지표 | 목표 |
|---|---|
| 문제 페치 (시험 시작 순간) | p95 < 400ms · 100 concurrent |
| 답안 저장 (upsert) | p95 < 200ms |
| 감독 이벤트 bulk insert | p95 < 300ms · 40건/배치 |
| 최종 제출 (Server Action, idempotent) | p95 < 500ms · 100 동시 |
| R2 청크 upload 성공률 | > 99% · 3회 재시도 포함 |
| Agora SFU 연결 성공률 | > 98% |
| face-api 감지 정확도 | 미검출 감지율 > 90% · 오탐 < 5% |
| 브라우저 CPU (감독 + 녹화 + WebRTC 동시) | 지속 < 40% · 피크 < 70% |
| 브라우저 메모리 | 지속 < 500MB · 피크 < 1GB |

---

## 3. 병목별 상세 대응 (각 병목마다 · 원인 · 대응 · 검증 · 잔여 리스크)

### 3.1 시험 시작 순간 문제 페치 폭주

- **원인**: 100명이 정각에 응시 시작 → 문제 payload 100 요청 폭주 (동일 payload)
- **대응**:
  - Vercel Edge Cache로 문제 payload 캐싱 (`unstable_cache` + tag revalidation)
  - Supabase는 서버 컴포넌트에서 뷰 조회 → 캐시 히트 시 DB 요청 없음
  - 응시자별 시작 시각 랜덤 지연 ±3s 옵션 (thundering herd 완화)
- **검증**: Playwright + Artillery로 초당 100 페치 시뮬레이션 · p95 < 400ms 목표
- **잔여 리스크**: 캐시 무효화 타이밍 (관리자가 시험 시작 직전 문제 수정)

### 3.2 답안 저장 지속 write

- **원인**: 100명 × 문제당 debounce 3s → 지속적 upsert 트래픽
- **대응**:
  - PgBouncer transaction mode (물리 커넥션 압축, 안전용)
  - 응답 IndexedDB 로컬 백업 → 네트워크 순단 시 대기 후 flush
  - Server Action idempotent (session_id + question_id 유니크 upsert)
- **검증**: 부하 스크립트로 100 세션 × 5분간 지속 저장 · 유실 0 확인
- **잔여 리스크**: 브라우저 오프라인 시간이 길면 IndexedDB 데이터 손실

### 3.3 감독 이벤트 폭주

- **원인**: 100명 × 초당 2 이벤트 = **200 writes/s peak**
- **대응**:
  - **클라에서 5s 윈도우 배치** → 초당 20~40 bulk insert만
  - 이벤트 테이블 파티셔닝 (month + session_id)
  - Realtime inbound는 관리자만 (그리드 진입 시)
- **검증**: 100 명 × 5분 시뮬 · 이벤트 유실률 < 0.1%

### 3.4 Agora SFU 스트림

- **원인**: 감독관 그리드에서 20~50 명 receive 시 대역폭
- **대응**:
  - Agora의 **dual-stream mode (simulcast)** 활성화 → 저해상도 SD 스트림 요청
  - 이벤트 발생 응시자만 HD로 확대 (감독관 UI에서 자동 전환)
  - 감독관 2명 이내 · 각자 50명 담당 (100명 규모)
- **검증**: 실기기 리허설 (Playwright는 WebRTC 미지원)
- **잔여 리스크**: 감독관 노트북 사양 · 회사 네트워크

### 3.5 R2 청크 업로드 병목

- **원인**: 100 명 × 2 트랙 × 초당 2 chunks = **400 writes/s peak** (500ms 청크)
- **대응**:
  - Presigned URL로 클라 → R2 직접 (서버 개입 없음)
  - **청크 5초 단위로 늘려 write 수 1/10** (비용 · 병목 동시 완화)
  - 실패 시 IndexedDB 큐 + 3회 재시도
- **검증**: 실기기 30~50명 리허설로 실측

### 3.6 최종 제출 순간 폭주

- **원인**: 시험 종료 시각 근처 100명 동시 최종 제출 클릭
- **대응**:
  - Server Action idempotency key (`session_id`) 강제
  - 자동 제출은 클라에서 순차 발동 (미세 시차)
  - 서버는 락 없이 upsert (한 번만 처리되도록 응답 상태 확인)
- **검증**: 100 세션 종료 시각을 초 단위로 동시 발동 시뮬

### 3.7 브라우저 부하 (WebRTC + MediaRecorder + face-api 동시 · WebAudio 제거)

- **원인**: 응시자 노트북에서 4개 스트림 처리 동시
- **대응**:
  - 대기실에서 CPU 벤치마크 측정 (경쟁하는 4개 작업 시뮬)
  - 최소 사양 명시 (Intel i5 8세대 or Apple M1 이상 권장)
  - 저사양 응시자는 관리자 예외 승인 · 신분증 인증 강화
  - face-api 감지 주기 2.5s → 저사양이면 5s로 자동 완화
- **검증**: 저사양(Celeron·구형 i3) 노트북에서 실측 리허설

### 3.8 Supabase Realtime 채널 요금

- **원인**: 응시자 outbound POST + 관리자 폴링 방식으로 채널 최소화 필요
- **대응**:
  - 응시자는 outbound POST만 (Realtime channel 미사용)
  - 관리자 대시보드는 5초 폴링
  - 개별 상세 진입 시에만 Realtime channel 개별 구독
- **검증**: 부하 시나리오에서 총 Realtime 채널 < 15 확인

---

## 4. 부하 검증 시나리오 (M6)

### 4.1 API 부하 (Playwright · Artillery)

- 100 세션 동시 시험 시작 → 문제 페치 · 답안 저장 · 이벤트 batch → 최종 제출
- **측정**: p50/p95/p99 응답 시간 · 유실률 · 에러율

### 4.2 실기기 리허설 (WebRTC 필수)

- 실 노트북 20~30대로 30 세션 실행 → 스케일링해서 50 세션까지
- **측정**: Agora 연결 성공률 · 녹화 청크 성공률 · 감독 이벤트 정확도

### 4.3 감독관 시나리오

- 감독관 1~2명 로그인 · 그리드 관찰 · 개별 상세 진입 · 채팅 · 강제 종료
- **측정**: 이벤트 알림 지연 · 스트림 프레임률 · CPU 부하

### 4.4 종합 리허설 (M6 종료 조건)

- **50명 실기기 리허설** (100명 완전 시뮬 절반 규모로 검증)
- 답안 유실 0 · p95 < 목표 · 시험 종료 시각 최종 제출 폭주 안전
- 사전 30명 리허설 → 문제 발견 → 50명 리허설 → 시험 배포

### 4.5 프로덕션 실전 시험 첫 회

- 첫 회 100명 그대로 배포 가능 (300명 대비 부담 훨씬 적음)
- 문제 발생 시 즉시 대응 가능한 규모

---

## 5. 비용 절감 · 대안 검토

### 5.1 화상 관찰 미사용 시 (Agora 완전 제거)

- Agora 비용 0원
- 감독은 이벤트 로그 + R2 녹화 사후 재생만
- 회당 총비용 **8.6만원 → 6.9만원** (약 20% 절감)
- 담당자이 실시간 관찰 필요하다고 결정 A했으므로 유지 · 향후 재검토 여지

### 5.2 R2 청크 최적화

- **5초 청크** (권장) — write 요금 1/10 · 실시간 검토 다소 손실
- 30일 이후 자동 삭제 → 저장 비용 안정
- 이의 제기 대비 원본 필요 시험은 별도 archive 티어

### 5.3 Resend Free 유지 (월 1~2회 시험)

- Free tier 3,000통/월 커버
- 도메인 인증(SPF/DKIM)은 Free에도 지원
- 월 3회 이상 시험이면 Pro($20)로 업그레이드

---

## 6. 위험 · 미검증 항목 (실제 실행 필요)

우선순위 순:

| # | 항목 | 위험 | 실행 방법 |
|---|---|---|---|
| 1 | **Agora 계정 생성 · 프로젝트 세팅** | Seoul 리전 App ID 발급 · Web SDK 통합 · **결제 카드 등록** (Free 10,000분 초과 대비) | agora.io 계정 → Console에서 프로젝트 생성 → App ID/Certificate 발급 → Billing 카드 등록 |
| 1a | **Agora screen sharing 카운트 방식 확인** | 응시자당 웹캠+화면공유 트랙이 각각 카운트되면 요금 2배 | Agora sales(support@agora.io)에 문의 or 소규모 실사용 후 계산서 확인 |
| 2 | **실기기 50명 리허설** | Playwright는 WebRTC 미지원 → 실기기 필요 | 회사 노트북 30~50대 준비 |
| 3 | **저사양 응시자 CPU 부하** | Celeron 노트북에서 프리징 가능성 | 대기실 CPU 벤치 · 최소 사양 명시 · 예외 승인 |
| 4 | **face-api 감지 정확도** | 안경·조명·마스크 영향 | 다양한 환경에서 사전 검증 · 임계치 조정 |
| 5 | **R2 청크 write 요금** | 500ms 청크면 요금 10배 | 5s 청크로 고정 · 미결정 §7 보관 기간 조기 확정 |
| 6 | **Supabase 커넥션 초과 (안전용)** | 100명은 여유이나 예외 상황 방어 | Supavisor(transaction mode) M1에서 활성화 |
| 7 | **Fullscreen API 우회** | 응시자가 브라우저 로 열어놓고 우회 가능 | Page Visibility · pointerlock · 녹화로 사후 증거 확보 |
| 8 | **시험 시작 시각 클럭 스큐** | 응시자 로컬 시계 조작 | 서버 시간 재동기(NTP 스타일 · RTT/2 보정) · start_time 서버 확정 |

---

## 7. 미결정 재정리 (M1 진입 전 확정 필요)

| # | 항목 | 실행 액션 (담당자) |
|---|---|---|
| 1 | ~~Daily.co 플랜~~ → **Agora 확정 (2026-07-14)** | agora.io 계정 생성 · Seoul 프로젝트 · App ID/Certificate 발급 |
| 2 | **Cloudflare R2 계정 명의** | daeasy 회사 계정 vs 담당자 개인. R2 활성화 · SigV4 credential 발급 |
| 3 | **이메일 발송 채널** | Resend Free plan (권장 · 월 1~2회면 충분) · 도메인 SPF/DKIM 인증 |
| 4 | **얼굴 감지 모델** | face-api.js 유지(원본과 동일 · 안정) vs MediaPipe → 초기 face-api 권장 |
| 5 | **응시 녹화 보관 기간** | 30일 (권장 · 이의 신청 커버) / 90일 / 1년. R2 저장 비용에 직결 |

---

## 8. 결론

- **100명 동시 응시는 여유롭게 처리 가능**
- **회당 총 약 14만원 · 응시자 1인당 약 1,400원** (Video HD 기준 · 인건비 제외)
- **첫 회는 Agora Free tier로 실질 약 8만원** (10,000분/월 활용)
- Daily.co → **Agora로 화상회의 비용 30% 절감** (10만원 → 7만원 · Free tier 활용 시 더 큼)
- 실기기 리허설 규모도 30~50명 이내로 현실적
- 첫 시험도 100명 그대로 배포 가능
- M1 진입 전 미결정 5개 (Agora 계정 + screen sharing 검증 · R2 계정 · 이메일 · 얼굴 모델 · 보관기간) 확정하면 됨

**다음 액션**: 미결정 4개 결정 → M1 진입 → Agora/Supabase/R2 계정 세팅 → 스키마 마이그레이션.
