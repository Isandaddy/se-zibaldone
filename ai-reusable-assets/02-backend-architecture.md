# 🏗️ 백엔드(Express) 아키텍처 및 실장 룰

> Express와 PostgreSQL 환경에서 AI가 백엔드 소스코드를 작성할 때 준수해야 하는 공통 실장 규칙입니다.

## 🛡️ 1. 데이터 검증 및 보안 (Validation & Security)

### 📌 규칙 01. 외부 입력 데이터 검증(Validation) 필수 적용

- **적용 가이드**: 컨트롤러(Controller) 또는 라우터(Router) 레벨에서 외부 입력값(Req.body, Req.query, Req.params)에 대한 유효성 검증을 반드시 완료해야 합니다.
- **실장 방식**: 검증을 통과한 안전한 데이터(Sanitized Data)만 비즈니스 로직을 담당하는 **서비스 레이어(Service Layer)**로 전달합니다.

### 📌 규칙 02. 에러 응답 시 내부 정보 반환 금지 (Secure Error Handling)

- **적용 가이드**: 클라이언트에게 전송되는 HTTP 에러 응답 내용에 시스템 내부 구조를 유추할 수 있는 민감한 정보가 절대 포함되지 않도록 제어해야 합니다.
- **보안 금지 사항**: 에러 응답 객체에 아래 정보의 포함을 전면 금지합니다.
  - 스택 트레이스 (Stack Trace)
  - 데이터베이스(PostgreSQL) 테이블명 및 스키마 정보
  - 서버 내부 파일 경로 및 소스코드 라인 번호
- **실장 방식**: Express의 글로벌 에러 핸들러(Error Handling Middleware)를 구현하여 시스템 내부 에러 로그는 서버(DevContainer/AWS 세션 등) 내부 로그에만 남기고, 클라이언트에게는 정제된 에러 메시지(예: `Internal Server Error` 또는 사전에 정의된 에러 코드)만을 반환합니다.
- **실장 방식**:
  - **[catch 블록 필수 구현]**: 비동기 API 통신, 외부 API 호출, 또는 데이터베이스 액세스가 발생하는 모든 백엔드 비동기 함수 로직에는 반드시 `try-catch` 블록을 명시적으로 구현해야 합니다. `catch` 없는 비동기 함수 작성을 전면 금지합니다.
  - Express의 글로벌 에러 핸들러 미들웨어를 경유하도록 `catch(error) { next(error); }` 구조를 철저히 유지하며, 시스템 내부 에러 로그는 서버 내부에만 기록하고 클라이언트에게는 정제된 에러 메시지만을 반환합니다.

---

## 🏗️ 2. 애플리케이션 계층 구조 및 의존방향 (Architecture & Dependency)

### 📌 규칙 01. 레이어 간 단방향 의존성 엄격 준수 (Strict Layered Architecture)

- **적용 가이드**: Express 백엔드 애플리케이션의 모든 로직은 각 계층(Layer)의 고유 책임을 명확히 분리하고, 상위 계층에서 하위 계층으로의 **단방향 의존성**만을 허용합니다. 계층을 건너뛰거나 하위에서 상위로 역행하는 흐름은 전면 금지합니다.
- **표준 의존 방향**:
  - **`Route` ➡️ `Controller` ➡️ `Service` ➡️ `Repository (Prisma/PostgreSQL)`**
