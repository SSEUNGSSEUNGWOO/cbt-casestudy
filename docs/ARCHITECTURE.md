# kbrain-cert — 아키텍처

**핵심 목표**: 동시 응시 100명·120분·회당 시험 진행하는 동안 답안 유실 0건, p95 응답 시간 500ms 이하 유지, 실시간 화상 관찰·전면 녹화 안정 동작. (2026-07-14 초기 300명 가정에서 축소)

---

## 기술 스택

| 레이어 | 기술 | 선택 이유 |
|---|---|---|
| Frontend / Backend | **Next.js 15 (App Router) + TypeScript** | 서버 컴포넌트로 정답 격리, Server Actions, kbrain-ems·dataeasy와 스택 통일 |
| UI | **shadcn/ui + Tailwind CSS + Radix Primitives** | 원본과 동일 |
| 상태 관리 | **TanStack Query v5** | 서버 상태 캐싱, 답안 저장 debounce · retry |
| DB · Auth · Storage · Realtime | **Supabase Pro** | RLS·Realtime·PgBouncer·Storage 통합 |
| **화상회의 (감독관 관찰)** | **Agora Web SDK** (`agora-rtc-sdk-ng`) | Seoul 리전 · SD simulcast (그리드 저해상도) · Daily.co 대비 비용 1/6 (2026-07-14 재확정) |
| **응시 녹화 스토리지** | **Cloudflare R2** + WebAssembly MediaRecorder + AWS SigV4 | 원본과 동일 (S3보다 저렴, egress 무료) |
| **이메일 발송** | **Resend** (or Supabase Auth 내장, 최종결정 대기) | 초대 · OTP 이메일 |
| 감독 (얼굴) | **face-api.js (TinyFaceDetector, v0.22.2)** | 브라우저 로컬 추론 (원본 유지) |
| 감독 (화면) | **Fullscreen API + Page Visibility API** | native |
| 문제 렌더링 | **remark-gfm (Markdown)** | 원본과 동일 |
| E2E 테스트 | **Playwright** | WebRTC 부분은 실기기 리허설 필요 |
| Unit 테스트 | **Vitest** | Next.js 15 호환 |
| 배포 | **Vercel** (Frontend) + Supabase (Backend) + Agora + R2 | 서버리스 · 자동 스케일 |
| 코드 품질 | ESLint (flat) + Prettier + TypeScript strict | |

**의도적으로 제외**: AWS Rekognition (신분증 → 업로드만), Lovable Gemini (AI 채점 미사용), Zoom SDK.

---

## 폴더 구조

