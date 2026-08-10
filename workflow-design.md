# 워크플로우 설계서

> **산출물 1** — 워크플로우 구조, 각 단계별 역할과 연결 구조
> 이 문서는 **실제 구축된 시나리오 기준**으로 작성되었습니다. 모듈 번호는 캔버스 표시와 동일합니다.

## 1. 전체 구조

![Make 시나리오 전체 구조](01-scenario-overview.png)


## 2. 데이터 흐름 한 줄 요약

**RSS n건** → (주제 필터) **AI 기사** → (중복 필터) **신규 기사** → (건별 요약) **요약** → **Notion 행**

핵심은 **비싼 단계를 뒤로 미는 순서**입니다. 무료인 필터링(A)과 DB 조회(5·6)를 앞에 두고, 유료인 AI 호출(8)을 뒤에 배치했기 때문에 중복 기사나 비주제 기사에는 요약 비용이 한 푼도 들지 않습니다.

---

## 3. 모듈별 상세

### 트리거 — 스케줄

Make에서 스케줄은 별도 모듈이 아니라 **시나리오 속성**입니다. 캔버스 좌측 하단 시계 아이콘 → `Every day` / `08:00` / 타임존 `Asia/Seoul`.

| 항목 | 값 | 이유 |
|---|---|---|
| 실행 주기 | 매일 1회 | 요구사항 |
| 실행 시각 | 08:00 KST | 전날 오후~밤 기사가 모두 색인된 뒤이면서, 팀이 출근해 결과를 확인할 수 있는 시각 |
| 타임존 | Asia/Seoul | 계정 프로필 기본값이 UTC라 그대로 두면 실제로는 17:00에 실행됨 |

---

### 모듈 2 — RSS: Watch RSS feed items

**역할** — 피드를 읽어 기사 1건당 bundle 1개를 만들어 흘려보냅니다. 이후 모든 모듈은 **기사 1건 단위로 반복 실행**됩니다.

| 설정 | 값 |
|---|---|
| URL | `https://rss.etnews.com/03.xml` |
| Maximum number of returned items | `3` (실전) / `1` (테스트 중) |

**왜 `Watch`(폴링)이고 `Retrieve`(전체 조회)가 아닌가**
`Watch` 계열 모듈은 마지막으로 처리한 항목을 Make가 기억해, 다음 실행 때 **그 이후 항목만** 내보냅니다. 즉 트리거 자체가 1차 중복 방지 역할을 합니다. `Retrieve`를 쓰면 매일 같은 기사를 전부 받아 필터가 헛돌게 됩니다.

**주요 출력 필드**

| 필드 | 매핑 표기 | 용도 |
|---|---|---|
| Title | `{{2.title}}` | Notion 제목, 요약 입력 |
| URL | `{{2.url}}` | Notion 원본 링크, **중복 키의 원재료** |
| Description | `{{2.description}}` | 요약 입력 (300~420자 확보됨) |
| Date created | `{{2.dateCreated}}` | Notion 발행일시, 신선도 필터 |

**실행 결과**

![RSS 모듈 출력](rss.png)

기사 1건이 bundle 1개로 만들어진 모습입니다. 이후 모든 모듈이 이 bundle 하나를 넘겨받아 순차 처리합니다.

- `Description`과 `Summary`가 **같은 내용**입니다. 전자신문 피드는 두 필드에 동일한 발췌문을 넣기 때문에, 요약 입력으로는 `Description` 하나만 쓰면 충분합니다.
- 본문 발췌가 약 300자로 끝나 있습니다(`…국내 중소 게임·콘텐츠`). **원문 전체가 아니라 도입부만 제공**된다는 뜻인데, 3줄 요약에는 도입부만으로 충분하므로 원문 크롤링 단계를 두지 않은 근거가 됩니다.
- `RSS fields` 하위에 `guid`가 별도로 존재합니다. 중복 키를 URL 해시 대신 `guid`로 잡는 선택지도 있었으나, `guid`는 매체마다 형식이 제각각이라 피드를 교체하면 규칙이 깨집니다. URL 해시는 어느 피드에서든 동일하게 동작합니다.

> **`Choose where to start`** — 모듈 우클릭으로 시작 지점을 바꿀 수 있습니다. `All`을 고르면 **가장 오래된 항목부터** 시간순으로 처리하므로(항목 누락 방지 설계), 최신 기사로 테스트하려면 `Choose manually`로 직접 골라야 합니다. 실전 전환 시에는 `From now on`으로 설정합니다.

