# 02-2. React와 함께 사용하는 도구

<!-- 구분: 개발 -->
<!-- 유형: 설계(구성 소개) (+ 도구 비교) -->
<!-- 상태: 초안 (각 도구의 당시 선택 이유 미확인 — [확인 필요] 표시 부분은 사용자 확인 후 작성) -->
<!-- 코드 근거: moon-in-the-sky/pdps, 브랜치 claude/code-fix-planning-3q7615, 커밋 f1894bf8ced8c503074e47b276d2b7c877a3cdf3 — web/package.json, web/vite.config.js, web/src/App.jsx, web/src/lib/api.js, web/src/index.css, api/main.py -->
<!-- 비교의 시점: 도구는 비교 없이 도입됨(사용자 확인, 02-1 참고). 선택 이유는 구현 4일 뒤(2026-06-30)에 정리한 TECH_STACK.md §3.4와 코드 주석의 내용으로, 당시 판단이 아닌 사후 정리 -->
<!-- 사실 구분 규칙: "코드에서 확인"은 해당 커밋의 코드. "문서에 기록"은 TECH_STACK.md·docs/수정사항·코드 주석. 이 둘로 당시 선택 동기를 확정하지 않는다 -->

앞선 글에서는 React의 핵심 개념과 프론트엔드 도구를 선택하는 기준을 살펴보고, 이 플랫폼에 React를 적용한 이유를 정리했다. 하지만 React는 화면을 구성하는 UI 라이브러리이므로, 실제 화면을 만들고 배포하려면 다른 도구가 함께 필요하다. 이번 글에서는 이 플랫폼에서 React와 함께 사용하는 도구와, 각 도구가 맡는 역할을 정리한다.

여기서 선택 이유는 두 가지로 구분해 적는다. 하나는 코드와 설정에서 직접 확인한 사실이다. 다른 하나는 저장소의 기술 스택 문서(`TECH_STACK.md`)나 코드 주석에 기록된 사유다. 이 문서는 구현을 시작하고 4일 뒤(2026-06-30)에 정리한 것이라 개발 당시에 후보들을 비교한 기록으로 볼 수 없다. 02-1에서 밝혔듯 이 도구들은 다른 도구와 비교하지 않고 도입되었다.

## React만으로는 부족한 부분

React가 담당하는 것은 컴포넌트와 상태에 따른 화면 구성이다. 이 플랫폼의 웹 화면에는 그 밖에도 다음과 같은 일이 필요하다.

| 필요한 일 | React가 직접 제공하는가 |
| --- | --- |
| JSX를 브라우저가 읽는 JavaScript로 변환하고 배포 파일로 묶기 | 제공하지 않음 (빌드 도구 필요) |
| 주소에 따라 화면을 전환하고, 권한이 없으면 막기 | 제공하지 않음 (라우터 필요) |
| API 서버에 요청하고 응답·오류 처리하기 | 제공하지 않음 (브라우저 기능 또는 라이브러리 필요) |
| 스타일, 아이콘, 폰트, 차트 | 제공하지 않음 |
| 화면 전체가 공유하는 값(로그인 정보 등) 관리하기 | 기본 기능(Context)을 제공함 |

그래서 현재 프로젝트는 아래 도구를 조합해 구성한다.

## 현재 구성

`web/package.json`에서 확인한 구성은 다음과 같다. 의존성은 실행에 필요한 것 6개, 개발·빌드에 필요한 것 10개이고, `package-lock.json`에 기록된 설치 패키지는 간접 의존성을 포함해 327개다.

| 역할 | 도구 | 이 플랫폼에서 하는 일 |
| --- | --- | --- |
| 화면 구성 | React 19 | 컴포넌트와 상태 |
| 빌드·개발 서버 | Vite, `@vitejs/plugin-react` | JSX 변환, 배포용 파일 생성, 개발 서버 |
| 구형 브라우저 대응 | `@vitejs/plugin-legacy`, `autoprefixer`, `terser` | 대상 브라우저용 코드 변환과 압축 |
| 라우팅 | `react-router-dom` | 주소에 따른 화면 전환, 권한 확인 |
| 스타일 | Tailwind CSS 3, PostCSS | 화면 스타일 |
| 폰트·아이콘 | `@fontsource/noto-sans-kr`, `lucide-react` | 로컬 폰트, 아이콘 |
| 차트 | `recharts` | 내보내기 화면의 그래프 |
| 코드 점검 | `oxlint` | 린트 |
| 서버 요청 | 브라우저의 `fetch`를 감싼 자체 모듈 (`lib/api.js`) | 인증 토큰 첨부, 오류 처리 |
| 전역 상태 | React Context | 로그인 정보, 업로드 대기열 |