```
kbrain-cert/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   ├── callback/
│   │   ├── reset-password/
│   │   └── invite/[token]/            # 초대 링크 진입 → OTP
│   ├── (admin)/                        # role=admin
│   │   ├── layout.tsx
│   │   ├── dashboard/
│   │   ├── questions/                  # 문제은행
│   │   ├── question-sets/              # 세트 (proctoring_disabled)
│   │   ├── categories/                 # 카테고리 관리 (신규)
│   │   ├── grades/                     # 등급 관리 (신규)
│   │   ├── exams/                      # 시험 관리
│   │   ├── invitations/                # 초대 명단 · 이메일 발송
│   │   ├── identity-review/            # 신분증 사후 검토 (신규)
│   │   ├── grading/                    # 채점 관리
│   │   ├── results/                    # 결과 · 통계
│   │   ├── recordings/                 # 녹화 검토
│   │   ├── users/                      # 사용자 관리
│   │   └── settings/                   # 사이트 설정
│   ├── (examiner)/                     # role=examiner
│   │   ├── layout.tsx
│   │   ├── monitor/                    # 실시간 대시보드
│   │   └── events/                     # 이벤트 로그
│   ├── (grader)/                       # role=grader
│   │   ├── layout.tsx
│   │   ├── queue/
│   │   └── export/                     # 답안 export (신규)
│   ├── (applicant)/                    # role=applicant
│   │   ├── layout.tsx
│   │   ├── my/                         # 마이페이지
│   │   ├── waiting/[sessionId]/        # 대기실 (환경체크 + 신분증)
│   │   ├── exam/[sessionId]/           # 응시 (타이머 · 감독 · Daily · 녹화)
│   │   ├── submitted/[sessionId]/
│   │   └── result/[sessionId]/
│   └── api/
│       ├── time/                       # 서버 시간 (타이머 동기화)
│       ├── proctoring/events/          # 감독 이벤트 배치
│       ├── exam/submit/                # 최종 제출
│       ├── invitation/otp/             # 초대 OTP 발송·검증
│       ├── agora/token/                # Agora RTC 토큰 발급
│       ├── r2/presign/                 # R2 Presigned URL
│       ├── r2/playback/                # R2 재생 프록시
│       └── export/answers/             # 답안 CSV/JSON export
├── components/
│   ├── ui/                             # shadcn
│   ├── proctoring/                     # FaceMonitor, FullscreenGuard, EventBatcher (마이크 미사용)
│   ├── exam/                           # QuestionRenderer, Timer, SlotAnswerPanel, MarkdownView, PolicyBanner
│   ├── daily/                          # DailyProctor, DailyMonitorGrid
│   ├── recording/                      # RecordingStatusBadge, RecordingChunker
│   ├── waiting-room/                   # EnvCheck, WebcamSelector, IdentityUpload, SecurityPledge
│   ├── admin/                          # 관리자 위젯
│   ├── examiner/                       # MonitorGrid, MonitorPerfPanel
│   └── grader/
├── lib/
│   ├── supabase/                       # server / client / middleware / rls
│   ├── daily/                          # SDK 래퍼 · 토큰
│   ├── r2/                             # SigV4 서명 · presign
│   ├── resend/                         # 이메일 템플릿
│   ├── grading/                        # score.ts (toPercentage)
│   ├── proctoring/                     # 이벤트 배치 · Realtime
│   ├── time/                           # 서버 시간 동기화 · 타이머
│   └── invite/                         # 초대 토큰 · OTP
├── types/                              # DB 타입 (supabase gen)
├── supabase/
│   ├── migrations/
│   ├── functions/                      # daily-room, r2-presign, r2-upload, r2-playback, send-guest-otp, verify-guest-otp, send-exam-invitation, delete-user
│   └── config.toml
├── e2e/                                # Playwright
├── docs/                               # MASTER_PLAN, FEATURES, ARCHITECTURE, INVENTORY, DECISIONS
├── public/models/                      # face-api 모델 (self-host, CDN 대체)
├── .env.local.example
├── package.json
└── next.config.ts / tsconfig.json / tailwind.config.ts
```

---

## 데이터 모델 (핵심 테이블)