---

### 필터 A — AI 주제 & 신선도

**역할** — 관심 없는 기사를 여기서 끊습니다. Make의 필터는 모듈이 아니라 **연결선 위의 조건**이며, 통과하지 못한 bundle은 그 자리에서 조용히 사라집니다(에러 아님).

| # | 대상 | 연산자 | 값 |
|---|---|---|---|
| A-1 | `{{lower(2.title)}} {{lower(2.description)}}` | Text: `Matches pattern` | 소문자 키워드 정규식 (README §2-1) |
| A-2 | `{{2.dateCreated}}` | Datetime: `Later than` | `{{addHours(now; -36)}}` |

**설계 포인트 3가지**

1. **제목과 본문을 공백 하나로 이어 붙여 한 번에 검사.** 조건을 둘로 나누면 OR 그룹이 필요해져 복잡해지는데, 이어 붙이면 "제목에 없어도 본문에 있으면 통과"가 자연히 만족됩니다.
2. **`lower()`로 감싸는 것이 필수.** Make의 정규식은 JavaScript 엔진이라 `(?i)` 인라인 플래그를 지원하지 않습니다. 대소문자 무시는 검사 대상과 패턴 **양쪽을 소문자로 맞춰서** 구현합니다. 이걸 빼면 기사에 `AI`가 대문자로 쓰인 경우 소문자 패턴 `ai`와 매칭되지 않아, **AI 키워드 그룹 전체가 무력화**됩니다.
3. **`length()`와 `lower()`를 혼동하지 말 것.** 구축 중 실제로 발생한 실수입니다. `length(title)`을 넣으면 제목이 글자 수 숫자로 바뀌어 검사에서 통째로 빠지는데, 에러가 나지 않아 발견이 늦습니다.

**A-2 조건 운영 메모** — 구축 단계에서는 이 조건이 오래된 테스트 기사를 전부 막아 진행을 방해하므로, 파이프라인 전체를 검증하는 동안 일시 제거했다가 마지막에 복원했습니다. 조건을 하나씩 떼어 원인을 좁히는 것이 Make 필터 디버깅의 기본입니다. 필터 인스펙터에서 조건별 ✅/🚫/◎(미평가) 아이콘으로 어느 조건이 막고 있는지 확인할 수 있습니다.

---

### 모듈 4 — Tools: Set multiple variables

**역할** — 이후 모듈이 반복해서 쓸 파생 값을 한 곳에서 계산합니다. 같은 수식을 여러 모듈에 흩어 놓으면 나중에 한 군데만 고치는 실수가 납니다.

| 변수 | 수식 | 용도 |
|---|---|---|
| `link_hash` | `{{sha256(2.url)}}` | 중복 방지 키 |
| `feed_name` | `전자신문 IT` | 출처 표기 (Notion `출처` 속성 사용 시) |

**실제 설정**

![Set multiple variables 설정](tools.png)

**왜 URL 자체가 아니라 해시인가**
① URL은 길이 제한이 없고 쿼리 파라미터(`?utm_source=...`)가 붙으면 같은 기사인데 다른 문자열이 됩니다. ② 해시는 항상 64자 고정이라 Notion 텍스트 속성에서 비교가 안정적입니다. ③ 과제 요구사항이 명시한 "원문 링크 해시" 방식에 그대로 해당합니다.

> **`pub_iso`(Variable 2)는 현재 사용하지 않습니다.** 스크린샷에 정의가 남아 있지만 어느 모듈도 이 값을 참조하지 않습니다.
>
> 처음에는 `formatDate()`로 ISO 문자열을 만들어 Notion에 넘기려 했으나, Make의 Notion 모듈이 **날짜 타입 값을 직접 받아 스스로 타임존을 적용**하는 구조라 미리 가공한 문자열은 `Invalid date in parameter 'start'` 오류를 냅니다. 원본 필드 `{{2.dateCreated}}`를 그대로 넘기는 것이 정답입니다(모듈 13 참고).
>
> 동작에는 지장이 없으나, 쓰지 않는 변수를 남겨두면 나중에 읽는 사람이 "이건 어디에 쓰이지?"를 추적하게 됩니다. 제출 전 삭제를 권합니다.

---

### 모듈 5 — Notion: Search Objects

**역할** — 이 해시가 DB에 이미 있는지 조회합니다.

