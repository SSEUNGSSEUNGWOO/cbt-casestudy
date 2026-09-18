# kbrain-cert — 기능 리스트

역할별 분류. **[핵심]** = MVP(M1~M5) 필수. **[확장]** = M6 이후. **⚠️** = 원본 plan 이슈 반영. 이식 결정 상세는 `DECISIONS.md`.

---

## 역할 (Role)

| 역할 | 설명 | Supabase Auth 구분 |
|---|---|---|
| **admin** | 문제·시험·응시자·채점·통계 전체 관리 | `role='admin'` |
| **examiner** | 실시간 감독관 (Daily 스트림 관찰·이벤트 알림) | `role='examiner'` |
| **grader** | 서술형·작업형 수동 채점 (admin 겸임 가능) | `role='grader'` |
| **applicant** | 시험 응시 | `role='applicant'` |

---

## Admin (관리자) 기능

### 문제은행 관리 [핵심]
- 문제 CRUD — **모든 문항이 작업형(슬롯형)** (결정 I). 유형 분기 자체가 없음
- **슬롯 구성** — `submission_slots[]`, 각 슬롯 타입: `text` · `long_text` · `url` · `file` · `number`, 슬롯별 `label` · `max_score` · `accept`(파일 확장자 필터)
- 슬롯 예시: 결과 파일(`file`) + 구현 메모(`long_text`) + 실행 결과 URL(`url`)
- 문항별 `rubric`(채점 기준) — 서버 전용, 클라 미노출
- 문항별 첨부파일(zip 배포자료 등), 태그, 카테고리, 등급, 난이도
- ⚠️ **rubric은 서버 컴포넌트에서만 렌더**, 클라 응답에 미포함 (이슈 #2)
- 문제 변경 이력 로그

**자동채점 없음** (결정 I 파급) — `correct_answer` · `options` · `is_correct` · 자동채점 규칙 컬럼 모두 스키마에서 제거. `placeholder == correct_answer` 가드는 무의미해져 함께 제거.

### 문제 세트 관리 [핵심]
- 시나리오형 세트 (`question_sets`: 제목·시나리오·첨부·순서·총점)
- 세트에 문제 추가/순서 조정
- JSON/CSV 대량 업로드 (슬롯 구성 검증)
- ⚠️ **세트별 `proctoring_disabled` 토글** (이슈 #1)
  - "부정행위 감지 비활성화 (외부 도구 허용 구간)"
  - **모든 문항이 작업형이므로** 기본 정책은 감독 비활성 vs 활성 중 어느 쪽을 표준으로 할지 시험별 결정 (M2 진입 시 확정)

### 카테고리 · 등급 관리 [핵심] (원본 대비 개선)
- 원본 하드코딩 enum 해제
- `question_categories` 관리 페이지 (색상 · 이름 · 순서)
- `exam_grades` 관리 페이지 (색상 · 이름 · 순서)
- 사업 확장 시 유연

### 시험 관리 [핵심]
- 시험 CRUD (제목·시험일시·시간·상태 draft/open/closed·합격기준·최대응시자수)
- **인라인 예약 시각 편집** (2026-07-15 구현) — `/admin/exams` 카드 우상단 "시각 편집" 버튼 · datetime-local + 시험 시간(min) · `PATCH /api/admin/exams/{id}`로 저장
- 절대시간 (`use_absolute_end`) / 상대시간 / 테스트 모드 (`is_test_mode`) — 현재 정책은 **절대 시각 통일 (모든 응시자 exam_date 기준)** 이 기본
- 입실 허용 시간 (`entry_start_minutes` 분 전부터) — 스키마만 있음
- 세트 조합 (드래그앤드롭 순서)
- 감독 알림 화이트리스트 (`alert_event_types`)
- 시험별 커스텀 문구 (`custom_texts`)
- 듀얼모니터 허용·듀얼모니터 강제 등 정책 플래그

### 초대전용 응시자 관리 [핵심]
- **+ 초대 만들기 모달** (2026-07-15 구현) — 시험 · 이메일 · 이름 · 소속 · 발송 여부 · 12자 hex invite_code 발급 후 진입 URL 즉시 표시
- **명단 CSV 업로드** (2026-07-16 구현) — `email,name,organization` 헤더 · 최대 1000행 · UTF-8 · 브라우저 파싱 · 미리보기 → `POST /api/admin/invitations/bulk` · 중복 이메일 스킵 · 템플릿 다운로드 제공
- **초대 이메일 발송** — Resend stub (`lib/email/send-invitation.ts` · 콘솔 출력) · API 키만 넣으면 실 발송
- 개별 예외 설정 (듀얼 모니터·웹캠 없음·화면공유 없음 허용) — 스키마만 있음
- 초대 상태 조회 (미사용/사용/만료)
- **응시자별 준비 상태 뷰** (2026-07-15 구현) — 3개 dot 뱃지(환경·서약·대기) 클릭 시 상세 모달 · 5스텝 타임라인 + 환경 체크 6개 항목 스냅샷 + UA

### 응시 모니터링 [핵심] (2026-07-16 구현)
- **실시간 그리드** `/examiner/monitor` — 3단 알림 우선 정렬(주목 3열 · 경고 5열 · 정상 10열) · Supabase Realtime 구독 즉시 반영
- **개별 응시자 상세** `/examiner/session/[id]` — 5스텝 타임라인(환경/서약/대기실/시작/제출) · 환경체크 6항목 스냅샷 · 이벤트 타임라인(200개) · 답안 미리보기 · UA
- **감독관 액션**:
  - **강제 종료** — 사유 입력 · 즉시 status=submitted · `session_messages`에 system 안내
  - **시간 연장** — 1~120분(5/10/15/30 프리셋) · `exam_sessions.time_extension_minutes` 누적 · 클라이언트 타이머와 서버 cron 모두 자동 반영
- **실시간 채팅** (`session_messages` 테이블):
  - 감독관↔응시자 양방향 · Realtime 구독
  - **공지 모드** — 응시자 창 자동 open · 빨간 강조 · 📢 뱃지
  - system 메시지 (강제 종료 · 시간 연장 자동 안내)
- 감독 이벤트 필터/스트림 (이하 §감독 이벤트)

### 신분증 사후 검토 [핵심]
- 응시자가 업로드한 신분증 사진 조회 (AWS Rekognition 미사용, 관리자 눈으로 확인)
- 승인/반려 · 코멘트

### 채점 관리 [핵심]
- **자동채점 없음** (결정 I) — 모든 문항이 작업형 슬롯이라 자동채점 대상 자체가 없음
- 수동채점 대기열 · 채점자 배정
- ⚠️ 점수 표시 (raw/max)*100 통일 (이슈 #3)
- 슬롯별 부분점수 · 감점 규칙 편집 · 코멘트
- **답안 Export** — CSV/JSON, 시험/응시자/문제 기준 (외부 채점 위임용) — 작업형만이라 export 활용도 높아짐 (첨부 파일 zip 묶음도 함께)

### 결과 · 통계 [핵심] (2026-07-16 구현)
- **`/admin/exams/{id}/results`** 시험별 결과·통계 페이지:
  - 요약 통계 6개: 총 응시 · 제출 완료 · 진행 중 · 자동 제출 · Flagged · HIGH 이벤트
  - 메트릭 3개: 응시 완료율 · 평균 소요 시간 · 전 문항 답변 응시자 수
  - 응시자 목록 테이블 (이름 · 조직 · 상태 뱃지+🚩+AUTO+`+분` · 시작 · 소요 · 답변 진행바 · 이벤트 카운트 · 상세 링크)
  - flagged 응시자 행 붉게 하이라이트
- **답안 zip export** (`↓ 답안 zip 다운로드` 버튼):
  - `{시험제목}_answers_{YYYYMMDD}.zip`
  - `summary.csv` (15 컬럼 · UTF-8 BOM)
  - `README.txt` (구조 안내)
  - 응시자별 폴더 `{이름}_{초대코드}/`
    - `_info.md` — 상태·소요·이벤트 통계·감독관 메모
    - `Q{번호}_{슬롯순번}-{슬롯라벨}.txt` — text/long_text/url/number
    - `Q{번호}_{슬롯순번}-{슬롯라벨}-{원본파일명}.{ext}` — file 슬롯 (원본 파일)
  - `auto_submitted/{이름}_{초대코드}/` — 자동 제출 격리
- ⚠️ 채점자 위임용 (작업형 전용 · 원본 파일 자체가 채점 대상)
- ⚠️ CSV 헤더 · CrossTable 라벨 모두 `/100` 표기 (이슈 #3) — 채점 붙는 M5에서
- 통계 대시보드 (M5 이후):
  - 등급별 합격률 (Bar)
  - 월별 응시 추이 (Line)
  - 기관별 등급 분포
  - 카테고리별 평균 점수 · 문항별 정답률

### 녹화 검토 [핵심]
- 세션별 녹화 청크 검색 · 재생 (R2 프록시)
- 진단 로그 (초기화 → 청크 생성 → 업로드 → DB 저장 각 단계)
- 녹화 통계 (성공률·청크수·소요시간)

### 사용자 관리 [핵심]
- 사용자 역할 관리 (admin · examiner · grader · applicant)
- 프로필 (조직·부서·직책·전화)
- 계정 삭제

### 시스템 설정 [핵심]
- 사이트 브랜딩 (로고 · 색상 · 푸터 · 타이틀 · 서브타이틀)
- 이메일 헤더 (`email_from_name`, `email_from_address`, `email_subject_prefix`)

### 확장 [M6+]
- 감사 로그 (누가 언제 문제/점수 변경)
- 크로스 채점 (2인 채점 편차 검토)
- 오픈 등록 모드
- AI 유의사항 자동 정리 (OpenAI)

---

## Applicant (응시자) 기능

### 로그인 [핵심]
- 초대 이메일 링크 진입 → 이메일 OTP 검증 → 세션 생성
- 표준 회원(이메일·비밀번호) 로그인 지원
- Google OAuth (Supabase Auth)
- 비밀번호 재설정

### 응시자 4-Step Wizard [핵심] (2026-07-15 구현)
`PracticeRunner` 컴포넌트 하나가 Practice 링크와 실 시험 모두 지원. 스텝 완료 시 다음 스텝 활성 · 이전 스텝은 done ✓ 뱃지. 웹캠/화면공유 스트림은 Step 1 획득 후 시험 종료까지 유지 (재연결 없음).

#### Step 1 · 환경 체크 6개 항목
번호 매긴 순차 카드 · 진행바 N/6 · 각 항목 status(ok/warn/error) 색상 뱃지 + 재시도 버튼:

1. **듀얼 모니터** — Window Management API (`screen.isExtended`) · 2개 감지되면 error
2. **웹캠** — `getUserMedia` 자동 요청 · 프리뷰 표시
3. **화면 공유** — `getDisplayMedia` 사용자 클릭 · 전체 화면 선택 필요
4. **네트워크** — `/api/time` 3회 왕복 평균 ms · <300ms ok, <800ms warn, ≥800ms error
5. **CPU 성능** — `Math.sqrt` 5M 반복 벤치 + `navigator.hardwareConcurrency` · 4코어/400ms ok · 2코어/800ms error
6. **브라우저** — Chrome/Edge/Firefox + Fullscreen API 지원 여부

**통과 조건**: 1~4는 error 아니면 통과 · 웹캠·화면공유·브라우저는 ok 필수. 통과 시 하단 "보안 서약으로 이동 →" 활성.

#### Step 2 · 보안 서약 7개 항목
```
01. 시험 중 허용되지 않은 외부 프로그램·웹사이트·메신저 등을 사용하는 행위
02. 다른 응시자와 답안을 공유하거나 제3자의 도움을 받아 답안을 작성하는 행위
03. 시험 화면·문항·답안을 촬영·캡처·녹화하거나 외부로 유출하는 행위
04. 대리 응시 등 본인이 아닌 자가 대신 시험을 치르는 행위
05. 감독 기능(웹캠·화면공유 등)을 고의로 비활성화하거나 방해하는 행위
06. 시험 전·중·후 문항 내용과 답안을 타인과 공유하거나 논의하는 행위
07. 기타 공정한 시험 운영을 저해하는 일체의 부정행위
```
모두 체크해야 "동의하고 대기실로 이동 →" 활성.

#### Step 3 · 대기실
- **Practice**: `언제든 입장 가능` · **"시험 입장하기 →"** 즉시 버튼
- **실 시험**: `exam.exam_date`까지 HH:MM:SS 카운트다운 · **0되면 자동 입장** (모든 응시자 동시)
- 완료 체크리스트 3줄 (환경 · 서약 · 시험 정보) · 시험 정보 카드 (등급 · 시간 · 합격 기준)

#### Step 4 · 시험창 (이하 §시험 응시)

**신분증 촬영** — Rekognition 미사용, 관리자 사후 검토. 대기실 별도 스텝 or 초대 발송 시 첨부로 처리 (M3 후속).

### 시험 응시 [핵심]
- 문제 네비게이션 (이전/다음/번호 점프 · 좌측 그리드)
- **타이머 (2026-07-15 3중 방어 구현)**
  - **exam.exam_date + duration_minutes 절대 시각 통일** — 모든 응시자 동시 시작/종료 (개인차 없음). 13:00 시험이면 15:00에 전원 자동 제출
  - 개인 진입 시각(`exam_sessions.start_time`)은 감독관 참조용으로 로깅만
  - **클라이언트 회복**: `useExamTimer` 훅에 `visibilitychange` · `focus` · `online` 리스너 → 백그라운드 탭에서 돌아오면 즉시 `Date.now()` 재계산
  - **서버 크론 백업**: `pg_cron` `auto_submit_expired_sessions()` 매분 실행 → 페이지 닫힌 응시자도 정확한 종료 시각에 강제 제출 (`auto_submitted=true`)
  - 색상 존: >10분 파랑 · 3~10분 노랑 · <3분 빨강+깜빡
- 답안 auto-save (1.5초 debounce · `POST /api/exam/answers/save` · upsert by session+question) — 저장 상태 표시 (pending/saved · HH:MM:SS/error)
- 명시적 제출 (마지막 문항에 초록 CTA · 확인 다이얼로그 · 미답변 개수 경고)
- **슬롯형 답안 UI만** (결정 I) — 슬롯 조합 렌더러 하나로 모든 문항 처리
  - `text` — 단일 input
  - `long_text` — textarea
  - `url` — url validation input
  - **`file`** (2026-07-16 구현) — 드래그·드롭 업로드 + accept 필터 + 업로드 완료 후 다운로드/삭제 · 슬롯당 1개 · 최대 50MB
  - `number` — numeric input
- **파일 업로드는 Supabase Storage `answer-files` bucket** (2026-07-16 구현) — 서버 프록시 (`/api/exam/answers/upload`) · 세션 쿠키 인증 · path = `{sessionId}/{questionId}/{slotId}/{hash}{salt}{ext}` · slot_values에 `{ path, name, size, mime }` 저장
- **파일 다운로드** — `/api/exam/answer-files/[...path]` · admin/examiner OR 응시자 세션 쿠키 (path의 sessionId 매칭)
- 문항별 제출 체크리스트 (kbrain-ems 방식 참고) — 슬롯 채워짐 상태 자동 체크
- Markdown 문제 내용 렌더 (remark-gfm)
- 문제별 정책 배너 (외부 도구 허용 범위 안내)

### 감독 (Proctoring) [핵심]
- **ProctorGuard 컴포넌트** (2026-07-16 구현) · 시험창(exam 탭)에서만 활성:
  - **Fullscreen 강제** — 자동 요청 · 이탈 감지 · **5회 위반 시 자동 제출** · 검정 오버레이(클릭으로 복귀)
  - **탭 이탈** — visibilitychange 즉시 감지 → tab_switch 이벤트 + 검정 오버레이
  - **윈도우 blur 3초+** — 조용한 alt-tab 감지
  - **복사/붙여넣기/잘라내기 차단** — copy_blocked (input/textarea/contenteditable 제외)
  - **우클릭 · 드래그·드롭 차단** — context_menu_blocked
  - **키보드 단축키 차단** — F12 · PrintScreen · Cmd+Shift+3/4/5 · Ctrl+Shift+I/C/P/S/J · Ctrl+P/S/U
  - **인쇄 · beforeunload 트랩** — print_attempt · navigation_attempt
  - **진입 30초 유예** — permission dialog 오탐 방지
- **얼굴 감지: face-api.js** (M4 · 2.5s 간격, 7.5s 연속 미검출 시 경고, 다인원 즉시)
- **Agora Web SDK 웹캠·화면공유 실시간 스트림** (M4) — 감독관 관찰용 (Seoul 리전 · SD simulcast)
- **R2 녹화** (M4) — MediaRecorder 500ms 청크, 웹캠+화면 이중 트랙
- **이벤트 배치 전송** (2026-07-16 구현) — 5s window / 50개 도달 시 flush / beforeunload sendBeacon · `monitoring_events` bulk insert · high severity면 세션 `is_flagged=true` 자동 마킹
- ⚠️ `proctoring_disabled=true` 세트 진입 시 감독 unmount + 배너 표시 (이슈 #1)

### 응시자 ↔ 감독관 채팅 [핵심] (2026-07-16 구현)
- **`session_messages` 테이블** (applicant · examiner · system 3종 sender_role · is_announcement 플래그)
- 응시자 시험창 우측 하단 💬 팝업 · 안 읽음 배지(9+ 표시 · 깜빡)
- **감독관 공지 도착 시 자동 open** + 빨간 말풍선
- **시간 연장·강제 종료**는 자동으로 system 메시지 삽입 · 중앙 안내 뱃지
- Supabase Realtime 구독 · 15초 fallback 폴링 (Agora RTM 대비 비용 절감)

### 제출 [핵심]
- 명시적 제출 (미답 확인 dialog)
- 시간 초과 자동 제출
- 제출 확인 페이지

### 결과 확인 [핵심]
- 자동채점 없음 → 제출 즉시 점수 노출 불가. "채점 대기 중" 상태 표시
- ⚠️ 점수 라벨 "환산 점수(100점 만점)" (이슈 #3)
- 수동채점 완료 후 사이트 내 알림 → 결과 열람 가능

### 마이페이지 [핵심]
- 응시 가능 시험 · 진행 중 · 완료 목록
- 과거 응시 이력

---

## Examiner (감독관) 기능

### 실시간 모니터링 대시보드 [핵심]
- **3단 알림 우선 정렬 그리드** (담당자 결정, 2026-07-14 프로토타입 리뷰):
  - **주목 필요 (Tier 1)** — HIGH 이벤트 · 스트림 오류 · 대형 카드 (aspect-video · 3열 그리드)
  - **경고 (Tier 2)** — WARN 이벤트 · 중형 카드 (5열 그리드)
  - **정상 (Tier 3)** — INFO 이하 · 소형 카드 (10열 그리드, 존재 확인용)
- 이벤트 발생 시 응시자가 자동으로 상위 티어로 승격 (감독관 시선 자연 유도)
- 감독 이벤트 실시간 알림 토스트 (`alert_event_types` 필터)
- 개별 응시자 상세 진입 (풀 스트림 · 화면공유 · 이벤트 타임라인 · 채팅)
- 공지 발송 (전체 or 개별)
- 강제 종료 · 시간 연장
- 성능 지표 패널 (Daily 연결 상태 · 프레임률)

### 이벤트 로그 [핵심]
- 세션별 · 응시자별 · 이벤트 타입별 필터
- 검토 상태 표시 (`is_reviewed`) · 리뷰어 코멘트

---

## Grader (채점자) 기능

### 채점 대기열 [핵심]
- 배정된 작업형 응답 목록 (모든 문항이 작업형)
- 응시자 익명화 옵션

### 채점 인터페이스 [핵심]
- 응답 · rubric 나란히 표시
- **슬롯별 부분점수 + 코멘트** (작업형이 슬롯 조합이므로)
- 첨부 파일 미리보기·다운로드 (텍스트·이미지·코드 하이라이트)
- ⚠️ "현재 총점 {환산}/100점 (원점수 {raw}/{max})" 표기 (이슈 #3)
- 다음 응답 이동 · 저장

### 답안 Export (외부 채점용) [핵심] (신규)
- 배정된 세션 답안을 CSV/JSON으로 다운로드
- 외부 CLI 도구·다른 채점자·AI 도구 등에 위임 가능
- Import 시 원래 attempt에 점수 반영

### 확장 [M6+]
- 크로스 채점 (2인 채점 편차 검토, `CrossTableDialog`)

---

## 감독 이벤트 타입

| 이벤트 | 소스 | 심각도 | 구현 시점 |
|---|---|---|---|
| `fullscreen_exit` | Fullscreen API | high (5회 후 강제 제출) | 2026-07-16 |
| `tab_switch` | Page Visibility API | high | 2026-07-16 |
| `window_blur` | window blur 3s+ | warn | 2026-07-16 |
| `copy_blocked` | copy/cut event | warn | 2026-07-16 |
| `context_menu_blocked` | contextmenu | info | 2026-07-16 |
| `devtools_attempt` | F12 · Ctrl+Shift+I/C/P/S/J | warn | 2026-07-16 |
| `screenshot_attempt` | PrintScreen · Cmd+Shift+3/4/5 | high | 2026-07-16 |
| `shortcut_blocked` | Ctrl+P/S/U | warn | 2026-07-16 |
| `print_attempt` | beforeprint | warn | 2026-07-16 |
| `navigation_attempt` | beforeunload | warn | 2026-07-16 |
| `face_missing` | face-api.js | warn (7.5s 지속 시) | M4 |
| `multiple_faces` | face-api.js | high | M4 |
| `screen_share_off` | Agora 화면공유 트랙 종료 감지 | high | M4 |
| `recording_error` | MediaRecorder 오류 | high | M4 |
| `identity_mismatch` (수동) | 관리자 신분증 사후 검토 반려 | high | M5 |

이벤트별 임계치·감점 규칙은 시험 관리자 설정으로.

---

## MVP 범위 (M1~M5)

- Admin: 문제은행·세트·카테고리·등급·시험·초대·모니터링·신분증검토·채점·답안Export·결과·통계·녹화검토·사용자·설정
- Examiner: 실시간 대시보드·이벤트 로그
- Grader: 채점 대기열·인터페이스·답안 Export
- Applicant: 로그인·대기실(신분증)·응시(타이머+감독+화상+녹화)·채팅·제출·결과·마이페이지

**M6 이후 확장**: 감사 로그·크로스 채점·오픈 등록·AI 유의사항 자동 정리