```sql
-- 사용자 & 역할
user_roles (
  id uuid pk, user_id uuid fk auth.users, role text
    -- ('admin','examiner','grader','applicant')
)
profiles (id uuid pk fk auth.users, name, email, organization, department, position, phone)

-- 카테고리 / 등급 (원본 하드코딩 해제)
question_categories (id uuid pk, name text, color text, order_num int)
exam_grades (id uuid pk, name text, color text, order_num int)

-- 문제 & 세트 — 모든 문항이 작업형(슬롯형) (결정 I)
questions (
  id uuid pk, code text unique,
  category_id uuid fk question_categories,
  grade_id uuid fk exam_grades,
  -- type 컬럼 제거 (전부 작업형이라 의미 없음)
  difficulty text, tags text[],
  content text, attachments jsonb[],       -- 문제 내용 (Markdown) · 배포 자료 zip 등
  submission_slots jsonb not null,         -- [{ id, type, label, max_score, accept }]
                                           -- type ∈ (text, long_text, url, file, number)
  rubric jsonb,                            -- ⚠️ 서버 전용 (뷰로 격리, 채점자만 열람)
  max_score numeric,                       -- 슬롯 max_score 합계와 일치해야 함 (제약 검증)
  set_id uuid fk question_sets nullable,
  set_order int,
  created_by uuid, created_at timestamptz
)
-- 자동채점 컬럼 (options, correct_answer, is_correct 등) 전부 없음.

question_sets (
  id uuid pk,
  title text, scenario text,
  attachments jsonb[], total_score int, order_num int,
  category_id uuid, grade_id uuid,
  proctoring_disabled boolean not null default false,  -- ⚠️ 이슈 #1 최초부터
  created_at, updated_at
)

-- 시험
exams (
  id uuid pk,
  title text, grade_id uuid,
  exam_date timestamptz,
  duration_minutes int,
  max_participants int,
  status text check (status in ('draft','open','closed')),
  instructions text,
  registration_mode text check (registration_mode in ('invite_only','open','hybrid')) default 'invite_only',
  pass_score int,                        -- 100점 환산 기준
  is_test_mode boolean default false,
  use_absolute_end boolean default false,
  entry_start_minutes int default 60,
  allow_dual_monitor boolean default false,
  skip_waiting_checks boolean default false,
  daily_room_name text, daily_room_url text,
  custom_texts jsonb,
  alert_event_types text[],              -- 감독관 알림 화이트리스트
  created_by uuid, created_at, updated_at
)

exam_sets (exam_id uuid fk, set_id uuid fk, order_num int, primary key (exam_id, set_id))
exam_questions (exam_id uuid fk, question_id uuid fk, order_num int)

-- 응시자 초대
exam_invitations (
  id uuid pk, exam_id uuid fk,
  email text, name text, invite_code text unique,
  is_used boolean default false,
  allow_dual_monitor boolean, allow_no_webcam boolean, allow_no_screen_share boolean,
  created_at
)
guest_otp_codes (invitation_id uuid fk, email text, code text, expires_at, verified boolean)

-- 응시 세션
exam_sessions (
  id uuid pk,
  exam_id uuid fk, applicant_id uuid fk,
  status text check (status in ('waiting','in_progress','submitted','passed','failed')),
  start_time timestamptz, submit_time timestamptz,
  score_total numeric,          -- raw 저장, 표시는 헬퍼로 환산
  is_flagged boolean default false,
  identity_image_url text,      -- 신분증 이미지 (관리자 사후 검토)
  identity_review_status text check (identity_review_status in ('pending','approved','rejected')),
  identity_review_note text,
  monitoring_notes text,
  daily_room_url text,
  auto_submitted boolean default false,
  created_at, updated_at
)

-- 답안
answers (
  id uuid pk, session_id uuid fk, question_id uuid fk,
  content text, file_url text,
  slot_values jsonb, slot_scores jsonb,   -- 슬롯형
  score numeric,                          -- raw
  feedback text,
  graded_by uuid, graded_at timestamptz,
  submitted_at timestamptz
)

-- 감독 이벤트
monitoring_events (
  id bigserial pk,
  session_id uuid fk,
  event_type text,                        -- face_missing, multiple_faces, fullscreen_exit, tab_switch, screen_share_off, recording_error
  detected_at timestamptz,
  screenshot_url text,
  question_index int,
  severity text,
  is_reviewed boolean default false, reviewer_note text
)
-- session_id · 월 파티셔닝 (100명 * 이벤트 다수)

-- 녹화 청크
recording_chunks (
  id uuid pk, session_id uuid fk,
  kind text check (kind in ('webcam','screen')),
  chunk_index int,
  object_key text,                        -- R2 경로
  mime_type text, size_bytes int, duration_ms int,
  started_at, ended_at, created_at,
  is_header boolean default false
)

-- 사이트 설정
site_settings (key text pk, value text)   -- title, subtitle, footerOrg, email_from_*, ...
```

⚠️ **격리 뷰**:
```sql
create view questions_for_applicant as
  select id, code, category_id, grade_id, content, attachments,
         submission_slots,  -- 응시자에게 슬롯 구성은 그대로 노출 (label · type · max_score · accept)
         max_score, set_id, set_order
  from questions;
-- rubric 컬럼만 제외 (채점 기준은 응시자에게 노출 안 함)
-- RLS: applicant role은 이 뷰만 access
```