Redux 같은 상태 관리 라이브러리, axios 같은 HTTP 라이브러리, 폼 전용 라이브러리는 사용하지 않는다. 이후 각 항목을 살펴본다.

## 빌드 도구: Vite

Vite는 개발 중에는 개발 서버를 제공하고, 배포할 때는 JSX와 여러 소스 파일을 브라우저가 실행할 수 있는 파일로 변환해 묶는 빌드 도구다. 이 플랫폼의 설정은 다음과 같다.

```js
// web/vite.config.js (발췌)
export default defineConfig({
  plugins: [
    react(),
    legacy({ targets: ['Firefox >= 52', 'Edge >= 16', 'Chrome >= 60'] }),
  ],
  server: {
    proxy: { '/api': 'http://localhost:8000' },
  },
})
```

- `react()`는 JSX를 처리하는 플러그인이다.
- `legacy()`는 대상 브라우저를 지정해 구형 브라우저용 코드를 함께 만든다. 다음 절에서 자세히 설명한다.
- 개발 서버의 `proxy`는 `/api` 요청을 FastAPI 서버(`localhost:8000`)로 전달한다. 개발할 때 화면과 API를 서로 다른 서버로 실행해도 화면에서는 같은 주소처럼 요청할 수 있다.

저장소의 기술 스택 문서에는 빌드 도구로 Vite를 선택한 이유가 "빠른 개발 서버(HMR)와, legacy 플러그인으로 구형 브라우저용 빌드까지 한 번에 처리할 수 있어서"라고 기록되어 있다. 같은 문서에는 대안으로 CRA와 Webpack 직접 구성이 적혀 있다. 이 문서는 구현을 시작하고 4일 뒤(2026-06-30)에 정리한 것이어서 당시의 비교 기록은 아니다.

Vite는 첫 구현(2026-06-26)부터 사용했고, 첫 커밋에 Vite의 React 기본 템플릿 파일이 남아 있다. 개발 서버의 `/api` 프록시도 첫 설정에 들어 있었다. 이는 화면과 Python API 서버를 분리해 개발하는 구조를 처음부터 전제했다는 뜻이다.

## 지원 브라우저와 그에 따른 구성

이 플랫폼은 폐쇄망 환경에서 사용하므로 사용자의 브라우저가 최신이라고 가정할 수 없다. 저장소의 수정 사항 문서에는 2026-09-07에 확정한 지원 정책이 다음과 같이 기록되어 있다.

- 필수 지원: Firefox 52 이상, Edge 16 이상, Chrome 60 이상
- 지원 대상 제외: Safari

구현 순서로 보면 이 정책이 먼저 있었던 것은 아니다. 첫 구현(2026-06-26)은 Tailwind CSS 4로 시작했고, 3일 뒤인 2026-06-29에 폐쇄망 배포(FastAPI가 빌드 결과물을 제공)와 구형 브라우저 지원을 반영하면서 Tailwind CSS 3과 PostCSS 구성으로 바꿨다. 위 정책의 확정(2026-09-07)은 그 뒤다. 요구가 나중에 드러났기 때문에, 구성도 구현 후에 맞춰 바뀐 것이다.

이 정책은 빌드 구성에 직접 영향을 준다.

- `@vitejs/plugin-legacy`와 `terser`로 최신 브라우저용과 구형 브라우저용 코드를 함께 만든다(기술 스택 문서의 설명이며, 이번에 빌드 결과물을 열어 확인하지는 않았다).
- Tailwind CSS는 최신 버전이 아닌 3 버전을 PostCSS와 `autoprefixer`로 처리한다. 수정 사항 문서에는 이 구성을 유지해야 하는 이유가 기존 구형 PC 요구사항이라고 기록되어 있다.
- 구형 브라우저가 지원하지 않는 CSS는 수동으로 보완한 부분이 있다. 예를 들어 `index.css`에는 flex 컨테이너의 `gap`을 지원하지 않는 브라우저를 위해 여백으로 대체하는 규칙이 있다.

대상 브라우저를 정하면 쓸 수 있는 도구와 문법의 범위도 함께 정해진다. 최신 브라우저만 대상으로 하는 프로젝트와 달리 변환 단계와 확인할 항목이 늘어난다.

## 라우팅: react-router-dom

라우팅은 주소(URL)에 따라 어떤 화면을 보여줄지 정하는 기능이다. 전통적인 웹은 주소마다 서버가 HTML을 새로 보내주었다. React로 만드는 SPA(Single Page Application)는 HTML을 한 번 받은 뒤, 주소가 바뀌면 JavaScript가 화면 컴포넌트를 교체한다. 페이지 전체를 다시 불러오지 않는다.

