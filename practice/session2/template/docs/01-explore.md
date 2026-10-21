# 탐색 결과: 고객 보관

## 문제
지금 ClientFlow에서 더 이상 관리하지 않는 고객을 정리하는 방법은 삭제뿐이다. 삭제하면 고객 행과 그 고객의 자료 요청이 모두 DB에서 영구히 지워진다. 그래서 거래가 끝난 고객을 목록에서 치우려면 그동안 쌓인 고객과 자료 기록을 함께 잃을 수밖에 없다. 게다가 고객 목록, 대시보드 집계, 마감·지연 목록, 파일 가져오기가 모두 "DB에 있는 고객은 전부 관리 중"이라고 전제하고 동작한다. 요청을 풀려면 고객과 자료 데이터를 지우지 않은 채(제약) 관리하지 않는 고객을 일상 업무 화면과 집계에서 구분해 낼 수 있어야 한다. 이 일은 로그인·권한 없이, 기존 화면 흐름(대시보드 / 고객 관리 두 화면)을 크게 바꾸지 않고 해야 한다.

## 현재 상태

### 요구사항/문서
- 요구사항은 관리하지 않는 고객을 "목록에서 삭제한다"고 정의하고 있다. — `docs/REQUIREMENTS.md:14`
- 서비스 최종 기능도 "고객을 등록, 수정, 삭제한다"로 적혀 있다. — `docs/goals.md:53`
- README의 고객 관리 기능 설명도 "추가, 수정, 삭제"다. — `README.md:12`
- 로그인과 권한 관리는 이번 범위에서 빠져 있다. — `docs/REQUIREMENTS.md:66-67`

### 데이터
- `customers` 테이블에는 id, name, note, created_at, updated_at만 있고, 상태나 보관 여부를 나타내는 컬럼은 없다. — `apps/migrations/0001_create_customers.sql:1-7`
- `material_requests.customer_id`에는 외래키 제약이 없다. 인덱스만 있다. — `apps/migrations/0002_create_material_requests.sql:3,11`
- 자료 요청의 상태는 `not_requested`, `requested`, `submitted` 세 가지로 CHECK 제약이 걸려 있다. — `apps/migrations/0003_add_material_request_status.sql:1`
- 자료 요청 묶음(`request_templates`, `request_template_items`)은 고객과 연결되어 있지 않다. — `apps/migrations/0004_create_request_templates.sql:1-15`
- 마이그레이션은 0001~0004까지 있다. — `apps/migrations/`
- DB는 Cloudflare D1(binding `DB`)이다. — `apps/wrangler.json:7-13`

### API (Worker)
- 고객 삭제(`DELETE /api/customers/:id`)는 해당 고객의 `material_requests`를 먼저 지우고 `customers` 행을 지운다. 둘 다 물리 삭제다. — `apps/worker/index.ts:630-638`
- 고객 목록(`GET /api/customers`)은 조건 없이 모든 고객을 이름순으로 돌려준다. 검색어가 있으면 이름 LIKE 조건만 붙는다. — `apps/worker/index.ts:254-267`
- 대시보드(`GET /api/dashboard`)는 모든 고객과 모든 자료 요청을 읽어 전체 고객 수, 진행/지연/완료 고객 수, 제출·미제출·지연 자료 수를 계산한다. — `apps/worker/index.ts:364-439`
- 마감·지연 자료 목록(`GET /api/material-requests/deadlines`)은 고객 조건 없이 미제출이면서 마감일이 오늘 이전이거나 오늘인 자료를 모두 돌려준다. — `apps/worker/index.ts:441-453`
- 파일 가져오기(`POST /api/customers/import`)는 기존 고객 전체를 이름으로 매칭한다. 같은 이름의 고객이 이미 있으면 새 고객을 만들지 않고 그 고객에게 자료 요청을 추가한다. — `apps/worker/index.ts:287-322`
- 고객별 자료 요청 추가, 묶음 적용, AI 요청문 생성은 고객 id가 존재하는지만 확인한다. — `apps/worker/index.ts:455-460`, `apps/worker/index.ts:499-517`, `apps/worker/index.ts:529-536`
- 고객 수정(`PATCH /api/customers/:id`)은 name, note만 바꾼다. — `apps/worker/index.ts:610-628`

### 화면 (React)
- 화면은 `dashboard`와 `customers` 두 가지이고, 상단 메뉴로 오간다. — `apps/src/App.tsx:39`, `apps/src/App.tsx:740-743`
- 고객 관리 화면의 고객 목록에는 고객마다 "수정"과 "삭제" 버튼이 있다. — `apps/src/App.tsx:817-823`
- 삭제 버튼을 누르면 `window.confirm`으로 확인한 뒤 DELETE를 호출하고, 고객 목록·마감 목록·대시보드를 다시 불러온다. — `apps/src/App.tsx:623-640`
- 고객 목록은 검색어가 바뀔 때마다 `/api/customers?search=`로 다시 불러온다. — `apps/src/App.tsx:240-257`
- 대시보드에는 전체/진행/확인(지연)/완료 필터와 고객 표가 있고, 마감 현황에는 "오늘 마감"과 "지연" 탭이 있다. — `apps/src/App.tsx:746-780`
- 마감 목록에서 "자료 보기"를 누르면 고객 이름으로 `/api/customers`를 검색해 같은 id의 고객을 찾아 연다. — `apps/src/App.tsx:356-370`
- 프런트의 `Customer` 타입은 id, name, note, created_at만 가진다. — `apps/src/App.tsx:5-10`

