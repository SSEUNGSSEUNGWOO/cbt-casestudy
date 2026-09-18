# kbrain-cert — 결정 사항 확정

**최종 갱신**: 2026-07-16

> **2026-07-16 추가 결정 (감독관 대시보드 · 액션)**:
> - **감독관 액션 = 시간 연장 + 강제 종료** (2종) · 사유 텍스트 필드 · 확인 모달
> - **시간 연장은 세션별 누적** — `exam_sessions.time_extension_minutes` 컬럼 · 클라이언트 타이머와 서버 pg_cron 모두 반영
> - **강제 종료는 `auto_submitted=false`** (시간 만료 자동 제출과 구분) · `monitoring_notes`에 `[force_submit] 사유` 로그
> - **실시간 채팅 = session_messages 테이블** — applicant/examiner/system 3종 sender_role · is_announcement 플래그 · Agora RTM 대신 Supabase Realtime 자체 구현
> - **공지 메시지는 응시자 창 자동 open** · 빨간 강조로 시각 우선순위
> - **감독관 시선 자동 유도** — 이벤트 발생 응시자가 3단 알림에서 자동 상위 티어로 승격 (주목/경고/정상)
**목적**: `INVENTORY.md` 검토 후 담당자이 확정한 이식 방향. `FEATURES.md`·`MASTER_PLAN.md`·`ARCHITECTURE.md`는 모두 이 결정을 기반으로 작성됨.

> **2026-07-15 추가 결정 (응시 core loop)**:
> - **응시자 진입 = 4-step wizard** (환경 체크 → 보안 서약 → 대기실 → 시험창) — Practice/실 시험 공용 컴포넌트 하나
> - **환경 체크 6개 항목 통과 게이팅** (1듀얼모니터·2웹캠·3화면공유·4네트워크·5CPU·6브라우저)
> - **타이머 절대 시각 통일** — `exam.exam_date + duration_minutes`. 모든 응시자 동시 시작/종료 (개인차 X)
> - **타이머 신뢰성 3중 방어** — 클라이언트 visibilitychange 재계산 + pg_cron 매분 서버 자동 제출 + exam_date 절대 시각
> - **웹캠/화면공유 스트림 재연결 없음** — Step 1에서 획득 후 4스텝 내내 유지 (부모 컴포넌트가 관리)
> - **응시자 인증 = 익명 세션 쿠키** (HMAC 서명 · 6h · `kbrain_exam_session`)
> - **Precheck 서버 저장 = A-a-i** (실 시험만 · 스텝별 upsert · 마지막 스냅샷 덮어쓰기)
> - **이메일 발송 = Resend stub 우선** (콘솔 출력 · API 등록 후 4줄만 교체하면 실 발송)

> **2026-07-14 결정**:
> - 문제 유형은 **작업형(work_based, 슬롯형) 한 종류만** 사용 (아래 **I** 항목)
> - **규모: 100명 · 120분 · 회당 계산** (초기 300명 가정에서 축소)
> - **화상회의: Daily.co → Agora로 재확정** (100명 규모에서 비용 1/6, `docs/CAPACITY.md` §1.2)

---

## 원본 스코프 대부분 이식 (공식 자격증급 수준)

담당자 목표: **kbrain-cert = 공식 자격증 발급 가능한 수준의 CBT**. 원본(AI Champion) 기능 대부분을 이식하되, 원본 plan의 4개 이슈(감독 유연성·정답 격리·점수 통일·타이머)는 근본 개선.

---

## 큰 방향 결정 8개 (확정)