작업형 전용이라 격리해야 할 것이 rubric 하나뿐 — 원본의 정답 노출 이슈(#2)는 근본적으로 사라짐 (자동채점 자체가 없으니 노출할 정답이 존재하지 않음).

**표시·판정 규칙**: `lib/grading/score.ts` 의 `toPercentage(raw, max)` 헬퍼 하나만. DB view·CSV·화면 전부 여기 통과.

---

## 데이터 격리 (원본 이슈 #2 근본 해결)

**원본 문제**: 응시자 API가 문제 전체(정답·rubric 포함) 반환 후 클라 sanitize. 운영자가 `placeholder`에 정답 넣으면 노출.

**kbrain-cert 처리** (2중 방어):

1. **DB 뷰 격리**
   - `questions_for_applicant` 뷰가 정답 컬럼을 제외
   - RLS: applicant role은 이 뷰만 select 가능, 원본 테이블은 admin/grader/examiner만
2. **서버 컴포넌트**
   - `app/(applicant)/exam/[sessionId]/page.tsx`는 서버에서 뷰를 통해 페치, 클라 payload에 정답 존재 자체가 없음
3. **업로드 파서 가드**
   - `placeholder == correct_answer` 검사 → 경고 토스트 (2차 방어, 예방)

---

## 100명 동시 응시 처리 (상세는 `CAPACITY.md`)

### 병목 & 대응

| 병목 | 예상 부하 | 대응 |
|---|---|---|
| 시험 시작 순간 문제 페치 | 300 req/s | `unstable_cache` + Vercel Edge Cache (문제 payload는 응시자 공용) |
| 답안 저장 | 지속적 write (debounce 3s) | React Query mutation + PgBouncer transaction mode + upsert |
| 감독 이벤트 | 최악 초당 600+ | 클라 **5s 배치** → bulk insert. 파티셔닝(월/세션) |
| Agora 스트림 | 100 SFU 스트림 (SD 그리드) | Seoul 리전 · dual-stream mode(simulcast) 활성. 감독관은 저해상도 receive, 이벤트 발생 시만 HD 확대 |
| R2 녹화 업로드 | 청크 5s × 100명 × 2트랙 = ~40 upload/s peak | Presigned URL로 클라 → R2 직접 업로드, 서버 개입 최소 |
| 관리자 실시간 모니터링 | Realtime 채널 확산 | 관리자 대시보드 = 5s 폴링. Realtime은 개별 상세 진입 시만 |
| 최종 제출 폭주 | 종료 시각 근처 300 요청 동시 | 서버 액션 idempotency key(`session_id`) |
| Resend 이메일 발송 | 초대 명단 300건 배치 | Resend 배치 API, 백엔드 큐 |

### 티어 요구사항

- **Supabase Pro** (커넥션 200+, PgBouncer, Realtime 상향)
- **Agora Web SDK** (Seoul 리전 · 100명 동시 SFU · Free 10,000분/월 활용)
- **R2**: 저장 무제한(사용량 과금), egress 무료 (S3 대비 큰 이점)
- **Vercel Pro** (Edge Config, longer function timeout)

### 클라이언트 최적화

- 문제 전체 초기 1회 페치 → 오프라인/순단 대비
- 답안 IndexedDB 로컬 백업 → 네트워크 복구 시 sync
- face-api.js 모델 self-host (`public/models`), Service Worker precache
- Daily SDK · face-api · MediaRecorder 병렬 초기화 (대기실에서 warmup)

---

## 시간 동기화 (원본 이슈 #4)

`lib/time/serverClock.ts`:

```ts
// 대기실에서 오프셋 계산
const t0 = Date.now();
const { serverTime } = await fetch('/api/time').then(r => r.json());
const t1 = Date.now();
const rtt = t1 - t0;
const offset = serverTime - (t0 + rtt / 2);
// 응시 중 타이머: (Date.now() + offset)
```

`recomputeTimeLeft`:
```ts
async function recomputeTimeLeft(session) {
  if (!session.start_time) {
    for (let i = 0; i < 3; i++) {
      toast(`시간 동기화 재시도 중… (${i+1}/3)`);
      const fresh = await refetchSession();
      if (fresh.start_time) return computeFrom(fresh);
      await sleep(2000);
    }
    return autoSubmit();  // 3회 실패 후에만
  }
  return computeFrom(session);
}
```

절대시간 모드는 `isNaN(new Date(exam_date).getTime())` 가드.

---

## 감독 아키텍처 (원본 이슈 #1)

```
[Browser]
  ├─ FaceMonitor (face-api.js, 2.5s interval)
  ├─ FullscreenGuard + Page Visibility
  ├─ DailyProctor (Daily SDK: 웹캠·화면공유 send)
  ├─ RecordingChunker (MediaRecorder 500ms → R2 presigned upload)
  │
  └─ EventBatcher (5s window)
        └─→ POST /api/proctoring/events (bulk)
              └─→ Supabase insert → monitoring_events
                    └─→ Realtime → 감독관 상세 뷰

[Examiner]
  ├─ MonitorGrid (Daily SFU receive, 30~50개 그리드)
  └─ 이벤트 알림 토스트 (alert_event_types 필터)
```

**세트별 비활성화** (`proctoring_disabled=true`):
- FaceMonitor · FullscreenGuard 컴포넌트 unmount
- EventBatcher 정지 (이벤트 발생 자체를 안 함)
- Daily·녹화는 **유지** (감독관 관찰·사후 검토 목적, 응시자에겐 배너로 명시)
- 다른 세트 이동 시 자동 재활성화 (remount)

---

## Agora 통합 요약

- **채널명**: 시험별 (`exam-{exam_id}`) 하나. 응시자·감독관 같은 채널 join
- **응시자 토큰**: 응시 시작 시 서명 토큰 발급 (role=publisher · 웹캠·화면공유 send)
- **감독관 토큰**: 감독관 로그인 시 role=subscriber 토큰 (전체 receive) + role=publisher(개별 음성 안내 시)
- **Simulcast**: 응시자 dual-stream (HD + SD) 발행 → 감독관 그리드는 SD 요청 (비용 절감)
- **채팅**: Agora RTM 대신 **Supabase Realtime 자체 구현** (Agora 비용 절감 · 이벤트·채팅 통합 관리)
- **비용 관리**: 채널은 참여자·분 단위 과금 · Free 10,000분/월 활용 · 종료 후 leave

## R2 녹화 통합 요약

```
Browser MediaRecorder
  → 500ms chunk (webm)
  → POST /api/r2/presign (session_id, kind, chunk_index)
  → PUT chunk to R2 (presigned URL)
  → POST /api/r2/confirm (metadata → recording_chunks 테이블 insert)

실패 시 3회 재시도 → 실패해도 로컬 IndexedDB 큐잉 후 백그라운드 재전송
```

재생: `api/r2/playback` 서버 프록시로 R2 GET → MSE (Media Source Extension)로 청크 병합 스트리밍.

## 초대 · OTP 흐름

```
관리자
  → CSV 업로드 → exam_invitations 생성
  → send-exam-invitation Edge Function
       → Resend batch send (초대 링크 = /invite/{invite_code})

응시자
  → 링크 진입 → /invite/[token]
  → 이메일 입력 → send-guest-otp (Resend 6자리 코드)
  → OTP 입력 → verify-guest-otp
       → Supabase Auth 세션 생성 (또는 커스텀 JWT)
       → exam_sessions.applicant_id 연결
```

---

## 환경 변수

```
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Next.js
NEXT_PUBLIC_APP_URL=

# Agora
NEXT_PUBLIC_AGORA_APP_ID=
AGORA_APP_CERTIFICATE=          # 서버 전용 (토큰 서명)

# Cloudflare R2
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET=
R2_PUBLIC_BASE_URL=

# Resend
RESEND_API_KEY=
RESEND_FROM_EMAIL=

# 시크릿
OTP_SECRET=              # OTP HMAC
```

---

## 알려진 리스크

| 리스크 | 대응 |
|---|---|
| Agora 100 동시 SFU 요금 | SD simulcast + Free 10,000분/월 활용. 회당 약 1.7만원 예상 (CAPACITY.md §1.2) |
| R2 저장 무제한 → 비용 지속 증가 | 녹화 보관 정책 (기본 30일 후 자동 삭제 · 필요 시 archive 티어로 이동) |
| 브라우저 부하 (Agora·MediaRecorder·face-api 동시) | 대기실 CPU 벤치마크 · 최소 사양 표시 · 저사양 응시자는 관리자 예외 승인 |
| face-api 모델 첫 로드 지연(~10MB) | self-host + Service Worker precache + 대기실에서 warmup |
| WebRTC (Daily) Playwright 미지원 → 자동화 어려움 | M6에 실기기 다중 노트북 리허설 |
| Fullscreen API 브라우저 우회 가능 | 다층 이벤트(Page Visibility·pointerlock loss) 조합 + 녹화로 사후 증거 확보 |
| Supabase Realtime 채널 300 동시 | 관리자는 폴링, 응시자는 outbound POST만 (Realtime inbound 최소화) |
| Resend 발송 실패 (도메인 스팸 처리) | 대체 발신 도메인 · SPF/DKIM 인증 · 실패 시 관리자 대시보드에 재전송 버튼 |
| 신분증 이미지 개인정보 | Supabase Storage 격리 버킷 · 응시 종료 후 N일 자동 삭제 · 접근 로그 |
