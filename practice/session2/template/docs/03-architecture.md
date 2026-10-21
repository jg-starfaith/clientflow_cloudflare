# 구현 설계: 고객 보관

> 입력: `docs/02-prd.md`(PRD), `docs/decision.md`(정책 결정, 2026-10-21), `docs/01-explore.md`, 현재 코드(`apps/worker/index.ts`, `apps/src/App.tsx`, `apps/migrations/`)
> PRD의 A1~A9는 기본안(가정)이고, 확정 정책은 `docs/decision.md`다. 둘이 다르면 decision.md를 따랐다. 어떻게 맞췄는지는 바로 아래 "결정 반영으로 조정한 범위와 완료 기준"에 적었다.
> 줄 번호는 이 문서를 쓴 시점(커밋 `4b17be7`)의 파일 기준이다.

## 결정 반영으로 조정한 범위와 완료 기준

### PRD와 decision.md가 어긋난 부분과 맞춘 방법
| PRD 줄 | PRD 기본안 | decision.md | 이 설계에서 맞춘 방법 |
|---|---|---|---|
| `docs/02-prd.md:11-12`, `:26`, `:41` (A1) | 삭제 버튼 자리를 보관으로 바꾸고, 화면에서 영구 삭제는 제공하지 않는다. "삭제 버튼은 보이지 않는다" | 삭제 버튼은 그대로 두고 보관을 함께 둔다. 보관 고객은 되돌린 뒤에만 삭제할 수 있다. | 관리 중 목록에 "수정 / 보관 / 삭제"를 함께 둔다. 보관 목록에는 "되돌리기"만 둔다. 서버도 보관 고객 삭제를 거부한다. 기준 B1을 고쳤다. "휴지통·일괄 삭제"는 계속 범위 밖이다. |
| `docs/02-prd.md:38` (데이터 보존) | "화면에서 어떤 동작을 해도 고객이나 자료 요청이 사라지지 않는다" | A1에 따라 사용자가 직접 누르는 삭제는 남는다. | 기준 D3을 "보관·되돌리기로는 아무것도 사라지지 않는다. 삭제는 관리 중 고객에서 확인을 거친 경우에만 지금처럼 지운다"로 좁혔다. |
| `docs/02-prd.md:14`, `:47` (A3) | "관리 중 / 보관" 목록을 바꿔 가며 본다 | 보관 고객은 고객 관리 화면의 "보관 고객 보기"에서만 본다. 기본 목록에는 나오지 않는다. | 서로 맞는다. 고객 목록 제목 옆에 "보관 고객 보기" 버튼을 두고, 누르면 같은 자리에서 보관 목록으로 바뀐다(되돌아갈 때는 "관리 중 고객 보기"). |
| `docs/02-prd.md:16`, `:59-60` (A5) | 자료 추가·수정, 묶음 적용, AI 요청문 생성만 막는다. 고객 정보 수정은 정하지 않았다. | 요청문 생성, 자료 추가·수정·삭제, 묶음 적용, 상태 변경, 고객 정보 수정, 고객 삭제를 모두 막는다. 보기와 되돌리기만 된다. | 기준 P1에 자료 삭제, 상태 변경, 고객 정보 수정, 고객 삭제를 더했다. "허용하지 않는다"를 화면에서 버튼을 숨기는 것만으로 처리하지 않도록 서버 차단 기준 P3을 더했다(아래 "서버 쪽 차단 지점"). |
| `docs/02-prd.md:18`, `:67-70` (A7) | 보관 고객과 이름이 같은 줄만 빼고 나머지 줄은 가져온다. | 그런 줄이 하나라도 있으면(같은 이름의 관리 중 고객이 있어도) 파일 전체 가져오기를 막고, 어느 이름인지 알려 준다. 고객 추가 화면은 바꾸지 않는다. | 기준 I3·I4를 바꿨다. `:70` "다른 행은 지금처럼 가져와진다"는 뒤집혀서 "그 파일에서는 어떤 줄도 들어가지 않는다"가 된다. 고객 추가(`POST /api/customers`)는 바꾸지 않는다. |
| `docs/02-prd.md:19`, `:48` (A8) | 보관한 날짜를 기록하고 보관 목록에 보여 준다. | 보관 시각과 사유는 요구하지 않고, 화면에 보여 주지 않는다. | 보관 시각 컬럼을 만들지 않는다. 기준 R2에서 "보관한 날짜가 보인다"를 뺐다. |
| `docs/02-prd.md:73` (A9) | 문서에 "삭제한다" 설명이 남아 있지 않다. | REQUIREMENTS.md·goals.md·README.md의 "삭제한다" 설명을 보관 기능에 맞게 고친다. | 삭제 기능은 남으므로(A1) "삭제" 낱말을 다 지우지 않는다. 기준 M1을 "관리하지 않는 고객은 삭제한다는 설명이 없고, 보관·되돌리기와 '되돌린 뒤 삭제'가 설명되어 있다"로 바꿨다. |
| A2, A4, A6 | - | PRD 기본안과 같다. A4에서 "보관 고객 수는 대시보드에 따로 보여 주지 않는다"가 확정되었다. | 그대로 따른다. |