| # | 항목 | 결정 | 파급 |
|---|---|---|---|
| **A** | 감독관 실시간 화상 관찰 | **✅ 이식 (Agora — 2026-07-14 재확정)** | 감독관이 응시자 웹캠·화면공유를 실시간 관찰. 그리드 뷰 + 개별 상세. Agora Web SDK · Seoul 리전 · SD simulcast로 비용 절감. 초기 Daily.co 안에서 변경 (Daily 대비 비용 1/6) |
| **B** | 응시 녹화 | **✅ 이식 (Cloudflare R2 전면 녹화)** | 웹캠·화면 청크 500ms 단위 R2 업로드. 사후 재생·검토 가능. WebAssembly MediaRecorder + AWS SigV4 서명 |
| **C** | 본인 인증(신분증) | **🔧 업로드만** | AWS Rekognition은 **제외**. 응시자가 신분증 사진 업로드 → **관리자 사후 검토** (개인정보 처리방침 부담 감소) |
| **D** | 응시자 등록 방식 | **✅ 초대전용 메인** | 관리자가 명단 업로드 → 초대코드·상시링크 이메일 발송 → 이메일 OTP로 응시자 진입. **오픈 등록은 M6 이후로** |
| **E** | 서술형 AI 채점 | **❌ 없음** | 자동채점(객관식·단답)만 유지. 서술형은 **답안 CSV/JSON export** 기능으로 대체 → 관리자가 외부 CLI/사람에게 위임 |
| **F** | 인증서 발급 | **❌ 제외** | `certifications` 테이블·`/admin/certifications` 페이지 모두 제거. "시험만 보면 됨" 수준 |
| **G** | 카테고리 하드코딩(생성형AI/데이터분석/서비스구현) | **✅ 관리자 설정으로 해제** | `question_categories` 테이블 신설 |
| **H** | 등급 하드코딩(green/blue/black/전문인재) | **✅ 관리자 설정으로 해제** | `exam_grades` 테이블 신설. `exam_grade` enum 제거 |
| **I** | 문제 유형 | **🔧 작업형(work_based, 슬롯형)만 사용** | 객관식·단답·서술형·실기 제거. 자동채점 로직 자체 삭제. 모든 문항 = 슬롯 조합(파일·long_text·URL·숫자·텍스트). 채점은 슬롯별 부분점수 수동 or 외부 export 위임 (E 결정과 자연 결합) |

---

## 이식 매핑 상세

### ✅ 이식 (원본 그대로 or 마이너 수정)

- 페이지 21개 중 18개 (인증서·Face++ 테스트·`send-otp/verify-otp` 제외)
- **문제 유형은 작업형(슬롯형) 한 종류만** — 나머지 4종(객관식·단답·서술형·실기)은 스코프에서 제거 (결정 I) + 시나리오형 세트
- 감독 로컬 컴포넌트 (`FaceMonitor` · `FullscreenGuard` · `SecurityPledge`) — `AntiOcrWatermark` 제외 · `VoiceMonitor` 제외 (2026-07-15 담당자 결정: 마이크 미사용 · 웹캠만 감시)
- 감독관 화상 관찰 (원본은 Daily.co 이었으나 **Agora로 재확정** · `AgoraProctor`, `AgoraMonitorGrid`, `agora-token` Edge Function). 채팅은 Agora RTM 대신 **Supabase Realtime 자체 구현**으로 비용 절감
- R2 녹화 (`RecordingStatusBadge`, `RecordingReviewPage`, `r2-*` Edge Functions)
- 초대전용 OTP (`send-guest-otp` · `verify-guest-otp` · `send-exam-invitation` · `exam_invitations` 테이블)
- 크로스테이블 CSV export (`CrossTableDialog`)
- 통계 대시보드 (`StatsPage`)
- 시험 관리 (절대시간·상대시간·테스트 모드·`entry_start_minutes`·`custom_texts`·`alert_event_types`)

### 🔧 아이디어만 가져와 새로 설계

- **본인 인증**: `verify-identity` Edge Function 제외 → `applicant_documents` 테이블에 신분증 이미지 저장 + 관리자 사후 확인 UI
- **AI 유의사항 정리** (`polish-instructions`): OpenAI로 재작성 검토 → M6 이후
- **카테고리·등급 관리**: 원본 enum 제거 → `question_categories`, `exam_grades` 테이블
- **답안 export**: 원본에 없음 → 신규 기능. 세션·응시자·문제 기준 CSV/JSON export (서술형 채점용)

### 🕒 확장 (M6 이후)

- 오픈 등록 모드
- 듀얼 모니터 감지
- 관리자 감사 로그
- 크로스 채점 (2인 채점 편차 검토)

### ❌ 제외