## 부딪히는 것
- 요구사항 "더 이상 관리하지 않는 고객은 목록에서 삭제한다"는 요청("삭제하지 말고 보관")과 정면으로 부딪힌다. 요구사항 문장 자체를 바꿔야 하는지 정해야 한다. — `docs/REQUIREMENTS.md:14`, `docs/goals.md:53`, `README.md:12`
- 고객 삭제 API는 고객의 자료 요청까지 물리 삭제한다. 제약 "기존 고객과 자료 데이터는 지우지 않는다"와 어긋난다. 이 동작을 남길지는 아직 정해지지 않았다. — `apps/worker/index.ts:630-638`
- 대시보드의 "전체 고객 수"와 진행/지연/완료 집계는 DB의 모든 고객과 자료를 센다. 관리하지 않는 고객이 DB에 남으면 이 숫자에 계속 들어간다. — `apps/worker/index.ts:370-436`, `docs/REQUIREMENTS.md:42-44`
- 마감·지연 목록은 고객 조건 없이 미제출 자료를 모두 보여 준다. 관리하지 않는 고객의 미제출 자료가 계속 "지연"으로 잡힌다. — `apps/worker/index.ts:447-449`, `docs/REQUIREMENTS.md:36-37`
- 고객 목록과 검색은 모든 고객을 돌려준다. 관리하지 않는 고객이 일상 목록에 섞인다. — `apps/worker/index.ts:254-267`, `docs/REQUIREMENTS.md:15`
- 파일 가져오기는 이름이 같은 기존 고객에게 자료를 붙인다. 관리하지 않는 고객과 이름이 같은 행을 가져오면 그 고객에게 자료가 붙는다. — `apps/worker/index.ts:291-305`
- "자료 보기"로 고객을 여는 흐름은 `/api/customers` 검색 결과에 그 고객이 있다고 가정한다. 목록 응답이 바뀌면 이 흐름도 영향을 받는다. — `apps/src/App.tsx:356-370`
- 제약 "기존 화면 흐름을 크게 바꾸지 않는다" 때문에 화면은 현재 두 개(대시보드, 고객 관리)와 고객 목록의 수정/삭제 버튼 구성을 기준으로 봐야 한다. — `apps/src/App.tsx:740-743`, `apps/src/App.tsx:820`

## 확인하지 못한 것
- 운영 D1 DB에 실제로 고객과 자료가 몇 건 있는지, 이미 삭제된 고객이 있었는지는 저장소만으로는 확인할 수 없다.
- 자동 테스트 코드는 저장소에서 찾지 못했다. `npm run check`(타입 검사·빌드)만 README에 안내되어 있다(`README.md:53-55`). 보관 동작을 검증할 기존 테스트가 있는지 확인하지 못했다.
- `apps/seed.sql`의 예시 데이터가 보관 상태를 확인할 만큼 다양한지는 이번에 자세히 보지 않았다.
- 화면 하단(묶음 관리, 고객 상세 영역, `apps/src/App.tsx:825` 이후)이 고객 상태에 따라 달라져야 하는 부분이 있는지는 일부만 확인했다.

## 물어볼 것
- 보관 기능이 생기면 기존 "삭제" 기능은 어떻게 할까요? 없앨까요, 보관과 함께 둘까요, 보관된 고객에게만 허용할까요? (제약 "기존 고객과 자료 데이터는 지우지 않는다"가 사용자가 직접 누르는 삭제까지 막는다는 뜻인지 확인이 필요합니다.)
- 보관한 고객을 다시 "관리 중"으로 되돌리는 기능(보관 해제)이 필요할까요?
- 보관한 고객을 볼 수 있어야 할까요? 그렇다면 어디서, 어떤 방식으로 볼까요? (예: 고객 관리 화면 안에서 따로 보기)
- 보관한 고객과 그 자료를 대시보드 집계(전체 고객 수, 진행/지연/완료, 제출·미제출·지연 자료 수)와 오늘 마감·지연 목록에서 빼야 할까요?
- 보관한 고객의 자료를 수정하거나, 자료를 추가하거나, 묶음을 적용하거나, AI 요청문을 만들 수 있어야 할까요? 아니면 읽기만 가능하면 될까요?
- 고객 이름 검색에 보관한 고객도 나와야 할까요?
- 파일 가져오기에서 보관한 고객과 이름이 같은 행이 있으면 어떻게 할까요? (보관 고객에게 붙이기 / 새 고객 만들기 / 오류로 알리기)
- 보관한 날짜나 보관 사유를 기록해야 할까요?
- 요구사항 문서(`docs/REQUIREMENTS.md`)의 "목록에서 삭제한다" 문장을 이번 작업에서 "보관한다"로 바꿀까요?