- **계층별 역할 및 제한 사항**:
  - **Route (엔드포인트 매핑)**: HTTP 메서드와 URL 경로를 정의하고, 요청을 올바른 컨트롤러로 매핑하는 역할만 수행합니다. (비즈니스 로직 작성 금지)
    - **Controller (요청/응답 제어 및 HTTP 매핑)**:
    - **[핵심 책임]**: 클라이언트의 HTTP 요청(Request)을 받아 URL/메서드를 매핑하고, 외부 입력을 검증한 뒤, 알맞은 서비스 레이어를 호출하여 결과물을 HTTP 응답(Response) 객체로 반환하는 책임을 전담합니다.
    - **[엄격 금지 사항 - DB 접근 전면 금지]**: 컨트롤러 내부에서 데이터베이스 액세스 라이브러리(ORM, Query Builder 등)를 직접 참조하거나 DB에 쿼리를 날리는 행위를 전면 금지합니다. 데이터 관련 모든 작업은 반드시 서비스 레이어를 거쳐 리포지토리 레이어를 통해서만 수행되어야 합니다.
    - **[테스트 용이성을 위한 매핑 규칙]**: **하나의 컨트롤러 함수가 두 개 이상의 서로 다른 HTTP 라우트(엔드포인트)에 다중 매핑되는 것을 전면 금지합니다.** 반드시 하나의 컨트롤러 함수는 오직 하나의 고유한 엔드포인트 요청만을 처리하는 1:1 대응 관계(Single Endpoint Mapping)를 엄격히 준수해야 합니다. 이를 통해 각 API 단위의 독립적인 테스트 격리성과 용이성을 완벽히 보장합니다.

    - **Service (비즈니스 도메인 로직)**:
      - **[핵심 책임]**: 특정 환경이나 프로토콜에 종속되지 않는 핵심 도메인의 비즈니스 연산, 정책 흐름 제어 및 비즈니스 예외 처리를 담당합니다. 데이터가 필요한 경우 리포지토리 레이어만을 호출합니다.
      - **[엄격 금지 사항 - HTTP 의존성 전면 배제]**:
        - 서비스 내부에서 HTTP 요청/응답 객체(`req`, `res`)를 직접 참조하거나 조작하는 행위를 전면 금지합니다.
        - **비즈니스 예외 발생 시, 서비스 레이어 내부에서 직접 HTTP 상태 코드(예: 400, 404, 500 등)를 결정하여 반환하는 행위를 금지합니다.**
      - **[실장 방식]**: 서비스 레이어는 오직 도메인 고유의 에러 타입(예: `UserNotFoundError`, `InvalidPasswordError` 등 커스텀 에러 객체)만을 정의하고 상위로 던집니다(`throw`). 이 에러 타입을 기반으로 최종 HTTP 상태 코드를 결정하는 것은 컨트롤러(Controller) 또는 전역 에러 핸들러 미들웨어의 책임으로 일임합니다.

  - **Repository (데이터 액세스)**:
    - **[핵심 책임]**: 프로젝트에서 사용하는 데이터베이스 도구(예: ORM, Query Builder, Native SQL 등)를 사용하여 PostgreSQL 데이터베이스에 직접 접근하고 데이터를 생성/조회/수정/삭제(CRUD)하는 데이터 영속성 처리 역할만 전담합니다.

- ## **실장 방식**: AI가 새로운 기능을 구현할 때, 서비스 레이어에서 다른 컨트롤러 함수를 참조하거나 컨트롤러가 리포지토리를 직접 호출하여 책임을 혼재시키는 코드를 생성하지 않도록 계층 간 경계를 철저히 통제합니다.

### 📌 규칙 01. 외부 입력 데이터 검증(Validation) 필수 적용 및 실패 처리

- **적용 가이드**: 컨트롤러(Controller) 또는 라우터(Router) 레벨에서 외부 입력값(Req.body, Req.query, Req.params)에 대한 유효성 검증을 반드시 완료해야 합니다. 검증을 통과한 안전한 데이터(Sanitized Data)만 비즈니스 로직을 담당하는 서비스 레이어(Service Layer)로 전달합니다.
- **검증 실패 처리 규칙 (엄격 준수)**:
  - **발리데이터 함수 내부에서 검증 결과를 단순히 성공/실패 여부인 불리언(`boolean`, true/false) 값으로 반환하는 행위를 전면 금지합니다.**
- **실장 방식**:
  - 유효성 검증 실패 시, 발리데이터는 즉시 무엇이 어떻게 잘못되었는지 구체적인 정보(실패한 필드명, 실패 이유 등)를 담은 **프로젝트 전용 발리데이션 에러 타입(예: `ValidationError` 등 커스텀 에러 객체)을 생성하여 `throw`** 해야 합니다.
  - 라이브러리(Zod, Joi 등)를 사용하는 경우에도 라이브러리가 던지는 내장 에러를 그대로 방치하지 않고, 프로젝트의 표준 에러 포맷으로 가공하거나 전용 에러 핸들러 미들웨어에서 명확히 캐치할 수 있는 형태로 규격화하여 던집니다.

---

## ⚡ 3. 성능 최적화 및 반복문 실장 규칙 (Performance & Loop Control)

### 📌 규칙 01. 루프(Loop) 처리 내 DB 접근 및 외부 API 호출 전면 금지 (N+1 문제 방지)

