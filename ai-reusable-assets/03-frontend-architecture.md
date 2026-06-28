# 🎨 프론트엔드(React / Next.js) 아키텍처 및 실장 룰

> Next.js(App Router) 및 React 환경에서 AI가 프론트엔드 소스코드를 작성할 때 준수해야 하는 공통 실장 규칙입니다.

## 🏗️ 1. 컴포넌트 설계 및 상태 관리 (Component & State Architecture)

### 📌 규칙 01. "use client" 선언 최소화 및 클라이언트 컴포넌트 격리

- **적용 가이드**: Next.js의 App Router 환경에서 파일 최상단에 `"use client"` 지시어를 선언하는 행위는 반드시 필요한 경우로만 엄격히 제한해야 합니다. 페이지 전체나 거대한 레이아웃 컴포넌트에 무분별하게 `"use client"`를 부여하는 것을 전면 금지합니다.
- **선언 필수 기준**: 아래의 기능이 구현에 직접적으로 요구되는 최하위 단말 컴포넌트에만 `"use client"`를 선언합니다.
  - 유저 인터랙션 이벤트 핸들러가 포함된 경우 (예: `onClick`, `onChange`, `onSubmit` 등)
  - 리액트 내장 상태 관리 및 라이프사이클 훅을 사용하는 경우 (예: `useState`, `useReducer`, `useEffect`, `useContext` 등)
  - 브라우저 전용 API를 직접 참조해야 하는 경우 (예: `window`, `document`, `localStorage` 등)
- **실장 방식**:
  - 데이터를 조회하고 정적인 화면을 그리는 상위 레이어는 순수 **서버 컴포넌트(Server Component)**로 유지하여 SEO 최적화 및 초기 로딩 성능을 극대화합니다.
  - 인터랙션이 필요한 UI(예: 토글 버튼, 입력 폼, 모달 등)는 별도의 파일로 잘게 쪼개어 독립된 클라이언트 컴포넌트로 캡슐화한 뒤, 서버 컴포넌트 하위에 자식으로 삽입(격리)하는 구조를 취합니다.

### 📌 규칙 02. API 클라이언트 함수의 집약 관리 및 환경 변수 기반 Base URL 운영

- **적용 가이드**: 리액트 컴포넌트 내부에서 `fetch`, `axios` 등의 HTTP 통신 라이브러리를 사용해 백엔드 API를 직접 호출하는 코드를 작성하는 것을 전면 금지합니다. 모든 외부 API 통신 로직은 지정된 전용 디렉토리(`lib/api/[도메인명].ts`)에 함수 형태로 추상화하여 집약 관리해야 합니다.
- **API Base URL 하드코딩 전면 금지 [최신 추가]**:
  - API 클라이언트나 통신 코드 내부에 백엔드 서버의 절대 주소(예: `https://example.com`)를 직접 문자열로 기재(Hardcoding)하는 행위를 전면 금지합니다.
  - 모든 API 호출의 기준 주소는 반드시 **환경 변수(`process.env.NEXT_PUBLIC_API_BASE_URL` 등)**를 통해서만 주입받도록 구현해야 합니다. (프론트엔드 보안 규칙 02번을 준수하여 비민감성 엔드포인트 주소만 브라우저 노출 허용)