| 설정 | 값 |
|---|---|
| Search Objects | `Data Source Items` |
| Data Source ID | `AI 뉴스 아카이브 DB` |
| Filter | `링크해시` / `Text: Equals` / `{{4.link_hash}}` |
| Limit | `1` |

찾으면 bundle 1개, 없으면 **bundle 0개**를 내보냅니다. 그런데 Make에서 bundle 0개는 "이후 모듈이 아예 실행되지 않음"을 뜻하므로, 이대로는 "없을 때 저장" 로직을 만들 수 없습니다. 그래서 모듈 6이 필요합니다.

> **Legacy가 아닌 `Data Source` 계열을 선택했습니다.** Notion API가 "하나의 데이터베이스가 여러 데이터 소스를 가질 수 있는" 구조로 개편되면서 Make가 모듈을 새로 만들었고, `(Legacy)` 모듈은 향후 중단 예정입니다. 조회(5)와 저장(13)을 같은 계열로 통일해야 필드 스키마가 일치합니다.

---

### 모듈 6 — Flow Control: Array aggregator

**역할** — **0개든 1개든 반드시 bundle 1개로 만들어 주는 변환기.** 이 워크플로우에서 가장 헷갈리는 지점입니다.

| 설정 | 값 |
|---|---|
| Source Module | `5. Notion – Search Objects` |
| Aggregated fields | `Page ID` (아무 필드나 하나) |

집계 후 `{{6.array}}`의 길이를 보면 조회 결과 개수를 알 수 있습니다. 0이면 신규, 1 이상이면 중복입니다.

> **`Tools`가 아니라 `Flow Control` 그룹에 있습니다.** Tools 안의 `Numeric/Table/Text aggregator`는 값을 합계·표·문자열로 묶는 용도라 목적이 다릅니다.
>
> **필터 B는 이 모듈 *뒤* 연결선에 걸어야 합니다.** 앞에 걸면 `references inaccessible module` 오류가 납니다 — 아직 실행되지 않은 모듈의 출력을 참조할 수 없기 때문입니다.

---

### 필터 B — 중복 아님

| 대상 | 연산자 | 값 |
|---|---|---|
| `{{length(6.array)}}` | Numeric: `Equal to` | `0` |

여기를 통과한 기사만 유료 AI 호출로 넘어갑니다. **이 필터의 위치가 곧 비용 정책입니다.**

---

### 모듈 7 — JSON: Create JSON

**역할** — 프롬프트 텍스트를 JSON 안전 형태로 감쌉니다.

| 설정 | 값 |
|---|---|
| Data structure | `gemini_part` — 필드 **하나**: `text` (Text, Required) |
| `text` | 프롬프트 전문 ([../assets/prompt.md](../assets/prompt.md)) |

**왜 이 모듈이 필요한가**
기사 제목·본문에는 큰따옴표(`"`), 줄바꿈, 역슬래시가 섞여 있습니다. 이것을 HTTP 모듈의 raw JSON 본문에 그대로 끼워 넣으면 JSON 구조가 깨져 `400 Bad Request`가 납니다. 어제까지 잘 되다가 특정 기사에서만 갑자기 실패하는, 원인 찾기 가장 어려운 종류의 버그입니다. `Create JSON`은 매핑된 값을 자동으로 이스케이프하므로 이 문제가 구조적으로 발생하지 않습니다.

**왜 요청 본문 전체가 아니라 `text` 필드 하나만 만드는가**
사용자 입력이 섞이는 부분은 프롬프트 하나뿐이고, 나머지(`generationConfig`, `responseSchema`)는 값이 변하지 않는 고정 설정이라 이스케이프 문제가 없습니다. 게다가 이 모듈의 출력 `{{7.json}}`은 정확히 `{"text": "..."}` 형태인데, 이것이 **Gemini API의 `parts` 배열 원소 형태와 그대로 일치**합니다. 그래서 모듈 8에서는 `"parts": [ {{7.json}} ]`이라고 쓰기만 하면 조립이 끝납니다.

**실행 결과**

![Create JSON 모듈 출력](json.png)

이 모듈이 왜 필요한지가 출력에 그대로 드러납니다.