### 이 설계가 쓰는 완료 기준 (결정 반영본)
- **D1** 배포 직후, 배포 전에 있던 고객이 모두 "관리 중" 목록에 보이고, 고객마다 자료 요청 개수와 내용이 배포 전과 같다.
- **D2** 보관한 고객을 보관 목록에서 열거나 되돌린 뒤 보아도 이름, 메모, 자료 요청 개수, 각 자료의 상태·마감일이 보관 전과 같다.
- **D3** (조정) 보관·되돌리기로는 고객이나 자료 요청이 사라지지 않는다. 보관 전후로 "관리 중 수 + 보관 수"가 같다. 삭제는 관리 중 고객에서 확인을 거친 경우에만 지금처럼 지운다.
- **B1** (조정) 관리 중 목록에서 고객마다 "수정", "보관", "삭제" 버튼이 보인다.
- **B2** "보관"을 누르면 확인 창이 뜨고, 취소하면 아무것도 바뀌지 않는다.
- **B3** 확인하면 그 고객이 관리 중 목록에서 사라지고 보관 목록에 나타난다. 새로 고쳐도 그대로다.
- **B4** 보관 직후 대시보드 숫자와 마감·지연 목록을 다시 불러온다.
- **R1** 고객 관리 화면 안에서 "보관 고객 보기"로 관리 중과 보관 목록을 바꿔 본다. 상단 메뉴는 대시보드, 고객 관리 두 개 그대로다.
- **R2** (조정) 보관 목록에는 보관 고객만 나온다. 보관 날짜나 사유는 보이지 않는다.
- **R3** 보관 목록에서 고객을 열면 그 고객의 자료 요청 목록을 볼 수 있다.
- **R4** 보관 목록에서 "되돌리기"를 누르면 관리 중 목록으로 돌아온다. 자료는 보관 전과 같고, 대시보드 집계와 마감·지연 목록에 다시 잡힌다.
- **G1** 관리 중 고객 3명 가운데 1명을 보관하면 대시보드 전체 고객 수가 1 줄어든다.
- **G2** 보관 고객은 대시보드 고객 표에 어떤 필터(전체/진행/확인/완료)에서도 나오지 않는다.
- **G3** 보관 고객의 자료는 대시보드 요약의 자료 수(제출·미제출·지연)와 "지연"·"오늘 마감" 탭에서 빠진다.
- **G4** 마감·지연 목록에서 "자료 보기"를 누르면 지금처럼 그 관리 중 고객의 자료가 열린다.
- **P1** (확장) 보관 고객을 열면 고객 정보 수정, 고객 삭제, 자료 추가·수정·삭제, 상태 변경, 묶음 적용, AI 요청문 생성을 화면에서 쓸 수 없다(보이지 않거나 눌리지 않는다).
- **P2** 되돌리면 같은 고객에서 위 동작을 다시 쓸 수 있다.
- **P3** (decision.md A5에서 추가) 화면을 거치지 않고 API를 직접 불러도 위 동작이 보관 고객에게는 거부되고 데이터가 바뀌지 않는다.
- **S1** 관리 중 목록에서 보관 고객 이름으로 검색하면 나오지 않는다.
- **S2** 보관 목록에서 같은 이름으로 검색하면 나온다.
- **I1** 보관 고객과 이름이 같은 줄이 든 파일을 가져와도 보관 고객의 자료 요청 개수가 늘지 않는다.
- **I2** 그 이름으로 새 관리 중 고객이 생기지 않는다.
- **I3** (조정) 화면에 "파일 전체를 가져오지 않았다"는 안내와 해당 이름이 보인다.
- **I4** (조정) 그 파일의 다른 줄(새 이름, 관리 중 고객 이름)도 가져오지 않는다. 같은 이름의 관리 중 고객이 있어도 막힌다. 보관 고객 이름이 없는 파일은 지금처럼 가져온다.
- **M1** (조정) REQUIREMENTS.md, goals.md, README.md에 "관리하지 않는 고객은 삭제한다"는 설명이 없고, 보관·되돌리기와 "보관 고객은 되돌린 뒤 삭제"가 설명되어 있다.