- **실장 방식**:
  - 외부 API를 호출하는 비동기 함수를 `lib/api/` 하위에 도메인별로 개설하고 외부로 내보냅니다.
  - OpenAPI 스펙에서 자동 생성된 최신 타입스크립트 인터페이스를 이 클라이언트 함수에 엄격하게 주입하여 타입 안정성을 확보합니다.
  - 실제 컴포넌트에서는 이 집약된 API 클라이언트 함수만을 임포트하여 데이터 호출을 수행합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (주소가 하드코딩되어 환경 변화에 대응할 수 없는 구조) ❌
  export const fetchUserProfile = async () => {
    const response = await fetch("https://dev-server.com");
    return response.json();
  };

  // 좋은 예시 (환경 변수를 참조하여 DevContainer와 AWS 스테이징 환경에 유연하게 대응하는 구조) ⭕
  // lib/api/user.ts
  import { UserProfileResponse } from "@/types/generated/api";

  export const fetchUserProfile = async (): Promise<UserProfileResponse> => {
    // 환경 변수에서 Base URL을 안전하게 로드
    const baseUrl = process.env.NEXT_PUBLIC_API_BASE_URL;

    const response = await fetch(`${baseUrl}/v1/users/me`, {
      headers: { "Content-Type": "application/json" },
    });
    return response.json();
  };
  ```

### 📌 규칙 03. 서버 컴포넌트 데이터 페칭 및 클라이언트 컴포넌트 Props 전달 패턴 (Server-Fetch, Client-Props)

- **적용 가이드**: Next.js(App Router) 환경에서 화면을 구성할 때, 데이터의 호출(Fetching)과 인터랙션의 처리를 명확히 격리해야 합니다. 데이터를 서버 단에서 최우선으로 패치하고, 이 데이터를 클라이언트 컴포넌트의 `Props`로 하향식 전달하는 구조를 엄격히 준수합니다.
- **아키텍처 목적**:
  - **보안 및 성능**: 백엔드 API 호출 및 직접적인 DB 쿼리를 브라우저가 아닌 서버 환경에서 실행하여 보안 민감 정보 노출을 차단하고, 클라이언트 자바스크립트 번들 크기를 줄여 초기 로딩 속도를 최적화합니다.
  - **역할 분담**: 서버 컴포넌트는 '데이터 준비'를 전담하고, 클라이언트 컴포넌트는 'UI 인터랙션 및 상태 제어'만 전담하도록 관심사를 분리합니다.
- **실장 방식**:
  - 외부 데이터를 읽어와야 하는 페이지(Page)나 대형 레이아웃은 비동기 **서버 컴포넌트(`async/await`)**로 구현하여 필요한 데이터를 먼저 패치합니다.
  - 유저 이벤트 핸들러나 상태 관리가 필요한 UI 섹션은 `"use client"`가 선언된 독립 컴포넌트로 쪼개고, 서버 컴포넌트에서 조회한 데이터를 이 클라이언트 컴포넌트의 **정적 `Props`**로 주입하여 활성화합니다.
- **구조 예시**:

  ```tsx
  // 좋은 예시 (서버가 데이터를 가져오고 클라이언트가 전달받아 렌더링하는 이상적인 구조) ⭕

  // 1. components/KnowledgeListClient.tsx (클라이언트 컴포넌트)
  "use client";
  import { ZibaldonePost } from "@/types/generated/api";

  interface ClientProps {
    initialPosts: ZibaldonePost[];
  }
  export default function KnowledgeListClient({ initialPosts }: ClientProps) {
    const [filter, setFilter] = useState("");
    // Props로 전달받은 신뢰할 수 있는 데이터를 기반으로 상태 제어 및 UI 렌더링
    return (
      <div>
        <button onClick={() => setFilter("insights")}>필터링</button>
        {initialPosts.map((post) => (
          <div key={post.id}>{post.title}</div>
        ))}
      </div>
    );
  }

  // 2. app/zibaldone/page.tsx (서버 컴포넌트)
  import { fetchZibaldonePosts } from "@/lib/api/zibaldone"; // 서버용 API 클라이언트
  import KnowledgeListClient from "@/components/KnowledgeListClient";

  export default async function ZibaldonePage() {
    // 서버 사이드에서 안전하고 빠르게 데이터를 직접 페칭
    const postsData = await fetchZibaldonePosts();

    // 클라이언트 컴포넌트에 Props로 안전하게 전달
    return <KnowledgeListClient initialPosts={postsData} />;
  }
  ```

### 📌 규칙 04. 최상위 서버 컴포넌트 중심의 단일 데이터 페칭 및 하위 중복 페칭 금지

- **적용 가이드**: 특정 화면이나 유기적으로 얽힌 기능 영역을 구성할 때, 데이터 페칭은 반드시 **최상위 서버 컴포넌트(Top-level Server Component)에서 단 1회만 실행**되어야 합니다. 최상위에서 내려준 데이터를 활용하지 않고, 하위 자식 컴포넌트들이 각자 독립적으로 동일하거나 유사한 데이터를 중복으로 호출(Duplicate Fetching)하는 구조를 전면 금지합니다.
- **성능 및 데이터 일관성 목적**:
  - 중복 네트워크 요청으로 인한 백엔드/인프라 부하 및 불필요한 서버 리소스 소모를 원천 차단합니다.
  - 데이터의 원천을 단 하나로 묶어(Single Source of Truth), 화면 전체를 구성하는 자식 컴포넌트들이 언제나 완벽하게 일치된 최신 데이터 정합성을 유지하도록 보장합니다.
- **실장 방식**:
  - 하위 컴포넌트에 데이터가 필요할 경우, 상위 서버 컴포넌트에서 페칭한 데이터 객체나 배열을 쪼개어 하향식 **Props Drilling(또는 적절한 구조적 Composition)** 방식을 통해 안전하게 배분합니다.
  - 하위 컴포넌트가 클라이언트 컴포넌트일지라도 내부 `useEffect`나 이벤트 시점 외에 초기 렌더링을 위한 자체 독립 fetch 코드를 작성하는 것을 엄격히 제한합니다.

### 📌 규칙 05. 서버와 클라이언트 컴포넌트의 물리적 파일 분리 및 경계 명확화

- **적용 가이드**: 단 하나의 파일 내에서 서버 컴포넌트 로직과 `"use client"`가 필요한 클라이언트 컴포넌트 로직을 혼재하여 작성하는 행위를 전면 금지합니다. 두 영역의 컴포넌트는 반드시 **독립된 별도의 파일 단위로 엄격히 분리**되어야 합니다.
- **아키텍처 목적**: 파일 시스템 수준에서 컴포넌트의 실행 환경(Server/Client Context)을 명확히 분리하여, 개발자와 생성형 AI가 소스코드를 오독하거나 잘못된 의존성(의도치 않은 클라이언트 사이드 번들 포함 등)을 주입하는 실수를 예방하기 위함입니다.
- **실장 방식**:
  - **서버 컴포넌트**: 디렉토리의 기본 진입점(예: `page.tsx`, `layout.tsx`)이나 데이터 페칭 전담 컴포넌트는 파일 최상단에 어떠한 지시어도 넣지 않은 채 순수 서버 컴포넌트 파일로 유지합니다.
  - **클라이언트 컴포넌트**: 유저 인터랙션이나 상태 관리가 필요한 UI 요소는 반드시 독립된 별도의 파일(예: `CounterButton.tsx`, `SearchForm.tsx`)로 생성하고, 파일의 **최상단 첫 번째 라인에 반드시 `"use client"` 지시어를 명시**합니다.
  - **결합 패턴**: 상위 서버 컴포넌트 파일에서 분리된 클라이언트 컴포넌트 파일을 임포트(Import)하여 하향식으로 조립(Composition)하는 구조만을 허용합니다.

### 📌 규칙 06. 공통 컴포넌트의 도메인 로직 분리 및 순수 UI 컴포넌트화

- **적용 가이드**: 여러 화면이나 도메인에서 공유하여 사용하는 공통 컴포넌트(예: `components/common/` 또는 `components/ui/` 하위 파일) 내부에 특정 비즈니스 도메인 지식, 전역 상태, 혹은 특정 백엔드 API 클라이언트 함수를 직접 참조하여 구현하는 행위를 전면 금지합니다.
- **아키텍처 목적**: 공통 컴포넌트가 특정 도메인에 종속(Coupling)되는 것을 방지하여, 프로젝트 내 어떠한 화면이나 향후 다른 프로젝트(AI 자산 이식)에서도 수정 없이 즉시 재사용할 수 있는 완전한 독립성과 범용성을 보장하기 위함입니다.
- **실장 방식**:
  - 공통 컴포넌트는 오직 외부에서 전달받는 정적/동적 **`Props`(데이터, 스타일, 이벤트 핸들러 등)에 의해서만 동작하는 순수 UI 컴포넌트(Presentational Component)**로 설계합니다.
  - 컴포넌트 내부에 특정 비즈니스 데이터의 형태를 고정하여 명시하지 않으며, 제네릭(`Generic`) 타입을 활용하거나 원시 타입(Primitive Type) 위주로 Props를 구성합니다.
- **구조 예시**:

  ```tsx
  // 나쁜 예시 (공통 버튼인데 '회원가입' 도메인 로직과 특정 API에 종속되어 재사용 불가) ❌
  import { registerUserApi } from "@/lib/api/user";
  export default function CommonButton() {
    return <button onClick={registerUserApi}>제출하기</button>;
  }

  // 좋은 예시 (순수 UI 책임만 가짐으로써 어디서나 쓸 수 있는 완전한 공통 컴포넌트) ⭕
  interface ButtonProps {
    label: string;
    onClick: () => void;
    variant?: "primary" | "secondary";
  }
  export default function CommonButton({
    label,
    onClick,
    variant = "primary",
  }: ButtonProps) {
    return (
      <button onClick={onClick} className={`btn-${variant}`}>
        {label}
      </button>
    );
  }
  ```

### 📌 규칙 07. 공통 컴포넌트의 스타일 내부 완결 및 외부 className 덮어쓰기 금지

- **적용 가이드**: 여러 화면에서 공유하는 공통 UI 컴포넌트 인터페이스 설계 시, 외부에서 임의의 스타일 클래스명(`className`)을 문자열로 전달받아 컴포넌트 내부의 핵심 스타일을 강제로 덮어쓰는(Override) 속성 확장을 전면 금지합니다.
- **아키텍처 및 디자인 목적**:
  - **디자인 시스템 일관성 유지**: 프로젝트 전반의 핵심 UI 요소(버튼 색상, 패딩, 테두리 등)가 화면마다 자의적으로 변형되는 것을 차단하여 시각적 정합성을 보장합니다.
  - **스타일 충돌(Cascading) 예방**: 외부에서 주입된 클래스와 내부 클래스 간의 CSS 우선순위 경합으로 인해 UI가 깨지거나 기형적으로 렌더링되는 마진/패딩 버그를 원천 차단합니다.
- **실장 방식 (Variant 패턴 필수 적용)**:
  - 공통 컴포넌트의 스타일은 철저히 파일 내부(Tailwind CSS 조합 등)에서 완결 캡슐화합니다.
  - 크기, 색상, 테두리 등의 디자인 변형이 필요할 경우, 날것의 `className`을 받지 말고 **`variant`, `size` 등 명확한 규격 속성(Props Option)**을 정의합니다. 외부에서는 정의된 옵션 값(예: `variant="primary" | "danger"`, `size="sm" | "lg"`)만 선택할 수 있도록 강제합니다.
- **구조 예시**:

  ```tsx
  // 나쁜 예시 (외부에서 마진이나 색상을 마음대로 주입하여 컴포넌트 고유 스타일을 파괴함) ❌
  <CommonButton className="bg-red-500 mt-10 ml-5 text-xl" label="삭제" />;

  // 좋은 예시 (허용된 규격 옵션 내에서만 변형을 허용하고 스타일은 내부에서 통제하는 구조) ⭕
  // components/common/CommonButton.tsx
  interface ButtonProps {
    label: string;
    variant?: "primary" | "danger";
    size?: "sm" | "md";
  }
  export default function CommonButton({
    label,
    variant = "primary",
    size = "md",
  }: ButtonProps) {
    const baseStyle = "rounded font-medium transition-colors";
    const variants = {
      primary: "bg-blue-600 text-white",
      danger: "bg-red-600 text-white",
    };
    const sizes = { sm: "px-2 py-1 text-sm", md: "px-4 py-2 text-base" };

    return (
      <button className={`${baseStyle} ${variants[variant]} ${sizes[size]}`}>
        {label}
      </button>
    );
  }
  ```

  ### 📌 규칙 07-2. CSS 클래스 조건 분기 시 clsx / cn 유틸 함수 사용 통일 (Conditional Class Names)

* **적용 가이드**: 리액트 컴포넌트 내부에서 상태(State)나 Props에 따라 CSS 클래스명(특히 Tailwind CSS)을 동적으로 결합하고 조건 분기할 때, 백틱 템플릿 리터럴 내부에 삼항 연산자나 논리 연산자(`&&`)를 날것으로 나열하는 행위를 전면 금지합니다. 모든 클래스 조건 분기는 반드시 **`clsx`** 또는 프로젝트 공통 **`cn` 유틸 함수(clsx와 tailwind-merge의 조합형)**로 통일하여 참조해야 합니다.
* **실장 및 품질 목적**:
  - **DOM 오염 방지**: 조건문이 무시될 때 클래스명 자리에 `false`나 `undefined` 문자열이 그대로 렌더링되어 HTML 마크업이 오염되는 현상을 차단합니다.
  - **스타일 충돌 원천 차단**: 동적 클래스 결합 시 발생하는 불필요한 공백 문자 중복을 제거하고, 테일윈드 클래스 간의 우선순위 경합을 `cn` 유틸리티 내부에서 안전하게 병합(tailwind-merge)하여 UI가 기형적으로 깨지는 버그를 예방합니다.
* **구조 예시**:

  ```tsx
  // 나쁜 예시 (백틱 내 삼항 연산자가 엉켜 읽기 힘들고 false 글자가 노출될 위험이 있는 구조) ❌
  <div
    className={`box-style ${isActive ? "bg-blue-500" : "bg-gray-200"} ${isError && "border-red-500"}`}
  />;

  // 좋은 예시 (cn 유틸을 통해 스타일 분기의 의도가 선언적으로 간결하게 드러나는 구조) ⭕
  import { cn } from "@/lib/utils/cn"; // 프로젝트 내 표준 cn 유틸 함수 적용

  interface BoxProps {
    isActive: boolean;
    isError: boolean;
  }
  export default function StatusBox({ isActive, isError }: BoxProps) {
    return (
      <div
        className={cn(
          "box-style bg-gray-200 border border-transparent", // 1. 기본 스타일 배치
          isActive && "bg-blue-500 text-white", // 2. 활성화 조건 결합
          isError && "border-red-500 bg-red-50", // 3. 에러 조건 결합 (bg 중복은 cn이 자동 최적화)
        )}
      />
    );
  }
  ```

### 📌 규칙 08. 신규 공통 컴포넌트 추가 시 기존 파일 중복 조사 의무화 (No Duplicate UI Components)

- **적용 가이드**: AI가 UI 구현 중 필요에 의해 새로운 공통 컴포넌트(예: 버튼, 입력 폼, 모달, 탭 등)를 신규 생성하려고 할 때, 임의로 파일부터 생성하는 행위를 전면 금지합니다. 반드시 프로젝트 내에 이미 존재하는 공통 UI 자산이 있는지 선행 조사해야 합니다.
- **중복 확인의 목적**: 동일하거나 유사한 기능을 하는 UI 컴포넌트가 프로젝트 내부(`components/common/` 또는 `components/ui/` 등)에 중복 생성되어 디자인 시스템이 파편화되고 소스코드의 유지보수 비용이 불필요하게 증가하는 것을 원천 차단하기 위함입니다.
- **AI 필수 준수 사항**:
  - 새로운 공통 컴포넌트 코드를 작성하기 전, 프로젝트의 공통 컴포넌트 디렉토리 내부의 파일 목록과 명칭을 전체적으로 탐색(Read Directory)합니다.
  - 만약 기존에 구현된 컴포넌트 중 활용 가능한 자산이 있다면 신규 생성 대신 **기존 컴포넌트를 가져와 재사용**하거나, 약간의 기능 확장이 필요한 경우 기존 컴포넌트에 새로운 `Variant`나 `Size` 옵션(프론트엔드 규칙 07번 준수)을 안전하게 추가하여 확장하는 방향으로 설계합니다.
- **실장 및 제어 방식**: AI는 분석 결과 기존에 유사한 컴포넌트가 전혀 없어 신규 생성이 반드시 필요하다고 판단될 때만, 개발자에게 생성 목적과 파일명을 제안하고 최종 확인(승인)을 받은 후에 구현에 착수합니다.

### 📌 규칙 08. 페이지 컴포넌트의 책임 최소화 및 비즈니스 로직의 커스텀 훅 분리

- **적용 가이드**: Next.js(App Router)의 각 라우팅 진입점이 되는 페이지 컴포넌트(`page.tsx`) 내부에 클라이언트 사이드의 복잡한 비즈니스 로직, 상태 선언, 혹은 데이터 변환 계산 로직을 직접 길게 나열하여 구현하는 행위를 전면 금지합니다.
- **아키텍처 목적**: 페이지 컴포넌트를 가볍고 직관적인 상태로 유지하여 화면의 전체적인 레이아웃 구조를 단 3초 만에 파악할 수 있도록 가독성을 높이고, 비즈니스 로직을 독립된 훅으로 격리하여 프론트엔드 기능 테스트(Unit Test)를 손쉽게 수행하기 위함입니다.
- **실장 방식**:
  - **페이지 컴포넌트의 책임**: 최상위 서버 컴포넌트로서 데이터를 최초 1회 패치(프론트엔드 규칙 03, 04 준수)하고, 하위 컴포넌트들을 배치하여 전체적인 **화면의 레이아웃(Layout Structure)을 선언적으로 구성**하는 역할만 담당합니다.
  - **커스텀 훅의 책임**: 화면을 제어하는 복잡한 상태(`useState`/`useReducer`), 클라이언트 사이드 데이터 가공, 폼 제출 및 유저 이벤트 핸들러 등의 **실질적인 비즈니스 로직은 전용 커스텀 훅(Custom Hook, 예: `use[기능명]Logic` 또는 `hooks/`)으로 추출하여 완벽히 분리**합니다. 페이지나 하위 클라이언트 컴포넌트에서는 이 커스텀 훅이 반환하는 상태와 함수를 구조 분해 할당하여 선언적으로 사용하기만 합니다.
- **구조 예시**:

  ```tsx
  // 나쁜 예시 (페이지 내부에 상태와 통신 처리 로직이 얽혀 복잡도가 높은 구조) ❌
  export default function BadPage() {
    const [data, setData] = useState([]);
    const [loading, setLoading] = useState(false);
    const handleAction = () => { ... }; // 복잡한 로직 나열
    return <main>{/* 수백 줄의 레이아웃 소스 */}</main>;
  }

  // 좋은 예시 (책임이 깔끔하게 격리되어 가독성과 테스트 용이성이 확보된 구조) ⭕
  // 1. hooks/useZibaldoneLogic.ts (순수 비즈니스 로직만 담당하는 커스텀 훅)
  export function useZibaldoneLogic(initialPosts) {
    const [filter, setFilter] = useState('all');
    const filteredPosts = initialPosts.filter(p => p.type === filter);
    const handleSave = async (data) => { ... };
    return { filter, setFilter, filteredPosts, handleSave };
  }

  // 2. app/zibaldone/page.tsx (구조와 페칭만 정의하는 가벼운 페이지)
  export default async function ZibaldonePage() {
    const postsData = await fetchZibaldonePosts(); // 서버 페칭 (규칙 03 준수)
    return <ZibaldoneView initialPosts={postsData} />; // 선언적 레이아웃 배정
  }
  ```

### 📌 규칙 09. 커스텀 훅의 비동기 상태 세트 객체 반환 표준화 (Standardized Hook State Object)

- **적용 가이드**: 비즈니스 로직과 비동기 데이터 통신을 담당하는 커스텀 훅(Custom Hook)을 구현할 때, 조회된 데이터와 제어 상태들(Loading, Error)을 개별적인 단일 변수나 배열 형태로 파편화하여 반환하는 행위를 전면 금지합니다. 반드시 규격화된 하나의 **세트 객체(State Set Object) 형식**으로 묶어서 반환해야 합니다.
- **아키텍처 목적**:
  - 한 컴포넌트 내에서 복수의 커스텀 훅을 호출할 때 발생할 수 있는 변수명 충돌(네임스페이스 오염)을 방지하고 구조 분해 할당을 통해 가독성을 높입니다.
  - 프론트엔드 규칙 2-01(데이터 3대 상태 UI 구현)과의 결합성을 높여, 데이터를 소비하는 컴포넌트가 로딩/에러/성공 상태를 오차 없이 안전하게 제어하도록 강제합니다.
- **실장 방식**:
  - 비동기 훅의 반환 객체는 기본적으로 `{ data, isLoading, error, ...행동함수들 }` 패턴의 단일 객체 구조를 취하도록 규격화합니다.
  - 데이터가 비어있는 상태(Empty) 판별 역시 훅 내부에서 미리 연산된 상태(예: `isEmpty`)나 데이터 자체의 길이(Length)를 통해 반환 객체 내에서 직관적으로 유추할 수 있도록 설계합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (반환 형식이 배열이거나 파편화되어 다른 훅과 섞일 때 변수명이 충돌함) ❌
  const [data, loading, error] = useZibaldoneList();
  const [userData, userLoading, userError] = useUserStatus(); // 매번 이름을 새로 지어야 함

  // 좋은 예시 (세트 객체로 묶여 컴포넌트 단에서 직관적이고 깔끔하게 소비되는 구조) ⭕
  // 1. hooks/useZibaldonePosts.ts
  export function useZibaldonePosts(initialPosts) {
    const [posts, setPosts] = useState(initialPosts);
    const [isLoading, setIsLoading] = useState(false);
    const [error, setError] = useState<Error | null>(null);

    const handleRefresh = async () => { ... };

    // 상태와 로직 함수를 명확한 단일 세트 객체로 반환
    return {
      posts,
      isLoading,
      error,
      isEmpty: posts.length === 0,
      handleRefresh
    };
  }

  // 2. components/KnowledgeView.tsx
  export default function KnowledgeView({ initialPosts }) {
    // 구조 분해 할당 시 콜론(:)을 이용해 깔끔하게 별칭 지정 가능 및 3대 UI 상태와 즉시 매핑
    const { posts, isLoading, error, isEmpty } = useZibaldonePosts(initialPosts);

    if (isLoading) return <SkeletonScreen />; // 프론트엔드 규칙 2-01 준수
    if (error) return <ErrorRetryBanner error={error} />;
    if (isEmpty) return <EmptyNotice />;

    return <div>{posts.map(post => <Card key={post.id} data={post} />)}</div>;
  }
  ```