- **적용 가이드**: 반복문(예: `for`, `while`, `forEach`, `map` 등) 내부에서 데이터베이스 쿼리를 실행하거나 외부 벤더의 API를 호출하는 코드를 작성하는 것을 전면 금지합니다.
- **발생 가능한 위험**: 루프 횟수(N)만큼 네트워크 요청이 반복적으로 발생하여 대규모 레이턴시(Latency)가 유발되고, DB 커넥션 풀 고갈 및 인프라 비용 폭증을 초래합니다.
- **실장 방식 (대체 패턴)**:
  - **DB 접근 시**: 루프를 돌며 개별 조회/수정하지 않고, 루프 전단계에서 필요한 식별자(ID) 배열을 먼저 취합합니다. 이후 데이터베이스의 **`IN` 조건절 또는 벌크 연산(Bulk Operation)**을 사용하여 단 한 번의 쿼리로 데이터를 일괄 처리(Batching)합니다.
  - **외부 API 호출 시**: 대량의 데이터 처리가 필요할 경우, 대상 벤더 API가 '벌크/배치 엔드포인트'를 지원하는지 확인하여 일괄 요청합니다. 만약 개별 호출이 불가피하다면 루프 내 동기 처리를 금지하고, `Promise.all` 등을 활용하여 동시성(Concurrency)을 제어하거나 큐(Queue) 시스템을 활용하는 구조로 설계합니다.

### 📌 규칙 02. 코드 중첩 깊이 제한 및 복잡도 제어 (Max Depth 2 & Early Return)

- **적용 가이드**: 함수 내 제어문(`if-else`, `for`, `while`, `switch` 등)의 중첩 깊이(Nesting Depth)는 **최대 2단계까지만 허용**합니다. 3단계 이상의 깊은 중첩 코드를 작성하는 것을 전면 금지합니다.
- **복잡도 하향을 위한 실장 방식 (2대 해결 패턴)**:
  1.  **조기 반환 (Early Return 패턴)**: 예외 조건이나 실패 케이스를 함수의 최상단에서 먼저 검증하여 즉시 반환(`return` 또는 `throw`)시킴으로써, 함수 본문의 불필요한 `else` 블록을 제거하고 인덴트(들여쓰기) 깊이를 줄입니다.
  2.  **레이어 이식 및 함수 분할**: 루프 내에서 복잡한 조건 분기나 비즈니스 연산이 유기적으로 얽혀 2단계를 초과할 경우, 해당 로직을 독립적인 프라이빗 함수로 쪼개거나 비즈니스 논리를 담당하는 **서비스 레이어(Service Layer)**의 별도 메서드로 이관하여 캡슐화합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (Depth 3) ❌
  function process(user) {
    if (user.isActive) {
      if (user.hasPermission) {
        for (const item of user.items) { ... } // Depth 3 발생
      }
    }
  }

  // 좋은 예시 (Max Depth 1 ~ 2) ⭕
  function process(user) {
    if (!user.isActive || !user.hasPermission) return; // 조기 반환으로 가드

    // 코드가 가로로 확장되지 않고 직관적으로 유지됨
    for (const item of user.items) { ... }
  }
  ```

### 📌 규칙 03. 복잡한 조건식 및 NOT(`!`) 연산자의 설명 변수 분해 (Explaining Variables & NOT Operator)

- **적용 가이드**: `if` 문이나 삼항 연산자 내에 여러 개의 논리 연산자가 복합적으로 얽혀 있는 조건식, 혹은 단일 부정 연산자(`!`)를 직접 노출하여 가독성을 떨어뜨리는 행위를 금지합니다. 조건의 의도를 단 1초 만에 파악할 수 있도록 의미가 명확한 **설명 변수(Explaining Variable)**로 분해하거나 명확한 인라인 주석을 기입해야 합니다.
- **NOT(`!`) 연산자 특수 규칙 [최신 추가]**:
  - `if (!user.isActive)` 처럼 부정형 연산자가 쓰일 경우, 인간의 뇌는 인지적으로 한 번 더 꼬아서 해석해야 하므로 버그의 온상이 되기 쉽습니다.
  - 따라서 부정형 연산자가 쓰인 조건식은 반드시 그 '부정의 상태가 가지는 비즈니스적 의미'를 긍정형 변수명으로 전환하여 할당하거나, 옆에 명확한 주석을 명시해야 합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (느낌표를 놓치거나 복잡한 부정 논리를 뇌로 풀어야 하는 구조) ❌
  if (!token || !user.isVerified) { ... }

  // 좋은 예시 (설명 변수로 치환하여 부정의 의도를 100% 명확히 드러낸 구조) ⭕
  const isTokenMissing = !token;
  const isUserUnverified = !user.isVerified;
  const isAccessDenied = isTokenMissing || isUserUnverified;

  if (isAccessDenied) {
    // 인증 토큰이 누락되었거나 검증되지 않은 유저인 경우 진입
    return res.status(401).json({ message: ERROR_MESSAGES.UNAUTHORIZED });
  }
  ```