- 출력 전체가 `{"text":"…"}` **한 겹으로 감싸인 형태**입니다. 이것을 모듈 8의 `"parts": [ … ]` 안에 그대로 넣으면 요청 본문이 완성됩니다.
- 프롬프트 안의 큰따옴표가 전부 `\"`로, 줄바꿈이 `\r\n`으로 **자동 이스케이프**되어 있습니다. 예를 들어 작성 규칙 1번의 `"~다."`가 `\"~다.\"`로 바뀌었습니다. 이 처리를 사람이 직접 하려 했다면 기사 본문에 따옴표가 섞인 날 바로 `400 Bad Request`가 났을 것입니다.
- 프롬프트 하단 `[기사]` 블록에 제목·발행일·본문이 실제 값으로 채워져 있습니다. 매핑이 정상 동작했다는 증거입니다.
- 발행일이 `2026-08-09T06:03:12.000Z`(UTC)로 들어갔습니다. 요약 품질에는 영향이 없지만, Notion에 저장되는 발행일시는 모듈 13에서 `Asia/Seoul`로 변환되므로 **두 값이 9시간 차이 나는 것이 정상**입니다.

---

### 모듈 8 — HTTP: Make a request (Gemini)

**역할** — 요약·감성·키워드를 **한 번의 호출로** 받아옵니다.

| 설정 | 값 |
|---|---|
| Authentication type | `None` (인증은 헤더로 처리) |
| URL | `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent` |
| Method | `POST` |
| Header 1 | `x-goog-api-key` : `(API 키)` |
| Header 2 | `Content-Type` : `application/json` |
| Body content type | `application/json` |
| Body input method | `JSON string` |
| Body content | [../assets/gemini-request-body.txt](../assets/gemini-request-body.txt) |
| Parse response | ✅ `Yes` |

**핵심 설계 ① — 구조화 출력(Structured Output)**

요청 본문의 `generationConfig.responseSchema`에 응답 형태를 못 박아 두었습니다. 그래서 Gemini는 자유 문장이 아니라 반드시 아래 형태로만 답합니다.

```json
{ "summary": "...", "sentiment": "긍정", "keywords": ["...", "...", "..."] }
```

이 한 가지 설계가 세 가지를 동시에 해결합니다.

1. **파싱 불필요** — "요약: ~" 같은 텍스트에서 정규식으로 뽑아낼 필요가 없습니다.
2. **호출 1회 원칙 충족** — 요약·감성분석·키워드를 각각 호출하면 3회지만, 한 스키마에 담으면 1회입니다. (보너스 2를 추가 비용 0으로 달성)
3. **감성 값 고정** — `enum: ["긍정","중립","부정"]`으로 제한해, Notion Select 속성에 없는 값이 들어와 저장이 실패하는 사고를 막습니다.

**핵심 설계 ② — thinking 비활성화 (구축 중 실제 장애로 발견)**

```json
"maxOutputTokens": 2048,
"thinkingConfig": { "thinkingBudget": 0 },
```

`gemini-2.5-flash`는 **thinking 모델**이라 내부 추론에 출력 토큰을 먼저 소비합니다. 처음 `maxOutputTokens: 1024`로 설정했을 때, 추론이 예산 대부분을 쓰고 실제 JSON이 문장 중간에서 잘려 다음 모듈이 `Source is not valid JSON` 오류를 냈습니다.

| 조치 | 효과 |
|---|---|
| `thinkingBudget: 0` | 추론 과정을 끔. 기사 요약은 복잡한 추론이 불필요한 작업이라 품질 손해가 거의 없고, 속도·비용이 줄며 출력 토큰이 온전히 JSON에만 쓰임 |
| `maxOutputTokens` 1024 → 2048 | 한국어는 영어보다 토큰 소비가 큼. 여유를 두되 상한은 유지해 비용 폭주 방지 |

**진단법** — HTTP 모듈 출력의 `candidates[1].finishReason`이 `MAX_TOKENS`이면 잘림이 확정입니다.

`temperature`는 `0.2`로 낮췄습니다. 요약은 창의성이 아니라 **재현성**이 중요한 작업이고, 값이 높으면 같은 기사에 대해 실행할 때마다 요약 길이와 톤이 흔들립니다.

---

### 모듈 9 — JSON: Parse JSON

**역할** — Gemini 응답 안에 **문자열로 들어 있는** 결과 JSON을 실제 객체로 바꿉니다.

| 설정 | 값 |
|---|---|
| JSON string | `{{8.data.candidates[1].content.parts[1].text}}` |
| Data structure | `gemini_result` (summary / sentiment / keywords) |
| Strict | `No` |

> ⚠️ **Make의 배열 인덱스는 1부터 시작합니다.** 다른 언어 습관대로 `candidates[0]`이라고 쓰면 값이 비어 있는데 에러도 안 나서 원인을 못 찾습니다.