### 📌 규칙 10. 커스텀 훅 복잡도 제어 및 하위 훅(Sub-Hook) 분리 강제

- **적용 가이드**: 비즈니스 로직을 격리하기 위해 생성한 커스텀 훅(Custom Hook)의 내부 복잡도가 일정 기준을 초과할 경우, 단일 훅 내에 코드를 누적하는 행위를 전면 금지합니다. 반드시 기능별로 역할을 쪼갠 **하위 커스텀 훅(Sub-Hook)으로 재분리**하여 조립하는 구조를 취해야 합니다.
- **하위 훅 전면 분리 기준 (2대 조건)**:
  1.  **상태 갯수 과다**: 훅 내부에서 선언된 독립적인 상태(`useState` 또는 `useReducer`)의 개수가 **3개 이상**인 경우
  2.  **독립적 API 통신**: 단일 훅 내부에서 서로 다른 도메인이나 성격이 완전히 분리된 **비동기 API 호출 로직이 존재**하는 경우
- **실장 방식**:
  - 컴포넌트가 직접 호출하는 최상위 커스텀 훅(주로 페이지나 대형 뷰 전용 훅)은 스스로 모든 상태를 직접 관리하지 않습니다.
  - 데이터 통신이나 특정 도메인 상태 제어는 하위 훅(예: `useUserFetch`, `useFilterState`)으로 도려내어 각각 독립적으로 캡슐화합니다.
  - 최상위 훅은 분리된 하위 훅들이 반환하는 표준 세트 객체(프론트엔드 규칙 09 준수)들을 임포트하여 최종적으로 조합(Composition) 및 중계하는 역할만 담당합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (하나의 훅에 유저 정보 조회, 게시글 조회, 모달 상태 등 상태가 5개나 얽혀 복잡함) ❌
  export function useZibaldonePageLogic() {
    const [user, setUser] = useState(null);
    const [posts, setPosts] = useState([]);
    const [isLoading, setIsLoading] = useState(false);
    const [isOpen, setIsOpen] = useState(false);
    const [modalType, setModalType] = useState("create");
    // ...수많은 useEffect와 복잡한 API 호출들이 뒤섞임
  }

  // 좋은 예시 (각 상태와 통신 책임을 하위 훅으로 격리하고, 최상위 훅은 조립만 하는 구조) ⭕
  // 1. 하위 훅 A: 유저 상태 및 통신 담당
  function useUserContext() {
    const [user, setUser] = useState(null);
    return { user, isLoggedIn: !!user };
  }
  // 2. 하위 훅 B: 게시글 목록 및 통신 담당 (프론트엔드 규칙 09 준수)
  function useFetchPosts() {
    const [posts, setPosts] = useState([]);
    const [isLoading, setIsLoading] = useState(false);
    return { posts, isLoading };
  }
  // 3. 하위 훅 C: UI 모달 상태 제어 담당
  function useModalControl() {
    const [isOpen, setIsOpen] = useState(false);
    const [type, setType] = useState("create");
    return {
      isOpen,
      type,
      open: () => setIsOpen(true),
      close: () => setIsOpen(false),
    };
  }

  // 4. 최상위 조립 훅: 컴포넌트는 오직 이 훅만 바라보고 직관적으로 소비
  export function useZibaldonePageLogic() {
    const { user, isLoggedIn } = useUserContext();
    const { posts, isLoading } = useFetchPosts();
    const modal = useModalControl();

    return {
      userState: { user, isLoggedIn },
      postState: { posts, isLoading },
      modal,
    };
  }
  ```

### 📌 규칙 11. 데이터 포맷팅 로직의 공통 유틸리티 집약 관리 (Standardized Data Formatters)

- **적용 가이드**: 화면에 렌더링되거나 외부로 출력되는 날짜(Date), 금액(Currency), 수량(Quantity/Number) 등의 데이터를 날것의 소스코드 내에서 자의적인 인라인 함수나 임의의 정규식 스트링으로 가공하는 행위를 전면 금지합니다. 모든 포맷팅 연산은 반드시 지정된 공통 유틸리티 파일에 집약하여 참조해야 합니다.
- **표준 파일 배치 경로**:
  - **`lib/utils/formatter.ts`** 또는 **`utils/format.ts`**
- **아키텍처 및 비즈니스 목적**:
  - **표준화된 UI/UX 제공**: 시스템 전반의 핵심 지표(예: 일본 현장 엔화 표기 `¥1,000`, 날짜 표기 `2026/06/28`)가 화면마다 다르게 표현되는 파편화를 원천 차단합니다.
  - **유지보수 단일화**: 비즈니스 정책 변경으로 인해 글로벌 표준 표기법이나 통화 단위가 바뀌더라도, 수많은 컴포넌트 파일을 수정하지 않고 공통 포맷터 파일 하나만 수정하여 리워크(Rework)를 최소화합니다.
- **실장 방식**:
  - 숫자나 날짜 문자열을 입력받아 정제된 포맷 스트링을 반환하는 순수 함수(Pure Function)들을 `lib/utils/formatter.ts`에 선언하고 내보냅니다(`export`).
  - 컴포넌트나 비즈니스 로직에서는 이 공통 포맷터 함수를 임포트(Import)하여 데이터의 가시성을 확보합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (콤마 처리를 위해 컴포넌트 내부에 정규식을 직접 작성하여 재사용이 불가능한 구조) ❌
  <span>{price.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")}엔</span>

  // 좋은 예시 (비즈니스 규격을 한곳에서 완벽히 통제하고 선언적으로 사용하는 구조) ⭕
  // lib/utils/formatter.ts
  export const formatJapaneseCurrency = (amount: number): string => {
    return new Intl.NumberFormat('ja-JP', { style: 'currency', currency: 'JPY' }).format(amount);
  };
  export const formatStandardDate = (dateString: string): string => {
    // YYYY/MM/DD 형태로 일관되게 정제
    const date = new Date(dateString);
    return date.toLocaleDateString('ja-JP', { year: 'numeric', month: '2-digit', day: '2-digit' });
  };

  // components/OrderCard.tsx
  import { formatJapaneseCurrency, formatStandardDate } from '@/lib/utils/formatter';

  export default function OrderCard({ order }) {
    return (
      <div>
        <p>주문일: {formatStandardDate(order.createdAt)}</p>
        <span>결제금액: {formatJapaneseCurrency(order.totalPrice)}</span>
      </div>
    );
  }
  ```