React에는 라우팅 기능이 없어서 이 플랫폼은 `react-router-dom`을 사용한다.

```jsx
// web/src/App.jsx (발췌)
<Routes>
  <Route path="/download" element={
    <RequireAuth><Layout><DataDownload /></Layout></RequireAuth>
  } />
  <Route path="/params" element={
    <RequireAdmin><Layout><ParamExplorer /></Layout></RequireAdmin>
  } />
  <Route path="*" element={<NotFound />} />
</Routes>
```

- 주소 `/download`는 데이터 다운로드 화면, `/params`는 파라미터 관리 화면으로 연결한다.
- `RequireAuth`와 `RequireAdmin`은 로그인 여부와 관리자 권한을 확인하는 컴포넌트다. 권한이 없으면 로그인이나 공지사항 화면으로 보낸다.
- 차트 라이브러리를 쓰는 내보내기 화면은 `lazy`로 분리해, 해당 화면에 들어갈 때만 코드를 내려받도록 했다.

화면 안의 단계 전환(프로젝트 선택 → 모듈 진입)은 주소가 아니라 상태(`openProjectId`, `openModuleId`)로 처리한다. 화면 단위는 라우팅으로, 화면 안의 변화는 상태로 나눈 것이다.

SPA에서는 서버 쪽도 확인해야 한다. 사용자가 `/params`에서 새로고침하면 브라우저는 서버에 `/params`를 요청하는데, 이 경로에 해당하는 파일은 없다. 그래서 FastAPI는 요청한 정적 파일이 없는 경로에 `index.html`을 돌려주도록 구성했다(`/api/` 경로는 제외하고 404를 반환한다). 그러면 React가 주소를 읽어 맞는 화면을 그린다.

react-router-dom은 첫 구현(2026-06-26)부터 사용했다. 선택 이유와 대안을 비교한 기록은 없다.

## 폼

폼은 사용자에게 값을 입력받아 서버로 보내는 화면 요소다. HTML 입력창은 자기 값을 스스로 들고 있지만, React에서는 입력값을 상태로 관리하는 방식을 많이 쓴다.

```jsx
// web/src/pages/paramExplorer/ParamFormPanel.jsx (발췌)
const [form, setForm] = useState(() => initial)

<input
  value={form[c.k] ?? ''}
  onChange={e => setForm(f => ({ ...f, [c.k]: e.target.value }))}
/>

<button onClick={save} disabled={saving}>저장</button>
```

- 입력창의 값은 `form` 상태에서 읽고, 입력할 때마다 상태를 갱신한다.
- 저장 중(`saving`)이면 버튼이 비활성화된다. 상태가 화면을 결정하는 선언형의 예다.
- 입력 항목은 프로토콜별 정의(`PROTO[protocol].fields`)를 `map`으로 돌려 만든다. 프로토콜이 달라져도 항목 정의만 바꾸면 폼이 바뀐다.

현재는 폼 전용 라이브러리 없이 React의 상태만으로 구현했다. 입력 항목과 검증 규칙이 더 복잡해지면 폼 라이브러리를 검토할 수 있으며, 이 부분은 현재 적용하지 않은 확장 검토다.

## 서버 요청: fetch를 감싼 자체 모듈

화면은 FastAPI 서버에 조회·저장·처리를 요청한다. 이 플랫폼은 axios 같은 라이브러리를 쓰지 않고, 브라우저의 `fetch`를 감싼 `lib/api.js`를 사용한다. 이 모듈에서 코드로 확인한 동작은 다음과 같다.

- 로그인 토큰이 있으면 모든 요청에 `Authorization` 헤더로 붙인다.
- 서버가 오류를 돌려주면 응답의 `detail`을 사람이 읽을 수 있는 메시지로 바꿔 화면에 전달한다.
- 로그인 요청을 제외한 요청이 401(인증 만료)이면 저장된 토큰과 사용자 정보를 지우고 `/login`으로 이동한다.
- 파일 업로드는 진행률을 표시하기 위해 `fetch` 대신 `XMLHttpRequest`를 사용한다.
- 파일 다운로드는 `fetch`에 취소 신호(`AbortSignal`)를 전달할 수 있다. 이 부분은 02-3에서 다시 다룬다.

