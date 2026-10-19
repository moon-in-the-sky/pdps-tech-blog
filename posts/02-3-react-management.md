# 02-3. React를 선택하면서 관리해야 하는 부분

<!-- 구분: 개발 -->
<!-- 유형: 설계(트레이드오프와 한계) (+ 트러블슈팅 사례) -->
<!-- 상태: 초안 (배포 부분은 설계안 기준 — 구현 여부 확인 필요) -->
<!-- 코드 근거: moon-in-the-sky/pdps, 브랜치 claude/code-fix-planning-3q7615, 커밋 f1894bf8ced8c503074e47b276d2b7c877a3cdf3 — web/src/pages/paramExplorer/ParamExplorer.jsx, web/src/pages/paramExplorer/ParamFormPanel.jsx, web/src/lib/requestSequence.js, web/src/pages/DataDownload.jsx, web/src/pages/Export.jsx, web/src/lib/api.js, web/package.json, api/main.py, deploy/windows/README.md -->
<!-- 비교의 시점: 해당 없음(현재 구현의 관리 항목 정리). 문서 설계안과 구현 상태는 구분 -->

앞선 글에서는 React의 개념과 선택 기준, 함께 사용하는 도구를 살펴봤다. 이번 글에서는 React를 사용하면서 개발자가 직접 설계하고 관리해야 하는 부분을 정리한다. React는 상태에 따라 화면을 그려주지만, 상태를 어떻게 나눌지, 비동기 응답을 어떻게 다룰지, 의존성과 배포를 어떻게 준비할지는 프로젝트가 정해야 한다. 이 글의 내용은 현재 코드에서 확인한 것이며, 각 설계를 내린 당시의 판단은 별도로 확인하지 않았다.

관리해야 할 부분을 네 가지로 나눠 본다.

1. 상태 설계
2. 비동기 응답
3. 의존성
4. 빌드와 폐쇄망 배포

## 1. 상태 설계

React는 상태가 바뀌면 화면을 갱신하지만, 어떤 값을 상태로 둘지, 어디에 둘지는 개발자가 정한다. 파라미터 관리 화면(`ParamExplorer.jsx`)에서 이 판단이 드러난 부분은 다음과 같다.

**계산할 수 있는 값은 상태로 저장하지 않는다.**

같은 정보를 상태로 따로 저장하면 원본이 바뀔 때마다 둘을 함께 고쳐야 하고, 하나를 놓치면 서로 어긋난다. 이 화면에서 모듈이 연결된 프로젝트 수(`linkCounts`)와 현재 보여줄 모듈 목록(`currentModules`)은 상태가 아니라 트리 데이터에서 계산한다.

```jsx
// ParamExplorer.jsx (발췌)
const linkCounts = useMemo(() => {
  const m = new Map()
  tree.forEach(p => p.modules.forEach(mod => m.set(mod.id, (m.get(mod.id) || 0) + 1)))
  return m
}, [tree])

const currentModules = openProjectId === 'trash' ? unlinked : modulesOf(tree, openProjectId)
```

React 공식 문서의 'Effect가 필요하지 않은 경우'도 props나 state로 계산할 수 있는 값은 state에 넣지 말고 렌더링 중에 계산하도록 안내한다.

**상태의 초깃값도 설계 대상이다.**

트리에서 항목을 펼친 상태는 `expanded`에 저장하고, 토글은 `!e[id]`로 뒤집는다. 이때 값이 아직 없는(`undefined`) 항목은 "닫힘"으로 취급되므로, 처음 보이는 항목을 펼친 상태로 보여주려면 목록을 불러올 때 기본값을 채워 넣어야 한다. 코드의 주석에도 "시드 없이 두면 처음 클릭이 '닫힘→닫힘'이 돼 아무 반응 없어 보인다"고 적혀 있다. 화면에 보이는 모습과 상태에 저장된 값이 다를 수 있다는 점을 보여주는 사례다.

**서로 다른 의미의 상태는 나눈다.**

열어 둔 모듈(`openModuleId`)과 선택한 모듈(`selectedIds`)은 별도의 상태다. 상세 조회 대상과 일괄 작업 대상이 항상 같지는 않기 때문이다. 반대로 탐색 트리와 모듈 목록이 함께 쓰는 선택 정보는 부모 하나가 가진다(02-1 참고).