---

## 🎨 2. 데이터 기반 UI/UX 실장 규칙 (Data-Driven UX Rules)

### 📌 규칙 01. 비동기 데이터 3대 상태(Loading, Error, Empty) 구현 의무화

- **적용 가이드**: 외부 API나 데이터베이스에서 비동기로 데이터를 가져와 화면에 렌더링하는 모든 컴포넌트는 사용자의 인지적 흐름을 방해하지 않도록 **1) 로딩 중, 2) 에러 발생, 3) 데이터 없음**의 3가지 상태에 대한 전용 UI를 반드시 독립적으로 구현해야 합니다. 데이터가 성공적으로 존재할 때의 화면만 생성하고 예외 상태를 방치하는 것을 전면 금지합니다.
- **상태별 UI/UX 필수 구현 기준**:
  1.  **로딩 중 (Loading State)**: 데이터를 가져오는 동안 화면이 멈추거나 깨져 보이지 않도록 스피너(Spinner), 스켈레톤 UI(Skeleton Screen), 또는 최소한의 가이드 텍스트(예: `읽어오는 중...`)를 노출합니다.
  2.  **에러 발생 (Error State)**: 네트워크 단절이나 백엔드 에러 발생 시 화면이 하얗게 굳는(White Screen) 현상을 차단합니다. 사용자 친화적인 안내 메시지와 함께, 유저가 스스로 조치할 수 있는 **[다시 시도하기(Retry)] 버튼**을 반드시 함께 배치합니다.
  3.  **데이터 없음 (Empty State)**: 통신은 성공했으나 반환된 배열이나 객체가 비어있을 경우(예: 목록이 0건일 때), 빈 껍데기 화면만 보여주지 않고 `"등록된 노하우가 없습니다"`, `"조회된 내역이 없습니다"` 등 데이터가 없는 상태임을 인지시키는 안내 문구와 그래픽을 명시합니다.
