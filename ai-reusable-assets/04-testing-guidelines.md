# 🧪 단계별 테스트 룰 및 가이드라인 (Testing Guidelines)

> 본 문서는 기능 테스트, 화면 테스트, 통합 테스트 등 전 공정에서 AI와 함께 테스트 코드를 작성하고 검증할 때 적용하는 표준 서술 및 실장 규칙입니다.

---

## 📝 1. 테스트 케이스 서술 규칙 (Test Case Description Rules)

### 📌 규칙 01. 체언종결 전면 금지 및 명확한 서술어 사용

- **적용 가이드**: 테스트 케이스의 이름(`it()`, `test()`)이나 명세서 서술 시, 명사형으로 문장을 끝마치는 체언종결을 전면 금지합니다. 문장의 끝은 반드시 주어와 목적어의 동작이나 상태를 명확히 알 수 있는 **서술어(동사형, 형용사형)**로 완결해야 합니다.
- **나쁜 예시 (체언종결 ❌)**:
  - `올바른 비밀번호 입력 시 로그인 성공`
  - `외부 API 타임아웃 발생 시 에러 처리`
- **좋은 예시 (서술어 종결 ⭕)**:
  - `올바른 비밀번호를 입력하면 로그인이 정상적으로 수행되어야 한다.`
  - `외부 API 타임아웃이 발생하면 504 에러 핸들러가 이를 가로채서 정제된 메시지를 반환해야 한다.`
- **실장 방식**: AI가 생성하는 모든 테스트 코드의 케이스명은 조건(Context)과 기대 결과(Expected Outcome)가 하나의 완결된 문장 형태로 드러나도록 작성합니다.

### 📌 규칙 02. 추상적 표현 금지 및 구체적 상태/값 명시 (Concrete State Definition)

- **적용 가이드**: 테스트 케이스명 및 검증 조건(Assertion) 서술 시, 형용사 중심의 모호하고 추측 가능한 추상적인 표현의 사용을 전면 금지합니다. 반드시 입력되는 데이터의 조건과 기대되는 시스템의 구체적인 상태(HTTP 상태 코드, 객체의 형태, 데이터 유무 등)를 명확히 기재해야 합니다.
- **금지어 설정 (추상적 표현 ❌)**:
  - `올바른 데이터`, `적절한 값`, `에러 발생`, `정상 처리 완료`, `실패하는 경우`
- **나쁜 예시 (추상적 서술 ❌)**:
  - `적절한 토큰이 전송되면 정상적으로 데이터가 조회되어야 한다.`
  - `잘못된 패스워드를 넣으면 에러가 발생해야 한다.`
- **좋은 예시 (구체적 상태 명시 ⭕)**:
  - `Authorization 헤더에 유효한 JWT 토큰이 실려 전송되면, HTTP 200 상태 코드와 함께 사용자 프로필 객체가 반환되어야 한다.`
  - `8자리 미만의 패스워드를 입력하면, HTTP 400(Bad Request) 에러와 함께 '비밀번호는 최소 8자리 이상이어야 합니다'라는 메시지가 반환되어야 한다.`
- **실장 방식**: AI가 단언문(`expect()`, `assert()`)을 생성할 때, 명세서에 작성된 구체적인 상태와 실제 코드의 기대치(Expected Value)가 1:1로 매칭되도록 꼼꼼하게 검증 코드를 작성합니다.

---

## 🛠️ 2. 테스트 코드 실장 패턴 (Test Execution Patterns)

### 📌 규칙 01. Given-When-Then 구조화 및 시각적 분리 필수 (AAA Pattern)

- **적용 가이드**: AI가 작성하는 모든 단계별 테스트 코드(기능, 화면, 통합 테스트)는 반드시 **Given(준비) - When(실행) - Then(검증)**의 3단계 구조를 엄격히 준수하여 작성해야 합니다.
- **실장 방식**:
  - 코드 내에 주석으로 `// Given`, `// When`, `// Then` 단계를 명시하여 시각적으로 논리적 흐름이 분리되도록 합니다.
  - **Given (의도 및 조건)**: 테스트 수행에 필요한 가짜 데이터(Mock Data), 데이터베이스(PostgreSQL)의 초기 상태, 환경 변수, 필요한 모듈 등의 사전 조건을 설정하는 단계입니다.
  - **When (실행 행위)**: 실제로 검증하고자 하는 백엔드 API 호출, 프론트엔드 컴포넌트 이벤트 발생, 특정 서비스 레이어의 함수 실행 등 단 한 가지의 핵심 행위를 수행하는 단계입니다.
  - **Then (구체적 검증)**: 실행 결과로 나온 리턴 값, HTTP 상태 코드, UI의 변경 상태가 규칙의 기대치와 완벽히 일치하는지 단언문(`expect()`)을 통해 검증하는 단계입니다.