## 바꿀 파일
| 파일 경로 | 바꿀 내용 | 연결된 완료 기준 |
|---|---|---|
| `apps/migrations/0005_add_customer_archived.sql` (새 파일) | `customers`에 `archived` 컬럼 추가(아래 "DB 변경") | D1, B3, R2, S1, S2 |
| `apps/worker/index.ts:1-7` | `Customer` 타입에 `archived: number`(0 = 관리 중, 1 = 보관) 추가 | B3, R2, P1 |
| `apps/worker/index.ts:90-97` 부근 | (1) 보관 고객 거부 응답 함수 추가: 409, `"보관한 고객입니다. 되돌린 뒤 다시 시도해 주세요."` (2) 경로 함수 `customerArchiveId`(`/api/customers/:id/archive`), `customerRestoreId`(`/api/customers/:id/restore`) 추가 (3) 고객 상태 조회 함수(`SELECT id, archived FROM customers WHERE id = ?`)와 자료 요청 id로 그 고객의 보관 여부를 찾는 함수(`material_requests LEFT JOIN customers`) 추가 | B3, R4, P3 |
| `apps/worker/index.ts:254-267` (`GET /api/customers`) | `archived` 쿼리 파라미터를 받는다. `archived=1`이면 `WHERE archived = 1`, 없거나 다른 값이면 `WHERE archived = 0`. 검색어가 있으면 `AND name LIKE ?`. 정렬은 그대로. SELECT 목록에 `archived` 추가 | D1, B3, R1, R2, S1, S2, G4 |
| `apps/worker/index.ts:278-282` (`POST /api/customers`의 재조회) | SELECT에 `archived` 추가(새 고객은 기본값 0). INSERT 자체와 고객 추가 동작은 바꾸지 않는다(decision A7 "고객 추가 화면은 바꾸지 않는다") | B1 (새 고객이 관리 중 목록에 나와 보관·삭제 버튼을 가짐) |
| `apps/worker/index.ts:287-322` (`POST /api/customers/import`) | 입력 검사(288-289) 직후, 어떤 INSERT보다 먼저 `SELECT DISTINCT name FROM customers WHERE archived = 1`로 보관 고객 이름을 읽고, 파일 줄의 `customer_name`과 정확히 같은 이름을 모은다. 하나라도 있으면 409와 `{ error, archived_names }`를 돌려주고 끝낸다. 오류 문구 예: `"보관한 고객과 이름이 같은 줄이 있어 파일 전체를 가져오지 않았습니다: A, B"`. 이어서 291의 기존 고객 조회에 `WHERE archived = 0`을 붙인다(보관 고객에게 자료가 붙지 않게 하는 이중 안전장치) | I1, I2, I3, I4 |
| `apps/worker/index.ts:370-373` (`GET /api/dashboard`) | 고객 조회에 `WHERE archived = 0`과 SELECT `archived` 추가. 자료 조회에 `WHERE NOT EXISTS (SELECT 1 FROM customers WHERE customers.id = material_requests.customer_id AND customers.archived = 1)` 추가. 381-436의 집계 로직은 그대로 둔다(입력에서 빠지므로 자동으로 빠짐) | G1, G2, G3, B4, R4 |
| `apps/worker/index.ts:447-449` (`GET /api/material-requests/deadlines`) | 이미 `JOIN customers`가 있으므로 WHERE에 `AND customers.archived = 0` 추가 | G3, G4, B4, R4 |
| `apps/worker/index.ts:455-460` (AI 요청문 생성) | 457의 고객 조회에 `archived` 포함. 보관이면 AI 호출 전에 409 | P1, P3 |
| `apps/worker/index.ts:504-509` (묶음 적용) | 505의 고객 조회를 `SELECT id, archived`로 바꾸고, 보관이면 509 다음에 409 | P1, P3 |
| `apps/worker/index.ts:519-527` (`GET /api/customers/:id/requests`) | 바꾸지 않는다. 보관 고객 자료 보기가 그대로 되어야 한다 | R3, D2 (변경 없음 확인용) |
| `apps/worker/index.ts:529-536` (자료 추가) | 533의 고객 조회를 `SELECT id, archived`로 바꾸고, 보관이면 536 다음에 409 | P1, P3 |
| `apps/worker/index.ts:552-562` (자료 상태 변경) | 558 UPDATE의 WHERE에 `AND NOT EXISTS (SELECT 1 FROM customers WHERE customers.id = material_requests.customer_id AND customers.archived = 1)` 추가. `changes`가 0이면 자료 요청 id로 다시 조회해 없으면 404, 보관 고객 자료면 409 | P1, P3 |
| `apps/worker/index.ts:572-582` (자료 수정) | 578 UPDATE에 위와 같은 조건 추가, 같은 방식으로 404/409 구분 | P1, P3 |
| `apps/worker/index.ts:592-596` (자료 삭제) | 593 DELETE에 위와 같은 조건 추가, 같은 방식으로 404/409 구분 | P1, P3, D3 |
| `apps/worker/index.ts:610-620` (`PATCH /api/customers/:id`) | 616 UPDATE에 `AND archived = 0` 추가. `changes`가 0이면 고객 상태 조회로 404/409 구분. 623 재조회 SELECT에 `archived` 추가 | P1, P3 |
| `apps/worker/index.ts:630-635` (`DELETE /api/customers/:id`) | batch 두 문장에 조건을 붙인다. 632: `DELETE FROM material_requests WHERE customer_id = ? AND NOT EXISTS (SELECT 1 FROM customers WHERE id = ? AND archived = 1)`. 633: `DELETE FROM customers WHERE id = ? AND archived = 0`. `result[1].meta.changes`가 0이면 고객 상태 조회로 404/409 구분. batch는 한 트랜잭션이므로 "자료만 지워지고 고객은 남는" 경우가 생기지 않는다 | P1, P3, D3 |
| `apps/worker/index.ts:609` 부근 (새 라우트 2개, `customerId` 처리 앞) | `POST /api/customers/:id/archive`: `UPDATE customers SET archived = 1 WHERE id = ? AND archived = 0`. `POST /api/customers/:id/restore`: `UPDATE customers SET archived = 0 WHERE id = ? AND archived = 1`. `changes`가 0이면 없으면 404, 이미 그 상태면 409. 성공하면 `{ customer }`(archived 포함) 반환. `material_requests`와 `updated_at`은 건드리지 않는다 | B2, B3, R4, D2, D3 |
| `apps/src/App.tsx:5-10` | 프런트 `Customer` 타입에 `archived: number` 추가(`DashboardCustomer`는 이를 상속) | P1, P2, R3 |
| `apps/src/App.tsx:189-190` 부근 | 새 상태 `customerListMode: "active" \| "archived"`(기본 `"active"`) | R1 |
| `apps/src/App.tsx:240-257` (`loadCustomers`, effect) | 요청 URL에 `customerListMode === "archived"`일 때 `&archived=1`을 붙인다. effect 의존성을 `[search, customerListMode]`로 바꾼다. 검색어는 지금 보는 목록에만 적용된다 | R1, R2, S1, S2, B3 |
| `apps/src/App.tsx:351-354` (`openCustomerWorkspace`) | `setCustomerListMode("active")`를 함께 부른다. 보관 목록을 보던 중 대시보드에서 "자료 보기"로 들어와도 관리 중 목록이 보이게 한다. 356-371의 `openCustomerWorkspaceById`는 서버가 기본으로 관리 중만 돌려주므로 그대로 동작한다 | G4 |
| `apps/src/App.tsx:615-621` (`startEditing`) | `customer.archived === 1`이면 아무것도 하지 않는다(화면 쪽 이중 안전장치) | P1 |
| `apps/src/App.tsx:640` 다음 (새 함수 2개) | `archiveCustomer(customer)`: `window.confirm("'이름' 고객을 보관할까요? 되돌리기 전까지 수정·자료 관리를 할 수 없습니다.")` → `POST /api/customers/:id/archive` → 편집 중이면 `resetForm()`, 선택된 고객이면 선택 해제와 자료 폼 정리(623-640 삭제와 같은 정리) → `loadCustomers()`, `loadDeadlineRequests()`, `loadDashboard()` 다시 불러오기. `restoreCustomer(customer)`: `POST /api/customers/:id/restore` → 선택된 고객이면 응답의 고객으로 `selectedCustomer`를 바꿔(archived 0) 버튼이 바로 살아나게 함 → 같은 세 가지 다시 불러오기 | B2, B3, B4, R4, P2, D2 |
| `apps/src/App.tsx:785` (고객 목록 제목) | 제목을 모드에 따라 "고객 목록" / "보관 고객"으로 바꾸고, 오른쪽에 `text-button` 하나를 둔다: 관리 중일 때 "보관 고객 보기", 보관일 때 "관리 중 고객 보기". 누르면 모드를 바꾸고 열린 고객 폼은 닫는다(`resetForm()`, `setShowCustomerForm(false)`). 786의 고객 추가·파일로 추가·템플릿 버튼은 그대로 둔다 | R1 |
| `apps/src/App.tsx:807` (가져오기 안내 문구) | 문장 끝에 "보관한 고객과 같은 이름이 있으면 파일 전체를 가져오지 않습니다."를 더한다. 실패 안내 표시는 562/570이 이미 서버 `error`를 `importMessage`로 보여 주고 미리보기 줄도 지우지 않으므로 그대로 쓴다 | I3 |
| `apps/src/App.tsx:816` (빈 목록 문구) | 보관 모드에서 검색어가 없으면 "보관한 고객이 없습니다."를 보인다 | R2, S2 |
| `apps/src/App.tsx:818-822` (고객 목록 버튼) | 관리 중 모드: 이름, "수정", "보관"(`small-button`), "삭제"(지금 그대로). 보관 모드: 이름과 "되돌리기"(`small-button`)만. 보관 날짜 같은 추가 표시는 넣지 않는다 | B1, R2, R4, P1 |
| `apps/src/App.tsx:827-860` (고객 상세) | `const isSelectedArchived = selectedCustomer?.archived === 1`을 두고, 보관이면: 830 "수정" 숨김, 832 다음에 안내 "보관한 고객입니다. 보기만 할 수 있습니다." 와 "되돌리기" 버튼(기존 `ai-guide`·`small-button` 클래스 재사용), 836 "추가"·"일괄 추가" 숨김, 837 "AI 요청문 만들기" 숨김, 838 자료 폼 렌더 안 함, 853 자료별 "수정"·"삭제" 숨김, 853 상태 `select`는 `disabled`. 자료 목록과 자료 이름 검색(837 입력란)은 그대로 둔다 | P1, P2, R3, D2 |
| `apps/src/App.tsx:877` (묶음 "적용" 버튼) | `disabled` 조건에 `selectedCustomer?.archived === 1` 추가. 묶음 저장·수정·삭제는 고객과 무관하므로 그대로 둔다(PRD 범위 밖) | P1 |
| `docs/REQUIREMENTS.md:14` | "더 이상 관리하지 않는 고객은 보관한다. 보관한 고객은 기본 목록·대시보드·마감 목록에서 빠지고, '보관 고객 보기'에서 보고 되돌릴 수 있다. 보관 고객을 지우려면 되돌린 뒤 삭제한다." 취지로 바꾼다 | M1 |
| `docs/goals.md:53` | "고객을 등록, 수정, 보관(되돌리기), 삭제한다." 취지로 바꾼다 | M1 |
| `README.md:12` | "고객 이름과 메모를 추가, 수정, 삭제합니다." 다음에 "관리하지 않는 고객은 보관하고, '보관 고객 보기'에서 되돌릴 수 있습니다. 보관 고객은 보기만 할 수 있고, 되돌린 뒤 삭제할 수 있습니다." 취지를 더한다 | M1 |