- **실장 방식**: 리액트 컴포넌트 내부에서 조건부 렌더링(`if` 분기문)을 활용하여 성공 화면(Success UI)을 그리기 전 단계에서 이 3가지 예외 상태를 최상단에서 조기 처리(Early Return 패턴 적용)하도록 코드를 구조화합니다.

### 📌 규칙 02. 양식 제출 시 버튼 비활성화 및 finally 블록 내 해제 필수 (Prevent Double Submission)

- **적용 가이드**: 데이터 생성(Post), 수정(Put), 삭제(Delete) 등 백엔드의 상태를 변화시키는 요청을 보내는 모든 제출(Submit) 및 클릭 이벤트 핸들러는 API 통신이 완료될 때까지 관련 버튼을 반드시 **비활성화(`disabled`)** 처리해야 합니다.
- **이중 송신 방지의 목적**: 네트워크 지연 중 유저가 버튼을 여러 번 연타하여 동일한 데이터가 중복 등록되거나, 결제가 중복으로 발생하는 시스템 트랜잭션 오류를 프론트엔드 레벨에서 원천 차단하기 위함입니다.
- **실장 방식**:
  - 요청이 시작되는 즉시 로딩 상태(`isLoading` 등)를 `true`로 전환하여 버튼의 `disabled` 속성에 바인딩합니다.
  - 비동기 통신 로직은 반드시 `try-catch-finally` 구조로 작성해야 하며, **성공/실패 여부와 상관없이 통신이 종료되면 무조건 실행되는 `finally` 블록 내부에서 로딩 상태를 `false`로 되돌려 버튼 비활성화를 확실하게 해제**해야 합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (에러가 나면 버튼이 영원히 잠기거나, 연타 시 중복 요청이 날아감) ❌
  const handleSubmit = async () => {
    await saveZibaldonePost(formData);
    setIsLoading(false);
  };

  // 좋은 예시 (어떤 예외 상황에서도 이중 송신을 막고 복구되는 구조) ⭕
  const handleSubmit = async () => {
    if (isLoading) return; // 가드 락(Guard Lock) 추가

    try {
      setIsLoading(true); // 버튼 비활성화 (disabled={isLoading})
      await saveZibaldonePost(formData);
      showSuccessToast("지식이 안전하게 보관되었습니다.");
    } catch (error) {
      handleError(error);
    } finally {
      setIsLoading(false); // [필수] 어떤 상황에서도 최종적으로 버튼 잠금 해제
    }
  };
  ```

  - **실장 방식**:
  * **[catch 블록 필수 구현]**: 백엔드와 통신하는 모든 프론트엔드 비동기 함수 및 클릭/제출 이벤트 핸들러는 반드시 `try-catch-finally` 구조로 감싸야 합니다. 예외 처리가 누락되어 브라우저 런타임 에러가 발생하는 것을 전면 금지합니다.
  * 요청이 시작되면 비활성화 락을 걸고, `catch` 블록 진입 시에는 사용자 친화적인 에러 UI 상태(프론트엔드 규칙 2-01 준수)를 활성화하거나 알림을 띄우며, 성공/실패 여부와 상관없이 `finally` 블록에서 버튼 잠금을 확실하게 해제합니다.

### 📌 규칙 03. 컨텍스트 기반 필드 에러 메시지의 입력창 직하단 배치 및 role="alert" 부여 [최신 수정]

- **적용 가이드**: 사용자가 폼 양식(Form)의 특정 필드에 값을 잘못 입력하여 유효성 검증(Validation) 실패 에러가 발생했을 때, 해당 에러 메시지를 화면 최상단이나 공통 팝업/토스트로 뭉뚱그려 표시하는 것을 금지합니다. 에러 메시지는 반드시 **문제가 발생한 해당 입력 창(Input Field) 바로 밑(직하단)**에 개별적으로 표시해야 합니다.
- **스크린 리더 배려 및 프로젝트 전체 통일성**:
  - 화면에 동적으로 렌더링되는 모든 필드 에러 메시지 HTML 태그에는 반드시 **`role="alert"` 속성**을 필수로 부여해야 합니다.
  - 이를 통해 시각장애인 등 보조공학 기기(스크린 리더) 사용자가 화면의 다른 영역을 탐색 중이더라도, 에러가 발생한 즉시 최우선 음성 경고 피드백을 전달받을 수 있도록 프로젝트 전반의 접근성 표준을 통일합니다.
- **실장 방식**:
  - 입력 컴포넌트나 폼 라이브러리를 구현할 때, 개별 필드 구조를 `[레이블] - [입력창] - [에러 메시지 영역]` 단위로 컴포넌트화합니다.
  - 백엔드나 발리데이터(Validator)로부터 전달받은 에러 객체에서 해당 필드의 에러 유무를 판별하여, 에러가 존재할 때만 직하단에 붉은색 텍스트 스팬 태그(예: `<span role="alert">...</span>`) 형태로 에러 문구를 렌더링합니다.
- **구조 예시**:

  ```tsx
  // 나쁜 예시 (스크린 리더가 에러 발생 여부를 실시간으로 인지하지 못하는 구조) ❌
  <div className="flex flex-col">
    <input id="email" {...register('email')} />
    {errors.email && <p className="text-red-500">{errors.email.message}</p>}
  </div>

  // 좋은 예시 (위치 정합성과 실시간 접근성 표준을 완벽히 동기화한 구조) ⭕
  <div className="flex flex-col gap-1">
    <label htmlFor="email">이메일 주소</label>
    <input id="email" {...register('email')} className={errors.email ? 'border-red-500' : ''} />

    {/* role="alert"를 통해 보조공학 기기에 실시간 경고 피드백을 보장 */}
    {errors.email && (
      <span id="email-error" role="alert" className="text-sm text-red-500 mt-1">
        {errors.email.message}
      </span>
    )}
  </div>
  ```

### 📌 규칙 04. API 에러 발생 시 입력 모달/팝업 상태 유지 (Preserve Modal Context on Error)

- **적용 가이드**: 데이터 저장 및 수정 등을 위해 화면에 열려있는 입력 모달(Modal), 다이얼로그(Dialog), 또는 슬라이드 오버 팝업 내에서 양식을 제출했으나 백엔드 API 통신 실패(Error)가 발생한 경우, 해당 모달을 자동으로 닫는 행위를 전면 금지합니다.
- **사용자 데이터 보호 목적**: 유저가 모달 안의 수많은 입력 필드에 공들여 작성한 폼 데이터(Form State)가 모달이 닫힘으로 인해 통째로 날아가는 파괴적인 경험을 차단하기 위함입니다. 유저가 작성한 맥락을 그대로 유지한 채 오류만 수정하여 재시도할 수 있는 기회를 제공합니다.
- **실장 방식**:
  - 비동기 제출 요청이 실패(`catch` 블록 진입)했을 때는 모달의 열림 상태 변수(예: `isOpen = false`)를 절대 건드리지 않습니다.
  - 모달을 최종적으로 닫는 액션(예: `setIsOpen(false)`)은 **반드시 API 응답이 성공(`try` 블록의 마지막 단계)했을 때만 실행**되도록 로직의 순서를 엄격히 제어합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (에러가 나도 무조건 모달을 닫아 유저의 입력을 증발시킴) ❌
  const handleSave = async () => {
    try {
      await saveApiData(formData);
    } catch (error) {
      showErrorToast("저장 실패");
    }
    setIsOpen(false); // [금지] catch 뒤에서 무조건 닫아버리는 흐름
  };

  // 좋은 예시 (성공 시에만 안전하게 닫고, 에러 시 입력 상태를 보호하는 구조) ⭕
  const handleSave = async () => {
    try {
      setIsSubmitting(true);
      await saveApiData(formData);

      // 1. 성공했을 때만 모달을 닫음
      setIsOpen(false);
      showSuccessToast("성공적으로 보관되었습니다.");
    } catch (error) {
      // 2. 에러 발생 시 모달은 열어둔 채 유저에게 피드백만 전달 (데이터 보존)
      handleError(error);
    } finally {
      setIsSubmitting(false);
    }
  };
  ```