**상태를 바꿀 때는 복사한 새 값으로 바꾼다.**

선택 상태는 `Set`이고, 갱신할 때마다 `new Set(s)`로 복사한 뒤 수정한다.

```jsx
// ParamExplorer.jsx (발췌)
setSelectedIds(s => {
  const n = new Set(s)
  if (n.has(moduleId)) n.delete(moduleId); else n.add(moduleId)
  return n
})
```

React는 이전 값과 새 값이 같은 객체인지 비교해 변경 여부를 판단하므로, 원래 `Set`을 직접 수정하면 화면이 갱신되지 않을 수 있다. 이 원리는 이후 학습 글에서 다룬다.

**입력 폼은 초깃값이 바뀌면 컴포넌트를 새로 만든다.**

`ParamFormPanel`은 `useState(initial)`로 입력값을 저장한다. 이 초깃값은 처음 한 번만 쓰이므로, 연속 등록 기능에서 다음 입력값(이름 +1, offset 자동 증가)으로 폼을 바꾸려면 부모가 `key`를 바꿔 컴포넌트를 다시 만들어야 한다. 코드 주석에도 이 이유가 설명되어 있다. 상태를 새 값으로 되돌리는 방식을 직접 선택한 사례이며, 입력 폼 하나에도 이런 규칙이 필요하다는 것을 보여준다.

**상위 컴포넌트에 상태와 동작이 모인다.**