- **예시 구조**:

  ```javascript
  test("유효한 아이디로 조회하면 사용자 정보 객체를 반환해야 한다", async () => {
    // Given
    const userId = "user_01";
    mockDatabase.find.mockResolvedValue({ id: "user_01", name: "Freelancer" });

    // When
    const result = await userService.getUser(userId);

    // Then
    expect(result).toHaveProperty("name", "Freelancer");
    expect(mockDatabase.find).toHaveBeenCalledWith(userId);
  });
  ```

### 📌 규칙 02. 기능 테스트 내 외부 의존성 전면 차단 및 목업(Mock) 대체 (Environment-Independent Testing)

- **적용 가이드**: 단위 및 기능 테스트(Unit / Functional Test) 환경에서는 실제 데이터베이스(PostgreSQL)에 연결하거나 외부 벤더의 API를 네트워크를 통해 직접 호출하는 행위를 전면 금지합니다. 모든 외부 환경 의존성은 가짜 객체인 **목업(Mock / Spy / Stub)**으로 완벽히 대체되어야 합니다.
- **환경 격리의 목적**: 외부 인프라 상태, 네트워크 병목, DB 데이터 오염 등에 영향을 받지 않고, 로컬 개발 환경(DevContainer)뿐만 아니라 빌드 서버(CI/CD 파이프라인 등)에서도 언제나 빠르고 안정적으로 100% 동일한 성공 결과(Deterministic Results)를 내기 위함입니다.
- **실장 방식**:
  - **DB 액세스 모듈 목업**: 리포지토리 레이어(Repository Layer)나 데이터베이스 클라이언트가 반환하는 값은 `// Given` 단계에서 테스트 목적에 맞는 가짜 데이터 객체(Mock Data)로 강제 지정(`mockResolvedValue`)합니다.
  - **외부 API 통신 모듈 목업**: `axios`, `fetch` 등 네트워크 통신 라이브러리 또는 외부 통신 전담 서비스를 모킹하여, 실제 요청은 날아가지 않고 규격화된 가짜 HTTP 응답만 즉시 리턴되도록 가로챕니다.

---

## 🖥️ 3. 화면 및 UI 테스트 실장 규칙 (UI & Component Testing Rules)

### 📌 규칙 01. 단일 요소 매칭 셀렉터 지정 및 동적 변환 값 지정 금지

- **적용 가이드**: 프론트엔드 컴포넌트 및 화면 테스트 코드를 작성할 때, 단언문(Assertion)이나 이벤트 시뮬레이션을 위해 요소를 찾는 셀렉터(Selector)는 반드시 **확실하게 1개의 고유한 요소만 매핑되도록 고유성**을 보장해야 합니다. 복수의 요소가 잡혀 테스트가 모호해지는 것을 금지합니다.
- **셀렉터 지정 금지 대상 (Flaky 요소 ❌)**:
  - 언제든 문구가 바뀔 수 있는 **프레이스홀더(Placeholder) 텍스트** (예: `input[placeholder="이메일을 입력하세요"]`)
  - 다국어 처리(i18n)로 변환되거나 비즈니스 로직에 의해 **동적으로 변하는 텍스트/입력값** (예: `button:has-text("3개 선택 완료")`)
- **실장 방식 (테스트 전용 속성 활용 ⭕)**:
  - 화면 레이아웃이나 가피(Copy) 문구, placeholder가 어떻게 바뀌더라도 테스트 코드가 깨지지 않도록, 테스트 대상이 되는 핵심 UI 요소에는 **테스트 전용 식별자 속성(예: `data-testid="submit-button"` 또는 `data-cy="..."`)**을 마크업에 미리 부여합니다.
  - AI가 테스트 코드를 생성할 때는 반드시 이 테스트 전용 식별자(data-testid)를 최우선 셀렉터로 지정하여 요소를 찾도록 제어합니다.
- **구조 예시**:

  ```tsx
  // 나쁜 예시 (placeholder 문구가 수정되면 기능이 멀쩡해도 테스트가 깨짐) ❌
  const emailInput = screen.getByPlaceholderText("이메일을 입력하세요");
  const submitBtn = screen.getByText("로그인하기");

  // 좋은 예시 (UI 디자인이나 문구 변경에 무너지지 않는 단단한 테스트 셀렉터) ⭕
  // 컴포넌트 실장: <input data-testid="login-email-input" placeholder="..." />
  // 컴포넌트 실장: <button data-testid="login-submit-button">로그인하기</button>

  const emailInput = screen.getByTestId("login-email-input");
  const submitBtn = screen.getByTestId("login-submit-button");
  ```

### 📌 규칙 02. 리액트 고유의 비트리거(Non-Trigger) 현상 회피 및 완벽한 상태 동기화