### 📌 규칙 05. 폼 진입 시 이전 입력값 및 에러 상태 필수 초기화 (Form State Initialization)

- **적용 가이드**: 입력 모달(Modal), 팝업, 또는 신규 등록 페이지 등 새로운 입력 양식(Form) 인터페이스에 유저가 진입(Open/Mount)하는 시점에, 과거에 입력했던 값이나 백엔드/발리데이터가 출력했던 이전 에러 메시지 잔재가 화면에 노출되는 것을 전면 금지합니다.
- **UX 목적**: 사용자가 매번 새로운 컨텍스트에서 깨끗한 상태의 입력 양식을 제공받도록 보장하여, 시스템의 오작동 오인을 방지하고 입력의 피로도를 낮추기 위함입니다.
- **실장 방식**:
  - 모달을 열어주는 토글 상태(`isOpen`)가 `false`에서 `true`로 전환되는 이벤트 핸들러 내부, 또는 폼 컴포넌트가 마운트되는 시점(`useEffect` 의존성 배열 제어)에 폼 전역 초기화 함수를 강제 실행합니다.
  - React Hook Form 등의 라이브러리를 사용하는 경우 `reset()` API를 호출하여 입력 필드 값(`values`)과 에러 객체(`errors`)를 동시에 기본 상태로 클리어합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (아까 입력하다가 닫은 에러 메시지와 텍스트가 새 모달에도 그대로 남아있는 구조) ❌
  const openCreateModal = () => {
    setIsOpen(true); // 단순히 열기만 하면 이전 에러 상태가 초기화되지 않고 방치됨
  };

  // 좋은 예시 (모달이 켜지는 순간 이전 흔적을 완벽히 소거하여 쾌적한 UX를 제공하는 구조) ⭕
  const {
    register,
    reset,
    formState: { errors },
  } = useForm();

  const openCreateModal = () => {
    // 1. 모달을 열기 직전 혹은 직후에 이전 입력값과 에러 객체를 통째로 클리어
    reset({ email: "", password: "" });

    // 2. 깨끗해진 상태로 폼 모달 활성화
    setIsOpen(true);
  };
  ```

### 📌 규칙 06. 파괴적/주요 액션 실행 전 최종 확인 다이얼로그 강제 (Destructive Action Confirmation)

- **적용 가이드**: 데이터의 등록(Create), 수정(Update), 특히 시스템에서 영구히 데이터가 소실되거나 비가역적인 변화를 일으키는 **삭제(Delete) 및 갱신(Refresh)** 동작을 수행하는 모든 유저 이벤트 핸들러는, 백엔드 API를 호출하기 전에 반드시 유저에게 의사를 재확인하는 **확인 다이얼로그(Confirmation Dialog 또는 Browser Confirm)**를 최선행하여 노출해야 합니다. 버튼 클릭 시 경고 없이 즉시 API 요청이 전송되는 것을 전면 금지합니다.
- **UX 및 데이터 안전 목적**: 유저의 단순 마우스 오클릭, 터치 미스, 혹은 순간적인 인지 오류로 인해 기존 데이터가 영구 삭제되거나 덮어씌워지는 파괴적인 데이터 유실 사고를 프론트엔드 레벨에서 안전하게 구제하기 위함입니다.
- **실장 방식**:
  - 중요 버튼 클릭 시 즉시 비동기 제출 함수를 실행하지 않고, 컨펌 다이얼로그 상태 변수(예: `showConfirm = true`)를 먼저 활성화하거나 브라우저 표준 `window.confirm()`을 가드로 세웁니다.
  - 사용자가 다이얼로그에서 **[확인 / 승인]** 버튼을 최종 클릭한 것이 검증된 시점에만 실제 백엔드 API 클라이언트 함수(프론트엔드 규칙 02 준수)를 트리거하도록 로직의 진입 조건을 제어합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (삭제 버튼을 누르자마자 경고 없이 데이터를 지워버려 유실 위험이 높은 구조) ❌
  const handleDelete = async (postId) => {
    await deleteZibaldonePost(postId); // 클릭 실수 시 복구 불가능
  };

  // 좋은 예시 (최종 확인 가드를 거쳐 유저의 실수를 완벽하게 안전망으로 방어하는 구조) ⭕
  const handleDelete = async (postId) => {
    // 1. 브라우저 표준 또는 커스텀 UI 다이얼로그를 통해 유저의 최종 의사를 가드로 확인
    const isConfirmed = window.confirm(
      "이 노하우 기록을 영구히 삭제하시겠습니까? 삭제 후에는 복구할 수 없습니다.",
    );

    // 2. 취소 선택 시 즉시 리턴하여 백엔드 API 요청 차단
    if (!isConfirmed) return;

    try {
      setIsDeleting(true); // 이중 송신 방지 락 (프론트엔드 규칙 2-02 준수)
      await deleteZibaldonePost(postId);
      showSuccessToast("안전하게 삭제되었습니다.");
    } catch (error) {
      handleError(error);
    } finally {
      setIsDeleting(false);
    }
  };
  ```