- 인증서 발급 (`certifications` 테이블 · 페이지 · 발급 로직 전부)
- AWS Rekognition (`verify-identity` Edge Function)
- 서술형 AI 채점 (`ai-grade` Edge Function · Lovable Gemini)
- Face++ 테스트 페이지, Zoom SDK (원본에서도 미사용), 개발용 테스트 페이지들
- **자동채점 로직 전부** (결정 I 파급) — 객관식 정답 비교, 단답 exact/numeric, `correct_answer` 컬럼 무의미. 모든 채점 = 슬롯별 수동 or 외부 export 위임
- **문제 유형 4종의 응시자 UI 컴포넌트** — 라디오 선택지, 단답 input, 서술형 textarea 등 유형별 UI 필요 없음. 슬롯 컴포넌트 하나로 통일

---

## 감수한 리스크 (담당자 확인 완료)

| 리스크 | 대응 방안 |
|---|---|
| **Agora 화상회의 비용** | 100명·120분·회당 약 1.7만원 (SD 그리드 · Free 10,000분/월 활용 시 첫 회 3천~5천원). Daily.co 대비 1/6. `docs/CAPACITY.md` §1.2 |
| **R2 저장 비용 (시험당 100~300GB)** | 보관 정책 정해야 함 — 기본 30일 후 자동 삭제 or 저사양 저장 archive |
| **브라우저 부하** (WebRTC + MediaRecorder + face-api 동시 · WebAudio 제거로 다소 완화) | 대기실 사전 체크에 CPU 벤치마크 추가, 최소 사양 명시. 저사양 응시자는 관리자 승인 예외 |
| **실기기 부하 테스트 불가** (Playwright는 WebRTC 미지원) | M6에 실제 다중 노트북 리허설 필수 |

---

## 미해결

1. ~~Daily.co 견적~~ → **Agora (2026-07-14 확정)**
2. **R2 계정** — Cloudflare 계정 · R2 버킷 (담당자 or daeasy 명의)
3. **Resend 발신 이메일** — 발신 도메인 미정. 코드는 stub 상태
4. **얼굴 감지 모델** — face-api.js 유지 vs MediaPipe
5. **응시 녹화 보관 기간** — 30일 / 6개월 / 1년

---

## 2026-07-15 결정 세부 근거

### 1. 4-step wizard (환경 → 서약 → 대기실 → 시험창)
- **원본 참조**: `AI Champion Certification System`의 `WaitingRoom.tsx` · `SecurityPledge.tsx` · 대기실 자동 입장 흐름을 참고
- **변경점**: 원본 텍스트 8개 서약 항목 중 5번(AI 도구 항목)이 작업형 전용 프로젝트에는 부적합 → 삭제 후 7개로 축소 · "평가" → "시험" 용어 통일
- **왜**: 응시자가 실전 전에 미리 익힐 수 있게 Practice 링크에서도 동일 흐름 재사용. 시험 당일 첫 사용이 아님

### 2. 환경 체크 6개 (듀얼모니터·웹캠·화면공유·네트워크·CPU·브라우저)
- **CPU 벤치마크 추가 이유**: 웹캠 인코딩 + 화면 공유 압축 동시 처리 시 저사양 노트북 프레임 드롭 발생 · `navigator.hardwareConcurrency` + `Math.sqrt` 5M 벤치로 스크리닝
- **듀얼 모니터 감지**: `Window Management API` (`screen.isExtended`) · Chrome 100+ 지원 · 부정행위 예방 · 감지되면 error로 진입 차단
- **브라우저 인식만**: 원본은 브라우저 체크 강제 · 여기선 다른 카드가 실 API로 검증하므로 브라우저 카드는 정보만 표시 (통과 조건 X)

### 3. 절대 시각 통일 (exam_date 기준)
- **원본**: `use_absolute_end` 플래그로 시험별 선택 가능
- **결정**: 실 시험은 **절대 시각 통일이 기본**. 13:00 시작 시험이면 15:00 전원 자동 제출. 각자 다르지 않음
- **왜**: 사후 채점·순위 비교 편의 · 감독관이 종료 시각 예측 · 공정성

### 4. 타이머 신뢰성 3중 방어
- **문제**: 브라우저 백그라운드 탭에서 setInterval throttle (Chrome 1분에 1회) · 페이지 완전 닫힘 시 자동 제출 실패
- **1단**: `useExamTimer`에 `visibilitychange` · `focus` · `online` 리스너 → 활성화 즉시 재계산 (Date.now() 기반이라 오차 없음)
- **2단**: `pg_cron` 매분 `auto_submit_expired_sessions()` 함수 실행 → 페이지 닫혀도 서버가 강제 종료 (`auto_submitted=true`)
- **3단**: exam_date 절대 시각 통일 (개인차 X)