바꾸지 않는 파일: `apps/wrangler.json`(설정 변경 없음), `apps/seed.sql`(INSERT가 컬럼을 지정하므로 새 컬럼은 기본값 0으로 들어간다), `apps/src/App.css`(기존 클래스 재사용), 묶음 관련 API(`apps/worker/index.ts:324-362`, `:601-608`).

### API 변경 요약
| API | 변경 | 구버전 호출 시 |
|---|---|---|
| `GET /api/customers` | `?archived=1` 추가. 기본은 관리 중만. 응답 고객에 `archived` 추가 | 파라미터 없이 부르면 관리 중만 받는다. 필드가 하나 늘 뿐이다 |
| `POST /api/customers/:id/archive` (새) | 보관. 200 `{ customer }` / 404 / 409(이미 보관) | - |
| `POST /api/customers/:id/restore` (새) | 되돌리기. 200 `{ customer }` / 404 / 409(이미 관리 중) | - |
| `PATCH /api/customers/:id`, `DELETE /api/customers/:id` | 보관 고객이면 409 | 구버전 화면은 보관 고객을 목록에 보이지 않으므로 거의 부를 일이 없다 |
| `POST /api/customers/:id/requests`, `.../request-templates/:tid/apply`, `.../ai-request-message` | 보관 고객이면 409 | 같은 이유 |
| `PATCH /api/material-requests/:id`, `PATCH .../status`, `DELETE /api/material-requests/:id` | 그 자료의 고객이 보관이면 409 | 같은 이유 |
| `POST /api/customers/import` | 보관 고객 이름이 하나라도 있으면 409 `{ error, archived_names }`, 아무것도 쓰지 않음 | 구버전 화면도 `data.error`를 보여 주므로(`App.tsx:562,570`) 안내가 나온다 |
| `GET /api/dashboard`, `GET /api/material-requests/deadlines` | 보관 고객과 그 자료 제외. 대시보드 고객에 `archived` 추가 | 응답 모양은 같다 |