### 📌 규칙 07. 검색 및 필터 상태의 URL 쿼리 파라미터 동기화 (URL Query State Management)

- **적용 가이드**: 리스트 화면이나 대시보드에서 사용하는 검색어, 카테고리 필터, 정렬 조건(Sort), 페이지네이션 번호 등의 검색 조건 상태를 단순 리액트 로컬 상태(`useState`)에만 가두어 관리하는 행위를 전면 금지합니다. 모든 검색·필터 조건은 반드시 브라우저의 **URL 쿼리 파라미터(Query Parameters / Search Params)**와 실시간으로 동기화되어야 합니다.
- **UX 및 기능 목적**:
  - **상태의 영속성(Persistence)**: 유저가 페이지를 새로고침(F5)하거나 브라우저 앞/뒤로 가기를 실행하더라도 이전에 조회하던 검색 필터 상태가 완벽히 유지·복원되도록 보장합니다.
  - **공유 가능성(Shareability)**: 특정 필터가 적용된 현재의 화면 주소(URL)를 그대로 복사하여 다른 사용자에게 공유했을 때, 상대방도 완벽히 동일한 검색 결과 화면을 볼 수 있도록 주소의 정합성을 확보합니다.
- **실장 방식**:
  - Next.js의 내장 라우팅 훅(App Router의 `useRouter`, `usePathname`, `useSearchParams`)을 활용하여 구현합니다.
  - 유저가 필터를 변경하면 로컬 state를 직접 바꾸는 대신, `URLSearchParams` 객체를 가공하여 URL 주소를 밀어 넣는 방식(`router.push` 또는 `router.replace`)으로 이벤트를 처리합니다.
  - 컴포넌트 마운트 및 데이터 페칭 시점(최상위 서버 컴포넌트 포함)에는 URL에 박혀있는 쿼리 파라미터 값을 읽어와 초기 검색 조건으로 바인딩합니다.
