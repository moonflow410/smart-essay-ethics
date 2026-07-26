# 📝 smart-essay-monitor — AI 서술형 평가 및 실시간 모니터링 시스템

고1 통합사회 · 고2 사회와 문화 서술형 평가용 웹앱입니다.
학생은 단계별 가이드로 답안을 조립해 제출하고, 교사는 실시간 대시보드에서
제출 현황·이탈 여부·AI 채점 결과를 한눈에 확인합니다.

## 파일 구성

| 파일 | 역할 |
|---|---|
| `index.html` | 대문 (학생/교사 역할 선택) |
| `subject_home.html` | 과목 선택 (통합사회 / 사회와 문화) |
| `student_quiz.html` | 학생용 시험실 (구글 로그인, 이탈 감지, 답안 제출) |
| `teacher_monitor.html` | 교사용 실시간 대시보드 |
| `vercel.json` | Vercel 정적 배포 설정 |

## 핵심 기능

1. **시험 이탈 감지 🔴** — 학생의 커서가 창을 벗어나거나, 다른 탭/프로그램으로
   전환하면 학생 화면에 빨간 경고 배너가 뜨고, 교사 대시보드에는 빨간불(깜빡임)과
   누적 이탈 횟수가 실시간으로 표시됩니다. 창을 닫으면 "접속 종료"로 표시됩니다.
2. **데이터 누적 📚** — 제출은 `push()`로 저장되어 재제출해도 이전 답안이
   사라지지 않습니다. 교사 화면에는 최신 답안과 총 제출 횟수가 함께 표시됩니다.
3. **실시간 동기화** — Firebase Realtime Database `on('value')`로 새로고침 없이
   표가 자동 갱신됩니다.

## Firebase 데이터 구조

```
sessions/{과목}/{학년-반-번호_이름}
    grade, class, number, name
    focusState: active | away | offline
    awayCount, lastAwayAt, updatedAt

evaluations/{과목}/{학년-반-번호_이름}
    info: { grade, class, number, name, subject }
    submissions/{pushId}:
        answer, status, score, feedback,
        awayCountAtSubmit, submittedAt
```

AI 채점 인프라(Vercel Functions + Gemini)는 `submissions` 하위에서
`status == "채점대기"`인 노드를 감지해 `status / score / feedback`을
업데이트하면 교사 화면에 즉시 반영됩니다.

## 🚀 Vercel 배포 방법

1. 이 폴더를 GitHub 저장소(예: `smart-essay-monitor`)에 업로드합니다.
2. [vercel.com](https://vercel.com)에 구글/깃허브 계정으로 로그인 →
   **Add New → Project** → 방금 만든 저장소를 **Import**.
3. Framework Preset은 **Other**(정적 사이트) 그대로 두고 **Deploy** 클릭.
4. 배포가 끝나면 `https://프로젝트명.vercel.app` 주소가 발급됩니다.

## 🔐 배포 후 Firebase 필수 설정 (안 하면 로그인 에러 발생!)

1. [Firebase Console](https://console.firebase.google.com) → 해당 프로젝트
   → **Authentication → Settings → Authorized domains**
2. **Add domain**을 눌러 발급받은 Vercel 주소를 등록합니다.
   예: `homework260630-1.vercel.app`
   (등록하지 않으면 구글 로그인 팝업에서 `auth/unauthorized-domain` 에러가 납니다.)
3. **Authentication → Sign-in method**에서 **Google** 로그인이
   사용 설정되어 있는지 확인합니다.

## ⚠️ 보안 규칙 권장 설정

현재 구조에서는 데이터베이스 규칙이 열려 있으면 누구나 데이터를 읽고 쓸 수
있습니다. 최소한 로그인한 사용자만 쓰기 가능하도록 규칙을 설정하는 것을
권장합니다. (Firebase Console → Realtime Database → 규칙)

```json
{
  "rules": {
    "sessions":    { ".read": true, ".write": "auth != null" },
    "evaluations": { ".read": true, ".write": "auth != null" }
  }
}
```

## 알아두면 좋은 점

- 학생 페이지와 교사 페이지의 `firebaseConfig`는 반드시 동일해야 합니다.
- 저장 경로가 `evaluations/{과목}/...` 구조이므로, 예전 버전
  (`evaluations/이름_번호`)의 데이터는 새 대시보드에 표시되지 않습니다.
- 이탈 감지는 브라우저 이벤트(blur / visibilitychange / mouseleave) 기반이라
  참고용 지표이며, 완벽한 부정행위 차단 도구는 아닙니다.
