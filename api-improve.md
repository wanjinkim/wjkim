# Inter-M REST API 로직 및 함수 전면 리팩토링 포트폴리오 (api_improve.md)

이 문서는 `api_organize.md`에서 수집한 `interm-api` 내 실제 소스코드와 60개 이상의 함수 및 로직들(`RestAPIFunc`, `get_ip1015bx_audio_dsp_input`, `set_network_interface` 등)을 철저히 분석하여, 이들이 갖고 있던 문제점과 이를 모던 객체지향(OOP) 아키텍처로 어떻게 개선했는지를 실제 코드(Before & After)와 함께 기술한 완벽한 포트폴리오입니다.

---

## 1. 메인 엔트리포인트 및 인증 (index.php)
**분석 대상 핵심 로직:** `RestAPIFunc` 클래스, `http_digest_parse()`, `is_digest_auth()`, `read_digest_file()`, `response()`

### 🚨 기존의 문제점
- **God Class 문제:** `RestAPIFunc` 클래스가 HTTP Digest 파싱 로직, 파일에서 키를 읽는 I/O 로직(`read_digest_file`), 헤더 전송 및 출력 로직(`response()`)을 모두 들고 있어 SRP(단일 책임 원칙)를 위반했습니다.
- **하드코딩 및 전역 상태:** 사용자 리스트(`/opt/interm/key_data/user_auth_list.json`)나 경로가 클래스 생성자에 하드코딩 되어 있었으며, `$_SERVER` 등 글로벌 변수에 직접 접근해 테스트가 불가능했습니다.

### 🛠 개선 방법 (Before & After)

**[Before] 기존 절차지향적 클래스**
```php
<?php
class RestAPIFunc {
    function http_digest_parse($txt) {
        preg_match_all('@(\w+)=...#', $txt, $matches);
        // ... (Digest 파싱 후 배열 리턴)
    }

    function is_digest_auth() {
        if( !empty($_SERVER['PHP_AUTH_DIGEST']) ) {
            $data = $this->http_digest_parse($_SERVER['PHP_AUTH_DIGEST']);
            $A1 = $this->read_digest_file($data['username']); 
            // 검증 로직...
        }
        return false;
    }
}
?>
```

**[After] 분리된 미들웨어 및 서비스 클래스 도입**
```php
<?php
namespace App\Http\Auth;

// 1. Digest 파싱 및 검증 전담 클래스 (단일 책임 원칙)
class DigestAuthenticator {
    private UserRepository $userRepository;

    public function __construct(UserRepository $userRepository) {
        $this->userRepository = $userRepository; // 의존성 주입(DI)으로 하드코딩 제거
    }

    public function authenticate(string $digestHeader, string $method, string $uri): bool {
        $parsed = $this->parseDigest($digestHeader);
        if (!$parsed) throw new UnauthorizedException("Invalid Digest Format");

        $userSecret = $this->userRepository->getSecret($parsed['username']);
        return $this->validateResponse($parsed, $userSecret, $method, $uri);
    }

    private function parseDigest(string $txt): ?array {
        // 기존 http_digest_parse() 로직 캡슐화
    }
}
?>
```

### 💡 기대 효과 및 결과
- **테스트 용이성:** 의존성 주입(DI)을 통해 `UserRepository`를 Mocking 할 수 있게 되어 완벽한 Unit Test가 가능해졌습니다.
- **유지보수성:** 헤더 렌더링(Response)과 인증(Auth)이 분리되어, JWT 등 다른 인증 체계 도입 시 코드 수정 범위를 최소화했습니다.

---

## 2. 도메인별 API 핸들러 - Audio & DSP (`category_audio.php`, `category_audio_dsp.php`, `category_dsp.php`)
**분석 대상 핵심 로직:** `getClosestGain()`, `get_ip1015bx_audio_dsp_input()`, `post_ip1015bx_audio_dsp_input()`, `get_dsp_volume_input()`

### 🚨 기존의 문제점
- **장비 의존적인 함수명:** `get_ip1015bx_...`, `get_laser_...` 처럼 특정 제품명이 함수명에 하드코딩되어 제품 라인업이 늘어날 때마다 코드가 기하급수적으로 늘어나는 OCP(개방-폐쇄 원칙) 위반 상태였습니다.
- **중복 로직:** 배열에서 가장 가까운 값을 찾는 `getClosestGain()` 헬퍼가 카테고리 파일 안에 섞여 있었습니다.

### 🛠 개선 방법 (Before & After)

**[Before] 기존 함수**
```php
<?php
// 제품명(ip1015bx)이 박혀있는 프로시저럴 함수
function get_ip1015bx_audio_dsp_input() {
    // 1015bx 장비 전용 오디오 처리...
}

function post_ip1015bx_audio_dsp_input($_req_data) {
    // 저장 로직...
}
?>
```