### 📌 규칙 04. 추상적/모호한 명명 금지 및 명확한 의미의 단어 선택 (Explicit Naming)

- **적용 가이드**: 변수명, 상수명, 함수명 정의 시 내부 데이터의 본질이나 함수의 구체적인 행위를 유추할 수 없는 추상적이고 모호한 단어의 사용을 전면 금지합니다. 단어 하나를 쓰더라도 비즈니스 도메인과 기술적 의도가 명확히 전달되는 단어를 선택해야 합니다.
- **금지하는 모호한 단어 예시 ❌**:
  - `data`, `info`, `value`, `temp`, `obj` (데이터의 본질을 알 수 없음)
  - `process()`, `handle()`, `doSomething()`, `check()` (무슨 처리를 하는지 알 수 없음)
- **실장 방식 및 매칭 패턴 ⭕**:
  - **변수/상수**: 담고 있는 컨텐츠의 명확한 명사형 단어를 조합합니다. (예: `data` ❌ ➡️ `verifiedUserList` ⭕, `info` ❌ ➡️ `monthlyOrderSummary` ⭕)
  - **함수**: '동사+목적어' 구조를 취하되, 단순 `get` 대신 구체적인 행위를 명시합니다. (예: `checkUser()` ❌ ➡️ `isEmailDuplicated()` ⭕, `processOrder()` ❌ ➡️ `calculateTotalTax()` ⭕)
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (무슨 데이터와 처리가 일어나는지 모호함) ❌
  const info = getInfo(user);
  function getInfo(u) {
    const temp = u.orders.filter((o) => o.valid);
    return temp;
  }

  // 좋은 예시 (코드 자체가 명세서가 되는 명확한 네이밍) ⭕
  const activeOrderHistory = fetchActiveOrdersByUser(user);
  function fetchActiveOrdersByUser(user) {
    const validOrders = user.orders.filter((order) => order.isValid);
    return validOrders;
  }
  ```

### 📌 규칙 05. 매직 넘버 및 매직 스트링의 상수화 관리 (No Magic Numbers/Strings)

- **적용 가이드**: 소스코드 내부(특히 컨트롤러의 응답 처리, 서비스의 예외 처리, 발리데이터의 조건식 등)에 하드코딩된 숫자(Magic Number)나 문자열(Magic String)을 직접 노출하는 것을 전면 금지합니다. 모든 고정된 값은 의미를 담은 상수로 정의하여 관리해야 합니다.
- **필수 상수화 대상 시스템 자산**:
  - HTTP 상태 코드 (예: `200`, `400`, `404`, `500`)
  - 클라이언트 반환용 표준 에러 메시지 문자열
  - 시스템 내 고정된 비즈니스 규칙용 숫자 (예: 페이지네이션 기본 Limit 수, 최대 글자 수 등)
- **실장 방식**:
  - 프로젝트 루트 또는 각 레이어의 지정된 위치에 독립된 상수 파일(예: `constants/httpStatus.ts`, `constants/errorMessages.ts`)을 개설합니다.
  - 값의 성격에 따라 `const` 객체나 `enum` 구조로 캡슐화하여 선언하고, 실제 소스코드에서는 이 상수를 반드시 임포트(Import)하여 참조하도록 구현합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (의도를 알 수 없는 하드코딩된 값들) ❌
  if (user.role === 3) {
    return res.status(403).json({ message: "접근 권한이 없습니다." });
  }

  // 좋은 예시 (상수 파일을 참조하여 안전하고 명확한 구조) ⭕
  import { USER_ROLES } from "../constants/roles";
  import { HTTP_STATUS } from "../constants/httpStatus";
  import { ERROR_MESSAGES } from "../constants/errorMessages";

  if (user.role === USER_ROLES.ADMIN_AUDITOR) {
    return res.status(403).json({ message: ERROR_MESSAGES.ACCESS_DENIED });
  }
  ```