`Strict`를 `No`로 둔 이유: 응답 형식은 이미 `responseSchema`가 API 레벨에서 강제하고 있습니다. 여기서 한 번 더 조이면 검증만 이중이 되고 실패 지점만 늘어납니다.

응답 구조가 이중으로 감싸여 있는 이유: 바깥은 Gemini API의 공통 응답 봉투(candidates/content/parts), 안쪽 `text`가 우리가 스키마로 요청한 결과입니다.

**실행 결과 — 이 워크플로우에서 가장 중요한 화면**

![Parse JSON 모듈 입출력](../image/json2.png)

**INPUT**(문자열)과 **OUTPUT**(구조화된 필드)을 나란히 보면 AI 가공 단계가 무엇을 했는지 한눈에 들어옵니다.

| 검증 항목 | 결과 |
|---|---|
| 3줄 이내 요약 | ✅ 정확히 3문장. 각 문장이 60자 이내 |
| 프롬프트 구조 준수 | ✅ ①무엇이(개발사가 참가한다) → ②누가/얼마나(진흥원이 공동관 운영) → ③그래서(시장 개척에 나선다) |
| 사실 왜곡 없음 | ✅ 모듈 7 입력의 원문과 대조 시 새로 지어낸 정보 없음 |
| 감성 태그 | ✅ `긍정` — enum 3값 중 하나 |
| 키워드 | ✅ 배열 3개. 일반어가 아닌 고유명사 위주(게임스컴 2026, 한국콘텐츠진흥원, 코리아 게임 로드쇼) |

INPUT 쪽이 `{"summary":"…","sentiment":"긍정","keywords":[…]}` 형태의 **깔끔한 JSON 한 줄**이라는 점에 주목하세요. 코드펜스(```` ```json ````)도, "다음은 요약입니다" 같은 서두도 없습니다. 이것이 `responseMimeType`과 `responseSchema`로 형식을 API 레벨에서 강제한 결과이며, 덕분에 이 모듈이 정규식이나 문자열 자르기 없이 그대로 파싱할 수 있습니다.

---

### 모듈 13 — Notion: Create a Data Source Item

**역할** — 최종 저장.

| Notion 속성 | 타입 | 매핑 값 |
|---|---|---|
| 제목 | Title | `{{2.title}}` |
| 요약 | Text | `{{9.summary}}` |
| 원본 링크 | URL | `{{2.url}}` |
| 발행일시 | Date | `{{2.dateCreated}}` + `Include Time: Yes` |
| 링크해시 | Text | `{{4.link_hash}}` |
| 감성 | Select | `{{9.sentiment}}` (Map 토글 ON) |
| 키워드 *(선택)* | Multi-select | `{{9.keywords}}` (Map 토글 ON) |
| 출처 *(선택)* | Select | `{{4.feed_name}}` (Map 토글 ON) |

**날짜 매핑 주의** — `Start Time`에 `formatDate()`로 만든 ISO 문자열을 넣으면 `Invalid date in parameter 'start'` 오류가 납니다. 필드 아래 `Time zone: Asia/Seoul` 표기가 말해주듯 이 모듈은 **날짜 타입 값을 받아 스스로 변환**하므로, 원본 `{{2.dateCreated}}`를 그대로 넘겨야 합니다.

**`Include Time`을 `Yes`로** — 기본값 `No`로 두면 시각이 잘려 "발행 일시"가 아니라 "발행 일자"가 됩니다.

**Select/Multi-select 매핑** — 속성 옆 `Map` 토글을 켜야 동적 값을 넣을 수 있습니다. 꺼져 있으면 미리 정의된 옵션 중 고정값만 선택 가능합니다.

**실행 결과**

![Notion 저장 모듈 출력](../image/notion.png)

- `Object: page` / `Database Item ID`가 반환됐습니다. **Notion에 페이지가 실제로 생성됐다는 API 차원의 증거**입니다.
- `In trash: false`, `is_archived: false` — 정상 상태로 저장됐습니다.
- `URL`에 생성된 페이지 주소가 담겨 있습니다. 향후 Slack 알림에 "저장 완료" 링크를 붙이고 싶다면 이 값을 쓰면 됩니다.
- `Input method: list`, `Data Source ID`가 입력에 찍혀 있어 **어느 DB에 썼는지**가 기록으로 남습니다.

이 모듈이 성공 응답을 반환하는 것으로 파이프라인 전 구간이 사람 개입 없이 완료됩니다.

---

