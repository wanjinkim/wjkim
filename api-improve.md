# REST API 구조를 책임별로 분리해 변경 범위를 줄였습니다

레거시 PHP REST API를 정리하면서 인증, 장비별 도메인 처리, 시스템 명령 실행, 예외 처리가 서로 다른 책임이라는 점에 집중했습니다.

## 프로젝트 정보
- 팀/소속: SW2팀
- 직접 설계·구현 범위: 아래 API 리팩토링 사례
- 기술적 의사결정: 직접 수행

---

# 01. 하나의 ApiHandler에 얽혀 있던 인증과 사용자 저장을 분리했습니다

기존 `ApiHandler`는 Digest 인증 헤더 파싱부터 사용자 파일 읽기, 인증 값 계산, HTTP 요청 정보 접근, 응답 생성까지 한 클래스에서 처리하고 있었습니다.

이 구조에서는 사용자 저장 위치를 바꾸거나 인증 흐름을 수정할 때 서로 다른 책임이 함께 변경됩니다. 그래서 먼저 **인증 로직이 사용자 데이터 저장 방식 자체를 알지 않도록 만드는 것**을 우선했습니다.

### 기존 구조

```text
ApiHandler
 ├─ Digest Parsing
 ├─ User File I/O
 ├─ Auth Calculation
 └─ HTTP Response
```

### 선택한 구조

```text
UserRepositoryInterface
        ↓
FileUserRepository

DigestAuthenticator
        ↓
JsonResponse
```

핵심은 저장소와 인증을 분리하는 것입니다.

```php
interface UserRepositoryInterface
{
    public function getSecret(string $username): ?string;
}

final class DigestAuthenticator
{
    public function __construct(
        private UserRepositoryInterface $userRepository
    ) {}

    public function authenticate(
        string $digestHeader,
        string $method,
        string $uri
    ): bool {
        // 필수 필드 검증
        // 사용자 secret 조회
        // Digest response 검증
    }
}
```

기존 Digest 계산 흐름은 유지하면서 입력값 검증을 강화했습니다. 특히 Digest Header에 포함된 URI와 실제 요청 URI를 비교하고, 계산된 응답과 클라이언트 응답을 `hash_equals()`로 비교하도록 했습니다.

```php
if (($digest['uri'] ?? '') !== $uri) {
    return false;
}

return hash_equals(
    $expectedResponse,
    $digest['response'] ?? ''
);
```

### 무엇이 달라졌는가

- 인증 로직이 파일 저장 방식에 직접 의존하지 않게 됨
- 저장소 교체 경계가 분리됨
- 인증 입력 검증과 HTTP 응답 처리를 별도로 관리할 수 있게 됨

---

# 02. 장비별 분기를 Controller에서 분리했습니다

Audio/DSP 영역에서는 장비 종류에 따라 함수와 조건문이 늘어나는 구조가 있었습니다. Gain 계산과 장비별 처리까지 같은 영역에 섞여 있어 모델이 추가될수록 Controller의 변경 범위가 커졌습니다.

저는 **공통 알고리즘과 장비별 차이를 서로 다른 계층으로 분리**했습니다.

```text
AudioDspController
        ↓
DspServiceFactory
        ↓
DspServiceInterface
      ↙       ↘
  Type A     Type B
```

```php
interface DspServiceInterface
{
    public function getInputData(): array;
    public function saveInputData(array $data): bool;
}

final class DspServiceFactory
{
    public function make(string $model): DspServiceInterface
    {
        return match ($model) {
            'type-a' => new TypeADspService(),
            'type-b' => new TypeBDspService(),
            default => throw new InvalidArgumentException(
                '지원하지 않는 장비 모델입니다.'
            ),
        };
    }
}
```

Gain 계산은 `GainCalculator`로 분리하고, 장비별 허용 범위와 처리 방식은 각 `DspService`가 담당하도록 했습니다. Controller는 모델별 구현체를 직접 선택하지 않고 Factory에 위임합니다.

이 구조를 선택한 이유는 장비별 차이가 계속 늘어날 수 있는 영역과 공통 처리 흐름을 분리하기 위해서입니다. 실제 신규 장비 추가 시 수정 범위가 얼마나 줄었는지는 별도 측정이 필요합니다.

### 무엇이 달라졌는가

Controller가 특정 장비 구현에 직접 의존하지 않고, 장비별 차이를 구현체로 격리할 수 있게 됐습니다.

---

# 03. 시스템 명령 실행을 별도의 경계로 만들었습니다

Network/SIP 설정은 입력값을 검증하는 책임과 실제 시스템 명령을 실행하는 책임이 같은 영역에 섞여 있었습니다.

그래서 다음 흐름으로 경계를 나눴습니다.

```text
Request
 ↓
Value Validation
 ↓
Domain Object / Config
 ↓
Controller / Manager
 ↓
CommandRunner / Repository
```

핵심 실행 경계는 다음과 같습니다.

```php
interface CommandRunnerInterface
{
    public function run(string $command): int;
}

final class NetworkInterfaceManager
{
    public function __construct(
        private CommandRunnerInterface $runner
    ) {}

    public function setInterface(
        string $interface,
        NetworkConfig $config
    ): void {
        // interface allow-list
        // IP / netmask validation
        // shell argument escaping
        // command execution
    }
}
```

실제 명령 실행을 `CommandRunnerInterface` 뒤로 분리했기 때문에 테스트 환경에서는 다른 Runner를 주입할 수 있습니다.

SIP도 요청 배열을 바로 파일에 저장하지 않고 `SipAccount` 검증 객체와 `SipAccountRepository`를 거치는 구조로 바꿨습니다.

### 무엇이 달라졌는가

- 입력값 검증과 명령 실행을 분리
- 파일 저장과 API 요청 처리를 분리
- 시스템 명령 실행 대체 지점 확보
- SIP 저장 책임 분리

---

# 04. 예외 처리와 HTTP 응답 생성을 분리했습니다

기존에는 오류가 발생할 때마다 배열을 만들어 반환하는 방식이라 엔드포인트별 응답 형식이 달라질 여지가 있었습니다.

그래서 서비스에서 오류를 발생시키고, HTTP 응답 생성은 전역 처리기가 담당하도록 구조를 바꿨습니다.

```text
Service
  ↓
throw ApiException
  ↓
GlobalExceptionHandler
  ↓
HTTP JSON Response
```

```php
class ApiException extends RuntimeException
{
    public function __construct(
        string $message,
        int $statusCode = 400,
        private readonly array $extraData = []
    ) {
        parent::__construct($message, $statusCode);
    }
}
```

`UnauthorizedException`, `ForbiddenException`, `NotFoundException`처럼 HTTP 의미에 맞는 예외를 분리하고 `GlobalExceptionHandler`에서 JSON 응답을 생성했습니다.

### 무엇이 달라졌는가

서비스 코드마다 `http_response_code()`와 `json_encode()`를 반복하지 않고, 예외를 한 흐름에서 처리할 수 있게 됐습니다.

---

# 이 경험에서 남은 설계 원칙

이번 API 리팩토링에서 중요한 것은 클래스를 많이 만든 것이 아니라 **변경 지점을 책임별로 분리한 것**이었습니다.

> 파일 I/O → 비즈니스 로직 → 시스템 명령 → HTTP 응답

이 경계를 나누면서 인증 방식, 장비 구현, 시스템 명령, 예외 응답이 서로 직접 얽히지 않도록 정리했습니다.