### 서버 쪽 차단 지점 (P3)
decision A5의 "허용하지 않는다"를 화면만으로 지키면, 열어 둔 구버전 탭이나 직접 호출로 보관 고객 데이터가 바뀔 수 있다. 그래서 고객을 바꾸거나 고객에게 자료를 붙이는 모든 쓰기 경로에서 서버가 막는다.
1. 고객 id로 들어오는 쓰기: 고객 수정 `index.ts:616`, 고객 삭제 `:632-633`, 자료 추가 `:533`, 묶음 적용 `:505`, AI 요청문 `:457`.
2. 자료 요청 id로 들어오는 쓰기(고객 id가 경로에 없어서 빠뜨리기 쉬움): 상태 변경 `:558`, 자료 수정 `:578`, 자료 삭제 `:593`.
3. 가져오기 `:291` 앞.
- 수정·삭제는 UPDATE/DELETE 문의 WHERE에 조건을 넣어 확인과 쓰기를 한 문장으로 처리한다. 추가·적용·AI는 지금 코드처럼 고객을 먼저 조회한 뒤 쓴다.
- 자료 요청 쪽 조건은 `customer_id IN (관리 중 고객)`이 아니라 `NOT EXISTS (보관 고객)`으로 쓴다. `material_requests.customer_id`에 외래키가 없어서(`apps/migrations/0002_create_material_requests.sql:3`) 고객 행이 없는 자료가 있을 수 있는데, 그런 자료를 다루는 방식은 지금과 같게 두기 위해서다. 대시보드 자료 조회(`:372`)도 같은 이유로 같은 방식을 쓴다.
- 읽기(`GET /api/customers/:id/requests`, `GET /api/customers?archived=1`)는 막지 않는다.

## DB 변경
- `customers`에 보관 여부 컬럼 추가: `ALTER TABLE customers ADD COLUMN archived INTEGER NOT NULL DEFAULT 0 CHECK (archived IN (0, 1));` — `apps/migrations/0005_add_customer_archived.sql` — D1, B3, R2, S1, S2
  - 기존 행은 모두 기본값 0(관리 중)이 된다. 따로 데이터를 옮기는 작업은 없다. 고객 행과 자료 행은 하나도 지우거나 고치지 않는다(D1, 요청서 제약).
  - 보관 시각·사유 컬럼은 만들지 않는다(decision A8).
  - 인덱스는 추가하지 않는다. 고객 수가 적고 지금도 고객 조회는 전체를 읽는다.
  - 형식은 `0003_add_material_request_status.sql:1`(ADD COLUMN + NOT NULL DEFAULT + CHECK)과 같다.