- **적용 가이드**: 화면 테스트 코드를 작성하거나 커스텀 입력 컴포넌트를 구현할 때, 브라우저의 요소 값(Value)만 변경되고 리액트의 내부 상태(`useState`)나 이벤트 핸들러(`onChange`)가 유실(Bypass)되는 비트리거 현상을 원천 차단해야 한다.
- **발생 상황**: 테스트 코드에서 `element.value = '값'`과 같이 DOM을 직접 조작하거나, 브라우저 네이티브 이벤트가 리액트의 가상 DOM(Virtual DOM) 이벤트 시스템과 결합하지 못해 입력값은 바뀌었으나 실제 내부 상태는 여전히 빈 값으로 남아있는 경우 발생한다.
- **실장 및 테스트 방식**:
  - **테스트 환경 (Testing Library 등)**: 원시 DOM 이벤트를 임의로 쏘는 행위를 금지하며, 반드시 사용자의 실제 행동을 브라우저 엔진 수준에서 그대로 모방하여 리액트 가상 DOM 이벤트를 완벽히 트리거해 주는 **`userEvent` API(예: `await userEvent.type()`, `await userEvent.click()`)**를 최우선으로 사용하여 상태 동기화를 강제한다.
  - **컴포넌트 구현 환경**: 외부 라이브러리나 제어되지 않는 컴포넌트(Uncontrolled Component)와 리액트 상태를 연동할 때는, 값이 변경될 때 리액트가 인지할 수 있도록 명시적인 이벤트 버블링 유틸리티를 거치거나 Next.js/React 표준 폼 제어 라이브러리(예: React Hook Form의 `Controller` 등)를 경유하여 실장한다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (리액트 내부의 onChange가 트리거되지 않아 상태가 동기화되지 않음) ❌
  const emailInput = screen.getByTestId("login-email-input");
  emailInput.value = "test@example.com"; // 브라우저 값만 바뀌고 useState는 먹통이 됨
  fireEvent.change(emailInput); // 불안정한 수동 트리거

  // 좋은 예시 (리액트 가상 DOM의 내부 이벤트와 useState를 완벽히 트리거하는 구조) ⭕
  import userEvent from "@testing-library/user-event";

  const user = userEvent.setup();
  const emailInput = screen.getByTestId("login-email-input");

  // 실제 유저의 타이핑 흐름을 시뮬레이션하여 비트리거 현상을 완벽히 회피
  await user.type(emailInput, "test@example.com");
  ```

### 📌 규칙 03. 유니크 제약 필드에 대한 동적 데이터 생성 및 반복 테스트 보장

- **적용 가이드**: 데이터베이스(PostgreSQL) 내에서 이메일, 아이디, 고유 번호 등 **유니크 제약(Unique Constraint)**이 걸려 있는 필드를 검증하는 테스트 코드를 작성할 때, 하드코딩된 고정 값(Static Value)을 사용하는 것을 전면 금지합니다. 테스트가 실행될 때마다 매번 고유한 **동적 값(Dynamic Value)**이 생성되도록 구현해야 합니다.
- **반복 테스트 보장의 목적**: 이전 테스트 실행으로 인해 데이터베이스나 목업(Mock) 세션에 잔재 데이터가 남아있더라도, 다음 테스트 실행 시 데이터 중복 충돌 오류(Unique Violation)를 일으키지 않고 수백 번 반복해서 독립적으로 안정적인 테스트(Idempotent Testing)를 수행하기 위함입니다.
- **실장 방식**:
  - 테스트용 가짜 데이터를 준비하는 `// Given` 단계에서, 유니크 필드에 타임스탬프, 랜덤 숫자, 또는 **UUID / 나노아이디(NanoID)** 유틸리티를 결합하여 고유성을 보장합니다.
  - 예시: `test-user@example.com` ❌ ➡️ `test-user-${Date.now()}-${Math.random().toString(36).substring(2, 7)}@example.com` ⭕
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (첫 실행은 성공하지만, 두 번째 실행부터는 이메일 중복으로 테스트가 깨짐) ❌
  test("회원가입 요청 시 유저가 성공적으로 생성되어야 한다", async () => {
    // Given
    const duplicatedSignUpDto = { email: "freelancer@example.com", name: "SE" };
    // ...이후 검증 로직 실행
  });

  // 좋은 예시 (언제 어느 환경에서 몇 번을 반복 실행해도 유니크 충돌이 없는 구조) ⭕
  test("회원가입 요청 시 유저가 성공적으로 생성되어야 한다", async () => {
    // Given
    const uniqueSuffix = `${Date.now()}_${Math.random().toString(36).substring(2, 5)}`;
    const uniqueSignUpDto = {
      email: `freelancer_${uniqueSuffix}@example.com`, // 동적 고유 값 생성
      name: "SE",
    };
    // ...이후 Given-When-Then 흐름에 따라 검증 실행
  });
  ```