- **구조 예시**:

  ```tsx
  // 나쁜 예시 (새로고침하면 필터가 초기화되고 주소창을 복사해도 공유가 안 되는 구조) ❌
  const [filter, setFilter] = useState("all"); // 새로고침 시 'all'로 리셋됨

  // 좋은 예시 (URL 주소가 모든 상태를 대변하여 완벽한 복원과 공유를 지원하는 구조) ⭕
  import { useRouter, usePathname, useSearchParams } from "next/navigation";

  export default function FilterBar() {
    const router = useRouter();
    const pathname = usePathname();
    const searchParams = useSearchParams();

    // 현재 URL에서 필터 값을 읽어옴 (기본값 'all')
    const currentFilter = searchParams.get("category") || "all";

    const handleFilterChange = (newCategory: string) => {
      const params = new URLSearchParams(searchParams.toString());
      params.set("category", newCategory);

      // 상태 변경 대신 URL 주소를 교체 (shallow routing 유도)
      router.replace(`${pathname}?${params.toString()}`);
    };

    return (
      <button onClick={() => handleFilterChange("insights")}>
        인사이트 보기
      </button>
    );
  }
  ```

### 📌 규칙 08. 텍스트 없는 아이콘 버튼의 aria-label 필수 지정 (Web Accessibility for Icon Buttons)

- **적용 가이드**: 글자(Text)가 포함되지 않고 오직 그래픽 아이콘(예: 닫기 `❌`, 검색 `🔍`, 설정 `⚙️` 등)으로만 구성된 모든 버튼(`button`) 및 클릭 가능한 상호작용 요소에는 반드시 해당 버튼의 구체적인 기능을 설명하는 **`aria-label` 속성**을 명시해야 합니다. 텍스트가 없는 버튼을 속성 없이 방치하는 행위를 전면 금지합니다.
- **웹 접근성 목적**: 시각 장애인이나 저시력 사용자가 스크린 리더(Screen Reader) 등의 보조공학 기기를 사용하여 화면을 탐색할 때, 해당 요소가 어떤 동작을 수행하는 버튼인지 명확하게 정량적인 음성 피드백을 제공하여 정보 접근성 표준(WCAG / JIS X 8341)을 준수하기 위함입니다.
- **실장 방식**:
  - 컴포넌트 내부에 아이콘 컴포넌트(예: Lucide React, FontAwesome 등)만 배치할 경우, 부모 `<button>` 태그에 명사형 또는 목적어+동사형 조합의 `aria-label="닫기"`, `aria-label="검색어 입력 제출"` 등의 명확한 설명 스트링을 바인딩합니다.
- **구조 예시**:

  ```tsx
  # 나쁜 예시 (스크린 리더가 단순 "버튼"으로만 읽어 무슨 기능인지 알 수 없는 구조) ❌
  <button onClick={handleClose}>
    <CloseIcon />
  </button>

  # 좋은 예시 (마우스 오버나 시각적 변경 없이 보조공학 기기에 완벽한 정보를 전달하는 구조) ⭕
  <button onClick={handleClose} aria-label="창 닫기">
    <CloseIcon />
  </button>
  ```

---

## 🔒 3. 프론트엔드 보안 및 안전성 규칙 (Frontend Security Rules)

### 📌 규칙 01. dangerouslySetInnerHTML 사용 전면 금지 (XSS 취약점 방지)

- **적용 가이드**: 리액트의 `dangerouslySetInnerHTML` 속성을 사용하여 원시 HTML 코드를 화면에 직접 렌더링하는 소스코드를 작성하는 것을 전면 금지합니다.
- **보안 위험성**: 유저가 입력한 폼 데이터나 외부 API에서 전달받은 텍스트에 악성 자바스크립트 스크립트(`<script>`) 코드가 포함되어 있을 경우, 그대로 브라우저에서 실행되어 세션/토큰 탈취 및 쿠키 해킹으로 이어지는 XSS(Cross-Site Scripting) 취약점을 유발합니다.
- **실장 방식 (대체 패턴)**:
  - 모든 데이터는 리액트의 기본 바인딩 방식(`{변수명}`)을 사용하여 단순 텍스트(String) 형태로 안전하게 화면에 출력(자동 이스케이프 처리)합니다.
  - 블로그 포스트, 마크다운 뷰어 등 어쩔 수 없이 서버로부터 받아온 HTML 태그를 해석하여 화면에 그려야 하는 특수 공정이 발생할 경우에는, 결코 날것 그대로 랜더링하지 말고 반드시 사전에 신뢰성이 검증된 **HTML 새니타이저 라이브러리(예: `DOMPurify` 또는 `isomorphic-dompurify` 등)를 거쳐 악성 스크립트를 완전히 제거(Sanitizing)**한 안전한 스트링만 바인딩하도록 강제합니다.

### 📌 규칙 02. 클라이언트 사이드 환경 변수 노출 제한 및 시크릿 정보 주입 금지 (Secure Environment Variables)

- **적용 가이드**: Next.js 환경에서 브라우저(Client Side)에 공개되는 환경 변수 내에 백엔드 전용 API 키, 데이터베이스 자격 증명, 암호화 시크릿 등 민감한 정보를 설정하는 것을 전면 금지합니다.
- **보안 위험성**: Next.js는 `NEXT_PUBLIC_` 접두사가 붙은 환경 변수를 자바스크립트 빌드 파일에 평문으로 포함시켜 브라우저로 전송합니다. 여기에 민감 정보를 기재할 경우 외부인에게 모든 인프라 권한이 노출되는 치명적인 해킹 사고를 유발합니다.
- **실장 방식**:
  - **브라우저 비공개 (백엔드/서버 전용)**: AWS 자격 증명, DB 주소, 결제 비밀키 등 서버(Next.js 서버 사이드 렌더링/API Routes 또는 Express 백엔드)에서만 참조하는 환경 변수는 절대로 `NEXT_PUBLIC_`을 붙이지 않고 선언합니다.
  - **브라우저 공개 가능 기준**: 오직 프론트엔드 브라우저 앱이 직접 참조해야 하는 비민감성 정보(예: Firebase 클라이언트 App ID, 공개용 웹 분석 트래킹 ID 등)에 한해서만 `NEXT_PUBLIC_` 접두사를 허용합니다.
- **구조 예시**:

  ```env
  # 나쁜 예시 (F12 개발자 도구로 소스코드를 열면 누구나 볼 수 있게 노출됨) ❌
  NEXT_PUBLIC_AWS_SECRET_ACCESS_KEY="super-secret-key"
  NEXT_PUBLIC_DATABASE_URL="postgresql://..."

  # 좋은 예시 (서버 환경 내부에서만 안전하게 참조되는 구조) ⭕
  AWS_SECRET_ACCESS_KEY="super-secret-key"
  DATABASE_URL="postgresql://..."
  NEXT_PUBLIC_GA_TRACKING_ID="G-XXXXXX" # 공개용 트래킹 ID만 브라우저 노출 허용
  ```