## 적용 순서
1. (데이터 확인) 운영 DB의 배포 전 건수를 기록한다: `npx wrangler d1 execute DB --remote --command "SELECT (SELECT COUNT(*) FROM customers) AS customers, (SELECT COUNT(*) FROM material_requests) AS requests"`. 이어서 `npx wrangler d1 export DB --remote --output=<백업 파일>`로 백업을 받고, `npx wrangler d1 time-travel info DB`로 복원 기준 시점을 적어 둔다. (D1 확인용)
2. (스키마) `apps/migrations/0005_add_customer_archived.sql`을 만들고 로컬에 적용한다: `npx wrangler d1 migrations apply DB --local`.
3. (코드) `apps/worker/index.ts`를 고친다: 타입·헬퍼 → 조회 필터(목록, 대시보드, 마감) → 보관/되돌리기 라우트 → 쓰기 차단 → 가져오기 차단.
4. (코드) `apps/src/App.tsx`를 고친다: 타입·상태 → 목록 모드와 전환 버튼 → 보관/되돌리기 함수와 버튼 → 상세 영역 읽기 전용 → 묶음 적용 비활성 → 가져오기 안내 문구.
5. (문서) `docs/REQUIREMENTS.md:14`, `docs/goals.md:53`, `README.md:12`를 고친다.
6. (확인) `apps`에서 `npm run check`, 그다음 아래 "확인 방법"의 로컬 시나리오를 돈다.
7. (스키마, 운영) `npx wrangler d1 migrations apply DB --remote`. **코드 배포보다 반드시 먼저 한다.**
8. (코드, 운영) `npm run deploy`. 저장소가 Cloudflare 빌드와 연결되어 있어(커밋 `860834f`) main에 push하면 자동 배포될 수 있으니, push·merge도 7번 뒤에 한다.
9. (데이터 확인, 운영) 1번과 같은 건수 조회를 다시 하고, `SELECT COUNT(*) FROM customers WHERE archived = 0`이 전체 고객 수와 같은지 확인한다(D1).

### 구버전 호환
- **새 스키마 + 구버전 코드(7번과 8번 사이, 또는 코드만 되돌렸을 때)**: 구버전 코드는 SELECT 컬럼을 하나하나 적고(`index.ts:258,263,279,371,457,623`) INSERT도 `(name, note)`만 넣으므로(`:274,300`) 새 컬럼이 있어도 그대로 돈다. 새 고객은 기본값 0으로 들어간다. 다만 이미 보관한 고객이 있으면 구버전은 그 고객을 일반 고객처럼 목록·대시보드에 보여 주고 수정·삭제도 허용한다.
- **새 코드 + 옛 스키마(순서를 어겼을 때)**: `archived`를 읽는 모든 쿼리가 실패해 고객 목록, 대시보드, 마감 목록이 모두 500이 된다. 그래서 7번 → 8번 순서를 지킨다. 자동 배포 때문에 순서가 뒤집히지 않게 한다.
- **새 서버 + 열어 둔 구버전 화면 탭**: 구버전 화면은 `archived` 파라미터 없이 목록을 불러 관리 중 고객만 받는다. 다른 탭에서 보관한 고객을 구버전 탭에서 고치려 하면 서버가 409를 주고, 화면은 기존 오류 표시(`App.tsx:603,664,714`)로 문구를 보여 준다. 자료 삭제(`:691`)와 고객 삭제(`:629`)는 서버 문구 대신 기존 고정 문구가 나온다. 데이터는 바뀌지 않는다.
- 프런트와 Worker는 한 번의 `wrangler deploy`로 같이 올라가므로(`apps/wrangler.json:4,21-23`) 두 버전이 섞이는 경우는 열린 탭뿐이다.

### 실패 시 복구
- **2번·7번 마이그레이션 실패**: 한 문장짜리 ADD COLUMN이라 실패하면 적용 기록(`d1_migrations`)에 남지 않는다. 원인을 고치고 다시 적용한다. 코드는 아직 배포 전이라 서비스 영향이 없다.
- **8번 배포 실패 또는 배포 후 문제 발견**: `npx wrangler rollback`으로 이전 Worker 버전으로 되돌린다. 컬럼은 남겨 둔다(구버전과 호환). 되돌린 동안에는 보관 고객이 일반 고객처럼 보이고 삭제도 되므로, 그동안 보관 고객을 삭제하지 말라고 사용자에게 알린다. 고친 코드를 다시 배포하면 보관 상태(`archived = 1`)는 그대로 살아 있다.
- **컬럼까지 없애야 할 때(권장하지 않음)**: 컬럼을 쓰지 않는 코드를 먼저 배포한 뒤, 새 마이그레이션(`0006`)으로 `ALTER TABLE customers DROP COLUMN archived`를 적용한다. 0005 파일을 고치거나 지우지 않는다.
- **데이터가 잘못된 경우(최후 수단)**: 1번에서 적어 둔 시점으로 `npx wrangler d1 time-travel restore DB --timestamp=<시점>` 또는 백업 파일로 복원한다. 그 시점 이후의 입력이 사라지므로 사람이 결정한 뒤에만 한다.
- **사용자가 잘못 보관한 경우**: 보관 목록에서 "되돌리기"로 바로잡는다. 데이터는 그대로다(D2).