### 📌 규칙 06. 모든 공개(Export) 함수에 대한 JSDoc/TSDoc 작성 의되화

- **적용 가이드**: 외부 모듈이나 레이어에서 임포트(Import)하여 사용할 수 있도록 공유하는 모든 공개 함수, 인터페이스, 메서드에는 반드시 규격화된 **JSDoc 또는 TSDoc 형식의 주석**을 필수로 작성해야 합니다.
- **필수 포함 정보**:
  - 함수의 핵심 역할 및 비즈니스 목적 요약 설명
  - `@param`: 입력 매개변수(Parameter)의 명칭과 의미 설명
  - `@returns`: 함수가 최종적으로 반환하는 결과 데이터의 타입 및 구조 설명
  - `@throws`: 함수 내부에서 발생하여 상위로 전파될 수 있는 주요 에러 타입 명시
- **실장 방식**: AI가 새로운 유틸리티나 서비스 메서드를 생성하고 외부로 내보낼 때, 주석 없이 코드만 노출하는 행위를 전면 금지하며, 편집기(VS Code 등)에서 마우스 오버 시 타입 인텔리센스 힌트가 정상적으로 팝업되도록 규격 주석을 선행 작성합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (함수 목적과 리턴 타입을 파악하려면 코드를 읽어야 함) ❌
  export function calcTax(amount, type) {
    if (type === "JP_CONSUMPTION") return amount * 0.1;
    return 0;
  }

  // 좋은 예시 (IDE 힌트 및 자동 연동이 활성화되는 구조) ⭕
  /**
   * 지정된 금액에 대해 일본 소비세 등 비즈니스 세액을 계산합니다.
   * @param {number} totalAmount - 세전 주문 총 금액
   * @param {string} taxType - 계산할 세금의 종류 상수 (예: 'JP_CONSUMPTION')
   * @returns {number} 최종 산출된 세액 결과 값
   * @throws {InvalidTaxTypeError} 지원하지 않는 세금 종류가 입력될 경우 발생
   */
  export function calculateApplicableTax(
    totalAmount: number,
    taxType: string,
  ): number {
    if (taxType !== TAX_TYPES.JP_CONSUMPTION) {
      throw new InvalidTaxTypeError();
    }
    return totalAmount * TAX_RATES.JP_CONSUMPTION;
  }
  ```

### 📌 규칙 07. 미확정 요건에 대한 규격화된 TODO 마킹 및 방치 금지 (Strict TODO Management)

- **적용 가이드**: 개발 진행 중 즉시 확정이 불가능하여 추후 담당자(인프라 프로퍼, 의뢰자 등)에게 확인이 필요한 내용이나 미구현 로직이 발생할 경우, 임의의 주석을 남기거나 코드를 방치하는 것을 금지합니다. 반드시 지정된 **표준 `TODO` 주석 포맷**을 사용하여 코드 내에 마킹해야 합니다.
- **표준 TODO 주석 포맷**:
  - `// TODO(확인 필요 사항): [구체적인 질문 및 확인이 필요한 맥락] - [확인 주체(예: 인프라 프로퍼/의뢰자)]`
