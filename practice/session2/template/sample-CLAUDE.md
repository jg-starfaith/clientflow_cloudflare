# 작업 지침
- 프로젝트 이름: clientflow_cloudflare
- DB 구조를 바꿀 때는 `apps/migrations/`에 SQL 파일을 새로 추가한다.
- 사용 기술은 React + TypeScript, Vite, Cloudflare Workers, Cloudflare D1이다.
- 로컬 실행은 `apps` 폴더에서 `npm install`, `npx wrangler d1 migrations apply DB --local`, `npm run dev` 순서로 한다.
- 요구사항과 작업 문서는 `docs/`에 둔다.
- 작업이 끝나면 `apps` 폴더에서 `npm run check`를 실행한다.

## 기록 규칙
- 언제: 결정이 생겼을 때, 작업을 마칠 때
- 어디에: 결정은 `docs/decision.md`에 한 줄씩 추가하고, 진행 상황은 `docs/handoff.md`를 덮어쓴다
- 무엇을: 결정과 그 이유 / 이번에 한 일과 다음에 할 일. 과정 전체는 적지 않는다

## 읽기 규칙
- 작업을 시작할 때 `docs/handoff.md`와 `docs/decision.md`를 먼저 읽는다.