## Side-effect와 위험도
| 위험도 | Side-effect | 영향받는 곳 |
|---|---|---|
| 상 | 마이그레이션보다 코드가 먼저 올라가면(자동 배포 포함) 고객 목록·대시보드·마감 목록이 모두 500이 된다 | `apps/worker/index.ts:254-267,370-373,447-449`, Cloudflare 빌드 연동 |
| 상 | 자료 요청 id로 들어오는 쓰기(상태 변경·수정·삭제)는 경로에 고객 id가 없어 차단을 빠뜨리기 쉽다. 빠뜨리면 보관 고객 자료가 바뀌고 decision A5와 P3이 깨진다 | `apps/worker/index.ts:558,578,593` |
| 상 | 고객 삭제에 보관 조건을 빠뜨리거나 batch 두 문장 조건이 어긋나면, 보관 고객과 그 자료가 영구히 지워지거나 "자료만 지워지고 고객은 남는" 상태가 생긴다 | `apps/worker/index.ts:630-635` |
| 중 | 가져오기 동작이 바뀐다. 지금은 같은 이름이면 자료를 붙였지만, 보관 고객 이름이 하나라도 있으면 같은 이름의 관리 중 고객이 있어도 파일 전체가 거부된다. 자주 쓰던 파일이 갑자기 안 들어갈 수 있다 | `apps/worker/index.ts:287-322`, `apps/src/App.tsx:550-574,807` |
| 중 | 코드를 되돌리면 보관 고객이 다시 일반 고객처럼 보이고 삭제도 된다 | 구버전 `apps/worker/index.ts:630-638` |
| 중 | 대시보드 숫자(전체 고객 수, 제출·미제출·지연 자료 수)가 보관한 만큼 줄어든다. 의도한 변화지만 사용자는 "숫자가 줄었다"로 느낄 수 있다 | `apps/worker/index.ts:370-436`, `apps/src/App.tsx:752-755,769-770` |
| 하 | 보관 목록을 보던 중 대시보드 "자료 보기"로 이동하면 목록이 관리 중으로 바뀐다 | `apps/src/App.tsx:351-354` |
| 하 | 보관 목록 모드에서도 "고객 추가", "파일로 추가"가 그대로 보인다. 거기서 추가한 고객은 관리 중이라 지금 보는 보관 목록에는 나타나지 않는다 | `apps/src/App.tsx:786` |
| 하 | 되돌리기 결과 같은 이름의 관리 중 고객이 둘이 될 수 있다. 이후 가져오기는 id가 큰 쪽에 자료를 붙인다(지금 동작, `index.ts:291-295`). 같은 이름 고객은 지금도 고객 추가로 만들 수 있다 | `apps/worker/index.ts:291-295` |
| 하 | 자료 추가·묶음 적용·AI 요청문은 "조회 후 쓰기"라서, 조회와 쓰기 사이에 다른 탭에서 보관하면 한 번은 통과할 수 있다. 로그인 없이 혼자 쓰는 서비스라 가능성이 낮다 | `apps/worker/index.ts:457,505,533` |
| 하 | 보관·되돌리기는 `updated_at`을 바꾸지 않는다. 화면에 `updated_at`을 보여 주는 곳은 없다 | `apps/worker/index.ts` 새 라우트 |
| 하 | 가져오기 실패 문구가 성공 문구와 같은 스타일(`.import-message`, 파란 굵은 글씨)로 보인다. 지금 다른 가져오기 오류도 같다 | `apps/src/App.tsx:812`, `apps/src/App.css:89` |
| 하 | 이름 비교는 지금 가져오기처럼 정확히 같은 문자열만 같은 이름으로 본다(대소문자·공백 다르면 다른 이름). 고객 이름 검색(LIKE)과는 기준이 다르다 | `apps/worker/index.ts:293-299` |

## 확인 방법
자동 테스트가 없으므로(`README.md:49-55`, PRD `:115`) 타입·빌드 검사, 로컬 화면 확인, API 직접 호출, DB 건수 조회로 확인한다.

**공통 준비(로컬)**: `apps`에서 `.wrangler/state`를 지운 뒤 `npx wrangler d1 migrations apply DB --local` → `npx wrangler d1 execute DB --local --file=seed.sql` → `npm run dev`. `seed.sql`만으로 필요한 경우가 갖춰진다: 국민무역(id 12)은 3일 지난 미제출 자료와 오늘 마감 미제출 자료가 있고, ABC상사(23)는 지연 자료, 알파무역(35)은 지연 자료, 좋은유통(33) 등은 자료가 없다. 같은 이름 확인용 파일은 CSV를 따로 만든다.