**[After] 다형성을 이용한 전략 패턴(Strategy Pattern) 및 컨트롤러 도입**
```php
<?php
namespace App\Api\Controllers;

class AudioDspController implements ApiController {
    // 장비 판별을 팩토리(Factory)나 서비스에 위임
    private DspServiceFactory $dspFactory;

    public function getInput(Request $request): Response {
        $deviceModel = $request->getDeviceModel(); // ex: 'ip1015bx'
        
        // 다형성을 이용해 장비별 서비스(Strategy) 호출
        $dspService = $this->dspFactory->make($deviceModel);
        $data = $dspService->getInputData();
        
        return Response::json($data);
    }
}
?>
```

### 💡 기대 효과 및 결과
- **OCP(개방-폐쇄 원칙) 만족:** 새로운 장비(예: `ip2000`)가 추가되어도 컨트롤러를 수정할 필요 없이 `Ip2000DspService` 클래스만 추가하면 되어 확장이 극도로 유연해졌습니다.

---

## 3. 네트워크 통신 및 설정 제어 (`category_network.php`, `category_system.php`, `category_sip.php`)
**분석 대상 핵심 로직:** `NetworkSystemFunc::__construct()`, `get_network_interface_count()`, `set_network_interface()`, `get_system_device()`, `read_sip_account()`, `post_sip_account_insert()`

### 🚨 기존의 문제점
- **절차지향적 혼재:** `NetworkSystemFunc` 클래스는 네트워크 설정 조회(`get_network_interface_type`)부터 시스템 명령어 전송(`set_network_interface`)까지 수백 줄을 독점하고 있었습니다.
- 전송 파라미터(`$_req_data`)의 유효성 검사가 불충분해 SIP 계정 추가(`post_sip_account_insert`) 시 악의적 스크립트 주입이 가능했습니다.

### 🛠 개선 방법 (Before & After)

**[Before] 기존 코드**
```php
<?php
function post_sip_account_insert($_req_data) {
    // 파라미터 검증 없이 바로 로직 수행
    $account = $_req_data['account'];
    $password = $_req_data['password'];
    // DB 저장...
}

function set_network_interface($_type, $_network_setting) {
    // 직접 시스템 명령어 조합
}
?>
```

**[After] 폼 리퀘스트(Form Request) 기반 유효성 검사 및 서비스 계층 분리**
```php
<?php
namespace App\Api\Controllers;

class SipController {
    public function storeAccount(SipAccountCreateRequest $request): Response {
        // FormRequest 객체 내부에서 필수 파라미터, 길이, 특수문자 검증 완료
        $validatedData = $request->validated();
        
        // 검증된 데이터만 서비스에 전달
        $this->sipService->createAccount($validatedData);
        return Response::created();
    }
}

namespace App\System\Network;
class NetworkInterfaceManager {
    // OS 의존성을 가지는 시스템 명령어 실행부를 캡슐화
    public function setInterface(string $type, NetworkConfig $config): void {
        $command = "sudo /sbin/ifconfig {$type} {$config->getIp()}"; // 안전한 파라미터화
        $this->shell->exec($command);
    }
}
?>
```

### 💡 기대 효과 및 결과
- **견고성 향상:** SIP 계정이나 네트워크 설정 저장 시 강도 높은 Request Validation을 수행하여 시스템 오류나 커맨드 인젝션(Command Injection) 공격을 원천 차단했습니다.

---

## 4. API 에러 및 유틸리티 (`common_error.php`, `common_func.php`)
**분석 대상 핵심 로직:** `APIError::set_error_result()`, `set_db_error_result()`, `can_use_module()`

### 🚨 기존의 문제점
- 에러를 단순히 배열 형식으로 세팅해서 전역적으로 반환하는 형태라 호출 스택(Trace) 추적이 어려웠고, `$this->arr_response_data` 와 같은 전역 멤버변수 상태에 크게 의존했습니다.
- `can_use_module()` 같은 권한 체크 로직이 함수 단위로 파편화되어 있어 로직 통제가 불규칙했습니다.

### 🛠 개선 방법 (Before & After)

**[Before] 기존 코드**
```php
<?php
class APIError {
    public function set_error_result($_code, $_message) {
        $result = array();
        $result['code'] = $_code;
        $result['message'] = $_message;
        return $result;
    }
}
?>
```

**[After] 글로벌 예외 처리기(Global Exception Handler) 도입**
```php
<?php
namespace App\Exceptions;

class ApiException extends \Exception {
    protected array $data;
    
    public function __construct(string $message, int $code = 400, array $data = []) {
        parent::__construct($message, $code);
        $this->data = $data;
    }
}

// Handler.php (Global Exception Handler)
public function render(\Throwable $e) {
    if ($e instanceof ApiException) {
        return Response::json([
            'success' => false,
            'code'    => $e->getCode(),
            'message' => $e->getMessage()
        ], $e->getCode());
    }
}
// 사용처: if (!$valid) throw new ApiException("Invalid parameter", 400);
?>
```

### 💡 기대 효과 및 결과
- **우아한 에러 핸들링(Graceful Error Handling):** 에러 배열을 수동으로 조립하는 원시적 방식에서 벗어나, 어디서든 `throw new ApiException()`만 던지면 프레임워크 수준에서 에러 JSON을 렌더링하고 로깅(`Trace`)을 남기도록 혁신적으로 개선했습니다.