저장소의 기술 스택 문서에는 axios를 쓰지 않은 이유가 "외부 의존성을 추가하지 않고, 토큰 첨부와 401 처리를 20줄 내외로 구현할 수 있어서"라고 기록되어 있다. 같은 문서에는 401일 때 `navigate()` 대신 `window.location.href`를 쓰는 이유가 "이 함수가 React 컴포넌트 바깥에서 호출되어 라우터에 접근할 수 없기 때문"이고, 업로드에 `XMLHttpRequest`를 쓰는 이유가 "`fetch`의 업로드 진행률 방식이 대상(구형) 브라우저와 호환성이 떨어지기 때문"이라고 설명되어 있다. 이 설명들은 코드 동작과 일치하지만 구현 후에 정리한 설명이며, 각 결정을 내릴 당시의 판단으로 볼 근거는 없다.

## 전역 상태: React Context

로그인 정보처럼 화면 전체가 공유하는 값은 React의 Context로 관리한다. 현재 코드에는 `AuthContext`(로그인 사용자)와 `UploadQueueContext`(업로드 대기열) 두 개가 있다. 기술 스택 문서의 설명은 "전역 상태가 로그인 사용자 하나뿐이라 별도 라이브러리가 필요 없다"인데, 현재 코드는 Context가 두 개이므로 이 문서 설명은 갱신이 필요하다. 이 판단은 현재 코드를 기준으로 다시 검토할 항목이다.

## 차트, 스타일, 폰트

- **차트(recharts)**: 내보내기 화면에서 시계열 그래프를 그린다. `App.jsx`에서 이 화면을 `lazy`로 불러와, 차트 코드를 초기 로딩에서 분리한다. 기술 스택 문서는 recharts를 고른 이유로 "React 컴포넌트 친화적인 API와 선언적 구성"을 기록하고 있다.
- **스타일(Tailwind CSS)**: 미리 정의된 클래스 이름을 화면 요소에 붙여 스타일을 지정한다. 앞서 설명한 지원 브라우저 정책 때문에 3 버전을 PostCSS와 함께 사용한다.
- **폰트(`@fontsource/noto-sans-kr`)**: 폰트를 패키지로 포함해 `index.css`에서 불러온다. 코드의 주석에는 "폐쇄망 배포라 CDN 대신 로컬 번들을 사용"한다고 적혀 있고, 실제로 쓰는 굵기와 한국어·라틴 서브셋만 포함해 용량을 줄였다.
- **아이콘(`lucide-react`)**: 화면의 아이콘을 컴포넌트로 사용한다.
- **코드 점검(`oxlint`)**: `npm run lint`로 실행한다.

## 배포: FastAPI가 빌드 결과물을 제공

개발이 끝나면 Vite가 화면을 `web/dist`로 빌드하고, 이 결과물은 별도의 웹 서버 없이 FastAPI가 직접 제공한다. 정적 파일 제공 조건, 캐시 설정, 폐쇄망 배포 방향은 02-3에서 정리한다. 저장소의 기술 스택 문서는 이 구성의 이유를 "폐쇄망에서 운영 구성 요소를 최소화하기 위해서"라고 기록하고 있다. 빌드된 결과물을 정적 파일로 제공하는 방식은 React만 가능한 것은 아니다.

## 도구를 조합할 때의 한계

- 도구마다 버전과 설정을 관리해야 한다. 지금은 실행 의존성 6개, 개발 의존성 10개이지만 간접 의존성까지 포함하면 327개 패키지가 설치된다. 이 숫자는 `package-lock.json` 기준이며, 설치 용량이나 빌드 시간은 측정하지 않았다.
- 자체 `fetch` 모듈은 외부 라이브러리를 줄이는 대신 요청·오류·인증 처리를 직접 유지해야 한다. 요청 중복 제거나 캐시가 필요해지면 같은 문제를 해결하는 라이브러리를 검토할 수 있으나, 현재 도입하지 않았고 비교 실험도 하지 않았다.
- 구형 브라우저 정책을 바꾸면(예: 지원 하한 상향, Safari 추가) 빌드 구성과 스타일 처리 방식도 함께 다시 확인해야 한다.

## 참고 자료와 연결 글

- 이전 글: [02-1. 프론트엔드를 React로 선택한 이유](02-1-react.md)
- 다음 글: [02-3. React를 선택하면서 관리해야 하는 부분](02-3-react-management.md)
- 코드(커밋 고정): [package.json](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/package.json), [vite.config.js](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/vite.config.js), [App.jsx](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/src/App.jsx#L51-L84), [api.js](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/src/lib/api.js), [index.css](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/src/index.css#L1-L14), [main.py](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/api/main.py#L79-L122)
- 저장소 문서: [TECH_STACK.md §3](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/TECH_STACK.md)