- **실장 및 제어 방식**:
  - AI는 도메인 업무적 판단이 모호하여 구현을 유예하는 세부 로직 블록에 반드시 위 포맷의 주석을 삽입하고, 상위 레이어에는 임시 예외 처리나 안전한 기본값 반환 코드를 배치하여 빌드가 깨지지 않도록 조치합니다.
  - 코드 리뷰 및 최종 머지(Merge) 전, 소스코드 내에 남아있는 `TODO` 리스트를 개발자가 터미널 검색 등을 통해 전수 취합하여 담당자와 최종 인식을 맞추는 가이드라인으로 활용합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (누가 무엇을 확인해야 하는지 알 수 없음) ❌
  // 나중에 확인 필요
  const shippingFee = 0;

  // 좋은 예시 (확인 맥락과 주체가 명확하여 추적이 가능한 구조) ⭕
  // TODO(확인 필요 사항): 일본 낙도/오키나와 지역 배송 시 추가 배송비 연산 로직 확정 필요 - 담당 의뢰자 확인 건
  const shippingFee = calculateDefaultShippingFee(order);
  ```

---

## 💾 4. 데이터베이스 트랜잭션 관리 규칙 (Transaction Management)

### 📌 규칙 01. 복수 테이블 쓰기 작업 시 단일 트랜잭션 보장 (Atomic Multi-Table Writes)

- **적용 가이드**: 비즈니스 로직(Service Layer) 수행 중 둘 이상의 서로 다른 데이터베이스 테이블에 생성(Create), 수정(Update), 삭제(Delete) 등 쓰기 작업을 연속적으로 수행할 경우, 각 작업을 개별적으로 실행하는 것을 전면 금지합니다. 복수의 쓰기 작업은 반드시 하나의 **단일 트랜잭션(Single Transaction) 내** 묶여 원자성(Atomicity)을 보장해야 합니다.
- **실장 방식 및 경계 제어**:
  - **원자성 보장 (All-or-Nothing)**: 복수의 쓰기 연산 중 단 하나라도 실패하거나 예외가 발생할 경우, 트랜잭션 내에서 실행된 모든 데이터 변경 사항이 자동으로 롤백(Rollback)되어 데이터 오염을 예방하도록 설계합니다.
  - **트랜잭션 실행 위치**: 프로젝트의 데이터베이스 처리 표준(예: 데이터베이스 도구의 전용 트랜잭션 API, 미들웨어, 혹은 Native SQL `BEGIN / COMMIT / ROLLBACK`)에 맞춰, 데이터 영속성 작업을 수행하는 적절한 계층(Service 또는 Repository)에서 정확하게 트랜잭션의 시작과 끝(Boundary)을 지정하여 실장합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (주문은 성공했으나 결제 실패 시 주문 데이터가 고립됨) ❌
  await orderRepository.create(orderData);
  await paymentRepository.create(paymentData); // 여기서 에러 나면 데이터 불일치 발생

  // 좋은 예시 (단일 트랜잭션으로 묶어 완벽한 원자성 보장) ⭕
  await databaseClient.\$transaction(async (tx) => {
    await orderRepository.createWithTx(orderData, tx);
    await paymentRepository.createWithTx(paymentData, tx);
  });
  ```

### 📌 규칙 02. 클라이언트 요청 값의 맹신 금지 및 내부 서비스 검증 값 사용 (Data Reliability)

- **적용 가이드**: 비즈니스 로직 연산이나 데이터베이스 변경(쓰기/수정) 작업을 수행할 때, 클라이언트가 HTTP 리퀘스트 바디(`req.body`)로 보내온 원시 데이터를 검증 없이 그대로 가져다 쓰는 행위를 전면 금지합니다. 핵심 연산에 사용되는 기준 데이터는 반드시 백엔드가 데이터베이스에서 직접 조회하여 반환한 **서비스 레이어의 검증된 결과값(Verified Value)**을 사용해야 합니다.
- **보안 및 신뢰성 목적**: 클라이언트 사이드(프론트엔드 또는 외부 요청)에서 조작되거나 위조될 가능성이 있는 데이터(가격, 권한 등)의 유입을 원천 차단하여 시스템의 데이터 무결성과 신뢰성을 확보하기 위함입니다.
- **실장 방식**:
  - 리퀘스트 바디로는 최소한의 식별용 데이터(예: 상품 ID, 유저 ID 등)만 전달받습니다.
  - 해당 식별자를 기반으로 서비스 레이어 내에서 데이터베이스를 조회하고, 최종 검증 및 비즈니스 연산(예: 총 결제 금액 계산, 할인율 적용 등)은 DB에서 갓 꺼내온 신뢰할 수 있는 데이터 객체를 참조하여 수행합니다.
- **구조 예시**:

  ```typescript
  // 나쁜 예시 (클라이언트가 보낸 가격 데이터를 그대로 믿고 처리 - 조작 위험) ❌
  const { productId, price } = req.body;
  await paymentService.process(productId, price);

  // 좋은 예시 (바디의 ID만 참조하고, 실제 가격은 DB 조회 결과로 엄격히 검증) ⭕
  const { productId } = req.body;
  const verifiedProduct = await productService.getProductById(productId); // DB 조회 및 반환
  await paymentService.process(productId, verifiedProduct.price); // 검증된 서비스 반환값 사용
  ```