1. **정적 검사**: `npm run check`가 통과한다.
2. **D1**: 마이그레이션 전후 `SELECT COUNT(*) FROM customers`, `SELECT customer_id, COUNT(*) FROM material_requests GROUP BY customer_id`가 같고, 화면 관리 중 목록에 고객 11명이 모두 보인다.
3. **B1~B4, G1~G3**: 대시보드 숫자를 적어 둔다 → 국민무역 "보관" → 확인 창에서 취소(변화 없음) → 다시 눌러 확인 → 관리 중 목록에서 사라짐, "보관 고객 보기"에 나타남, 새로 고쳐도 같음 → 대시보드 전체 고객 수 1 감소, 어느 필터에도 없음, 지연·오늘 마감 탭과 요약 자료 수에서 국민무역 자료가 빠짐.
4. **D2, D3, R2, R3**: 보관 목록에서 국민무역을 열어 자료 4건의 이름·상태·마감일이 그대로인지 본다. 보관 날짜·사유 표시가 없다. `SELECT COUNT(*) FROM customers`와 자료 건수가 보관 전과 같다.
5. **P1**: 보관한 국민무역 상세에서 고객 수정, 자료 추가·일괄 추가, AI 요청문, 자료 수정·삭제 버튼이 없고 상태 선택이 비활성이다. 묶음 영역을 열어 둔 상태라면 "적용"이 비활성이다. 보관 목록에는 "삭제" 버튼이 없다.
6. **P3 (API 직접 호출)**: 보관한 고객(예: 12)과 그 자료(예: 자료 8)에 대해 아래가 모두 409를 돌려주고, 뒤이어 DB 값이 그대로인지 확인한다.
   - `curl -X PATCH http://127.0.0.1:5173/api/customers/12 -H "Content-Type: application/json" -d "{\"name\":\"x\",\"note\":\"\"}"`
   - `curl -X DELETE http://127.0.0.1:5173/api/customers/12`
   - `curl -X POST http://127.0.0.1:5173/api/customers/12/requests -H "Content-Type: application/json" -d "{\"name\":\"x\",\"note\":\"\",\"due_date\":\"2026-12-31\"}"`
   - `curl -X POST http://127.0.0.1:5173/api/customers/12/request-templates/3/apply -H "Content-Type: application/json" -d "{\"due_date\":\"2026-12-31\"}"`
   - `curl -X POST http://127.0.0.1:5173/api/customers/12/ai-request-message`
   - `curl -X PATCH http://127.0.0.1:5173/api/material-requests/8 ...`, `.../8/status`, `curl -X DELETE http://127.0.0.1:5173/api/material-requests/8`
   - 대조군: 관리 중 고객(예: 23)에서는 같은 호출이 지금처럼 성공한다.
7. **S1, S2**: 관리 중 목록에서 "국민" 검색 → 결과 없음. 보관 목록에서 "국민" 검색 → 국민무역이 나옴.
8. **I1~I4**: 국민무역(보관) 한 줄 + 새 이름 한 줄 + ABC상사(관리 중) 한 줄이 든 CSV를 가져온다 → "파일 전체를 가져오지 않았습니다: 국민무역" 안내가 보이고, 고객 수·자료 수가 하나도 늘지 않는다. 국민무역 줄만 지운 파일은 지금처럼 들어간다. 관리 중 고객으로 "국민무역"을 하나 더 만든 뒤 같은 파일을 가져와도 막히는지 본다.
9. **R4, P2**: 보관 목록에서 국민무역 "되돌리기" → 관리 중 목록에 돌아옴, 자료 4건 그대로, 대시보드 숫자와 지연·오늘 마감 탭이 3번 전 값으로 돌아옴, 상세에서 버튼이 다시 살아남. 이후 "삭제"가 다시 보이고 동작한다(decision A1 "되돌린 뒤 삭제").
10. **G4**: 보관 목록을 띄워 둔 상태에서 대시보드 지연 탭의 "자료 보기"를 누르면 관리 중 목록과 그 고객 자료가 열린다.
11. **R1**: 상단 메뉴가 대시보드·고객 관리 두 개뿐이다.
12. **M1**: `docs/REQUIREMENTS.md`, `docs/goals.md`, `README.md`에서 "삭제"를 검색해 "관리하지 않는 고객은 삭제한다" 문장이 없고 보관·되돌리기 설명이 있는지 본다.
13. **운영**: 적용 순서 1번과 9번의 건수가 같다.

### 완료 기준 누락 점검
- 모든 기준(D1~D3, B1~B4, R1~R4, G1~G4, P1~P3, S1~S2, I1~I4, M1)이 "바꿀 파일" 표의 한 줄 이상과 이어진다.
- 표의 모든 변경은 기준 하나 이상과 이어진다. `POST /api/customers` 재조회의 `archived` 추가(`index.ts:278-282`)는 프런트 타입을 맞춰 새 고객이 관리 중 목록에서 보관·삭제 버튼을 갖게 하는 용도라 B1과 이었다.

## 올릴 질문
- **배포 담당과 자동 배포**: 저장소가 Cloudflare 빌드와 연결되어 main push 때 자동 배포되는지 확인이 필요하다. 그렇다면 운영 마이그레이션(`--remote`)을 누가 언제 push 전에 적용할지 정해야 한다.
- **확인 담당과 운영 건수 기록**: 자동 테스트가 없어 위 "확인 방법"을 사람이 직접 돌려야 한다. 누가 돌릴지, 운영 배포 전 고객·자료 건수와 백업을 누가 남길지 정해야 한다(PRD `:115`, `:118`).
- **"되돌리기"에 확인 창을 둘지**: PRD와 결정은 보관 확인만 말한다. 설계는 되돌리기를 확인 없이 바로 실행하는 것으로 잡았다(언제든 다시 보관할 수 있음). 확인 창이 필요하면 알려 달라.
- **보관 목록 모드에서 "고객 추가 / 파일로 추가" 버튼**: 설계는 화면 흐름을 바꾸지 않으려고 그대로 두었다. 보관 목록에서는 숨길지 정해 달라.
- **되돌릴 때 같은 이름의 관리 중 고객이 이미 있으면**: 설계는 막지 않는다(지금도 같은 이름 고객을 만들 수 있음). 막거나 경고해야 하면 정해 달라.
- **가져오기 거부 안내의 범위**: 설계는 이름만 알린다(decision A7 "어느 이름인지"). 줄 번호까지 보여 줘야 하면 프런트 미리보기(`App.tsx:141-186`)에서 줄 번호를 함께 보내도록 바꿔야 한다.