### 5. 응시자 익명 세션 쿠키 (Supabase Auth 자동 가입 대신)
- **선택**: HMAC 서명 쿠키 (`kbrain_exam_session` · 6h · HttpOnly · SameSite=Lax)
- **왜**: 응시자마다 auth.users 생성하면 관리 복잡 · 대량 응시(100~300명) 시 계정 폭증 · `exam_sessions.invitation_id`로 추적 가능하므로 불필요
- **보안**: `EXAM_SESSION_SECRET` 32bytes base64url · 서명 검증으로 위조 방지 · path별 exam 소속 확인 (첨부 API)

### 8. 답안 zip export 형식 (2026-07-16)
- **작업형 전용이라 CSV 하나로 부족** — 파일 슬롯 · 긴 long_text 등 이유
- **선택: 시험 전체 zip 하나** (담당자 결정 2026-07-16)
  - 응시자별 폴더 · 문항×슬롯별 파일 · 원본 파일명 유지
  - `summary.csv` (BOM) — 요약 · Excel 바로 열림
  - `auto_submitted/` 격리 — 채점자가 자동 제출자 별도 검토
  - `_info.md` — 응시자 상태·이벤트 요약 (채점자가 첫 파일로 참고)
- **파일 슬롯 원본 유지** — 확장자·인코딩 그대로 (zip 내에서만 관리 편의)
- **채점 위임 흐름**: 관리자가 zip 다운 → 채점자에게 전달 → 채점자가 파일별 점수 · CSV로 반영 후 되보내기 (import는 M5)

### 9. 파일 슬롯 저장 (2026-07-16)
- **Supabase Storage `answer-files` bucket** (private · service_role만 · 50MB 제한)
- **경로**: `{sessionId}/{questionId}/{slotId}/{md5hash 12}{salt 8}.{ext}`
  - md5 hash → 파일 무결성 검증 자연 획득
  - salt → 동일 파일 재업로드도 unique path (버전 관리)
- **slot_values 스키마**: `{ path, name, size, mime, uploadedAt }` (원본 파일명 유지)
- **왜 별도 Storage bucket?** — 시험 첨부(`exam-attachments`) vs 응시자 답안(`answer-files`) 완전 분리. 각기 다른 접근 제어
- **다운로드 인증 2방식**: admin/examiner 로그인 · 응시자 세션 쿠키(path의 sessionId 매칭)
- **여러 파일 슬롯 하나에?** — 지금은 1개만 · UI 단순화. 실 사용 후 필요하면 배열로 확장

### 7. 감독관 액션 · 채팅 스키마 (2026-07-16)
- **`exam_sessions.time_extension_minutes` int** — 세션별 누적 연장 시간. 종료 시각 = `exam_date + duration_minutes + time_extension_minutes`
- **`session_messages` 테이블** — sender_role(applicant/examiner/system) · content · is_announcement
  - **왜 별도 테이블?** monitoring_events는 감독 이벤트 로그 · 채팅은 사후 검토 목적이라 분리 (append-only는 공통)
  - Realtime publication에 추가 (즉시 반영)
- **강제 종료 = `auto_submitted=false`** — 시간 만료 자동 제출과 구분. `monitoring_notes` 컬럼에 사유 저장
- **`auto_submit_expired_sessions()` 함수 갱신** — `duration_minutes + time_extension_minutes` 기준으로 만료 판정
- **감독관 액션은 admin/examiner role만** — 감독관 자격증 발급 필요 시 role 확장

### 6. Precheck 저장 A-a-i
- **A (실 시험만)**: Practice는 응시자 불명 · 저장 노이즈. 실 시험 sessionId 있을 때만 저장. 훅 (`useSavePrecheck`)이 sessionId null이면 no-op
- **a (스텝별 upsert)**: env 완료 시 · 서약 동의 시 · 대기실 진입 시 각각 POST. 응시자가 어느 스텝에서 이탈했는지 감독관 파악
- **i (마지막 스냅샷)**: 재시도 이력 저장하면 UI 복잡 · 관리자는 "결과"만 필요
- **저장 위치**: 별도 테이블 대신 `exam_sessions` 컬럼 확장 — 세션과 1:1이라 join 불필요