여러 컴포넌트가 쓰는 상태를 부모 한 곳에 모은 결과, 현재 `ParamExplorer.jsx`는 507줄이고 `useState`가 16개다. 모듈 목록(`ModuleGrid`)에 전달하는 props는 27개다. 상태가 한곳에 모여 있어 어떤 정보가 어디서 바뀌는지 추적하기는 쉽지만, 기능이 추가될 때마다 이 파일과 전달 목록이 함께 커진다. 이 구조를 어떻게 나눌지는 [한계와 선택을 바꿀 조건](#한계와-선택을-바꿀-조건)에서 다시 다룬다.

## 2. 비동기 응답

화면은 API 서버에 요청한 뒤 응답을 받아 상태를 바꾼다. 요청과 응답 사이에는 시간이 걸리고, 그 사이에 사용자가 다른 조작을 할 수 있다. 응답을 상태에 언제 반영할지는 개발자가 정해야 한다.

### 오래된 응답이 최신 화면을 덮어쓰는 문제

검색 조건을 연속으로 바꾸면 이전 요청이 끝나기 전에 다음 요청이 시작될 수 있다. 응답이 요청한 순서대로 돌아온다는 보장은 없으므로, 이전 검색 결과가 늦게 도착해 최신 결과를 덮어쓰면 현재 검색 조건과 다른 목록이 표시된다.

이 플랫폼은 요청마다 순번을 부여하고, 응답을 받았을 때 그 요청이 최신인지 확인한다.

```js
// web/src/lib/requestSequence.js (발췌)
const seqRef = useRef(0)
const begin = useCallback(() => ++seqRef.current, [])           // 새 요청의 번호
const isCurrent = useCallback((seq) => seq === seqRef.current, []) // 아직 최신인가
```

```js
// web/src/pages/DataDownload.jsx (발췌, 일부 생략)
const fetchData = useCallback(async (goPage) => {
  const seq = begin()                        // 요청마다 번호를 매긴다
  setLoading(true)
  try {
    const res = await api.get(`/api/datasets?${params}`)
    if (!isCurrent(seq)) return              // 더 최신 요청이 있으면 이 응답은 버린다
    setData(res.items.map(/* ... */))
  } catch (err) {
    if (!isCurrent(seq)) return              // 오래된 요청의 오류도 화면에 띄우지 않는다
    setError(err.message)
  } finally {
    if (isCurrent(seq)) setLoading(false)    // 로딩 표시도 최신 요청만 끈다
  }
}, [sort, begin, isCurrent])
```

- 응답 데이터뿐 아니라 오류 표시와 로딩 표시도 같은 검사를 거친다. 오래된 요청이 늦게 실패해서 최신 화면에 오류를 띄우거나, 로딩 표시를 먼저 끄는 일을 막기 위해서다.
- 이 훅(`useRequestSequence`)은 데이터 다운로드, 데이터 업로드, 공지사항, 로그 보기, 변경 이력, 재처리 화면 등 6개 파일에서 사용한다.
- 요청 자체를 취소하는 것은 아니다. 이전 요청도 서버에서는 그대로 처리되고, 화면이 그 결과를 무시할 뿐이다.

같은 문제는 모듈을 바꿀 때도 생긴다. `ParamExplorer`의 `loadParams`는 훅 대신 `selIdRef`에 현재 모듈 ID를 저장해 두고, 응답이 왔을 때 모듈이 바뀌었으면 결과를 버린다. 같은 문제를 푸는 방식이 두 가지(훅과 ref 비교)로 공존하고 있으므로, 화면이 늘어나면 하나로 정리할지 검토할 수 있다.

### 요청을 취소하는 경우

다운로드는 오래 걸릴 수 있어서, 진행 중에 사용자가 다른 화면으로 이동할 수 있다. 내보내기 화면(`Export.jsx`)은 `AbortController`를 만들어 취소 신호를 요청에 전달하고, 화면이 사라질 때 요청을 끊는다. 취소로 인한 `AbortError`는 오류 메시지로 띄우지 않는다.

목록 조회는 응답을 무시하고, 다운로드는 요청을 끊는다. 두 방식을 구분해 쓴 이유는 확인하지 못했다. `api.js`의 주석에는 서버의 생성기가 `GeneratorExit`로 정리되므로 화면에서는 신호만 전달하면 된다고 적혀 있다. 이 서버 쪽 정리가 실제로 일어나는지는 이 글에서 검증하지 않았으며, 취소 전파를 다루는 학습 글에서 확인할 항목이다.

## 3. 의존성

React 프로젝트의 의존성은 React 자체뿐 아니라 함께 쓰는 도구까지 포함한다. `web/package.json` 기준으로 실행 의존성은 6개, 개발·빌드 의존성은 10개다. `package-lock.json`에 기록된 패키지는 간접 의존성을 포함해 327개다.

확인한 사실과 확인하지 못한 사실을 나누면 다음과 같다.

| 항목 | 상태 |
| --- | --- |
| 직접 의존성의 버전 범위 | `package.json`에 `^` 범위로 기록 |
| 설치할 정확한 버전 | `package-lock.json`이 저장소에 포함되어 있음 |
| 간접 의존성 수 | 327개 (`package-lock.json` 기준) |
| 설치 용량, 빌드 시간 | 측정하지 않음 |
| 보안 업데이트와 버전 갱신 절차 | [확인 필요: 폐쇄망에서 어떤 주기와 절차로 갱신하는지] |

패키지를 하나 추가하면 그 패키지가 끌어오는 간접 의존성도 함께 늘어난다. 폐쇄망에서는 서버에서 패키지를 내려받을 수 없으므로, 의존성이 늘수록 반입하고 확인해야 할 대상도 늘어난다. 이 플랫폼에서 상태 관리 라이브러리, HTTP 라이브러리, 폼 라이브러리를 쓰지 않은 것은 의존성을 줄이는 효과도 있다. 다만 02-2에서 설명했듯이 이 선택을 당시에 의존성 때문에 했는지는 확인하지 않았다.

## 4. 빌드와 폐쇄망 배포

React 화면은 소스를 그대로 실행하지 않고, 빌드한 결과물(`web/dist`)을 배포한다. 이 결과물은 저장소에 포함되지 않는다(`.gitignore`에 `dist`가 있다). 그래서 배포 전에 빌드 과정이 따로 필요하다.

코드에서 확인한 배포 방식은 다음과 같다.

- 빌드한 화면은 별도의 웹 서버 없이 FastAPI가 정적 파일로 제공한다.
- `web/dist`의 `assets/`나 `index.html`이 없으면 정적 파일 제공을 건너뛰고 API만 실행한다.
- 해시가 붙은 `/assets/*`는 오래 캐시하고, `index.html`은 항상 새로 확인하게 해 새 배포가 반영되도록 한다.

폐쇄망 서버에서는 인터넷에 접속할 수 없으므로, 빌드에 필요한 도구와 패키지를 어디에서 준비할지가 문제가 된다. 저장소에는 이에 대한 문서가 두 가지 있고, 전제가 서로 다르다.

| 문서 | 설명하는 방식 |
| --- | --- |
| 기술 스택 문서(`TECH_STACK.md` §4) | 백엔드와 프론트엔드 의존성은 사내 미러나 사전에 내려받은 패키지로 오프라인 설치해야 한다고 기록 |
| Windows 설치·운영 설계안(`deploy/windows/README.md`) | 인터넷이 되는 빌드 PC에서 화면을 빌드하고, 결과물을 묶음(zip)과 SHA-256 해시로 서버에 반입한다. 서버에는 Node.js가 필요 없다고 기술 |

설계안의 방식이면 서버는 `npm` 패키지를 설치하지 않고 빌드된 결과물만 받으므로, 폐쇄망에서 준비할 대상은 빌드 PC의 패키지와 반입한 묶음으로 줄어든다.

다만 **이 설계안은 아직 구현하지 않은 문서다.** 문서 첫머리에 "설계안 — 아직 구현하지 않았다"고 적혀 있고, 설명하는 `build_release.ps1`, `install.ps1` 같은 스크립트는 이 커밋의 저장소에 없다(`deploy/windows/`에는 README만 있다). 따라서 이 글에서는 폐쇄망 배포를 "구현해서 운영 중인 절차"가 아니라 "설계한 방향"으로만 소개한다. 실제 구현과 검증, 운영 절차는 이 시리즈의 72편(폐쇄망 Windows 서버에 배포하고 운영하기)에서 다룬다.

[확인 필요: 현재 화면 배포를 실제로 어떻게 하고 있는지(빌드 위치, 결과물 반입 방법)]

## 한계와 선택을 바꿀 조건

아래는 현재 구조에서 다시 검토하게 될 조건이며, 이 글에서는 비교 실험을 하지 않았다.

| 상황 | 검토할 수 있는 방향 | 확인할 점 |
| --- | --- | --- |
| 상위 컴포넌트의 상태와 전달 props가 계속 늘어난다 | 관련 상태를 묶는 방식(`useReducer`), Context, 동작을 모은 커스텀 훅으로 나누기 | 나눈 뒤에도 같은 정보를 공유하는 컴포넌트가 하나의 출처를 보는지 |
| 요청 중복 제거나 캐시가 필요해진다 | 서버 데이터 조회를 도와주는 라이브러리 검토 | 현재 순번 방식이 처리하지 못하는 구체적인 문제가 있는지, 추가되는 의존성과 폐쇄망 반입 비용 |
| 순번 방식과 ref 비교 방식이 계속 공존한다 | 하나로 정리 | 두 방식의 동작이 같은지 |
| 지원 브라우저 정책이 바뀐다 | 빌드와 스타일 구성 다시 확인 | 02-2의 지원 정책 절 참고 |
| 배포 절차를 구현한다 | 설계안과 실제 절차의 차이 기록 | 시험 서버에서 끝까지 실행한 결과 |

## 참고 자료와 연결 글

- 이전 글: [02-1. 프론트엔드를 React로 선택한 이유](02-1-react.md), [02-2. React와 함께 사용하는 도구](02-2-react-tooling.md)
- 연결할 이론 글(연재 계획 기준): 집합, 상태의 단일 출처, 불변성, 논리적 순서 번호, 취소 전파, 폐쇄망 배포
- React 공식 문서: 'Effect가 필요하지 않은 경우' (https://ko.react.dev/learn/you-might-not-need-an-effect)
- 코드(커밋 고정): [ParamExplorer.jsx](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/src/pages/paramExplorer/ParamExplorer.jsx), [ParamFormPanel.jsx](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/src/pages/paramExplorer/ParamFormPanel.jsx), [requestSequence.js](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/src/lib/requestSequence.js), [DataDownload.jsx](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/src/pages/DataDownload.jsx#L36-L81), [Export.jsx](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/src/pages/Export.jsx#L352-L400), [api.js](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/web/src/lib/api.js), [main.py](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/api/main.py#L79-L122)
- 저장소 문서: [TECH_STACK.md §4](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/TECH_STACK.md), [deploy/windows/README.md](https://github.com/moon-in-the-sky/pdps/blob/f1894bf8ced8c503074e47b276d2b7c877a3cdf3/deploy/windows/README.md)
