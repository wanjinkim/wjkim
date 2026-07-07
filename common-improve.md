# 공통 모듈 리팩토링 및 개선 포트폴리오 (Common Modules Improvement)

본 문서는 프로젝트에 존재하는 `common_` 접두사의 **총 19개 공통 모듈 전체(11개 그룹)** 에 대하여, 각각의 기존 코드 문제점과 이를 어떻게 캡슐화 및 객체지향적으로 리팩토링 했는지, 그리고 그 결과 어떤 실질적인 효과를 얻었는지를 전/후 비교 코드와 함께 정리한 문서입니다.

---

## 1. 설정 파일 (Define)
**대상 모듈:** `common_define.php` -> `AppConfiguration.php`

### 🚨 기존의 문제점
- 상수가 전역 네임스페이스에 선언되어 타 라이브러리와 이름 충돌 위험이 있었습니다.
- 용도(Path, Http, Status) 구별 없이 하드코딩되어 코드 응집도가 낮았습니다.

### 🛠 개선 방법 (Before & After)

**[Before]**
```php
<?php
namespace Common\Def {
    const PATH_WEB_CSS_STYLE    = "/css/style.css";
    const STATUS_SUCCESS        = 200;
}
?>
```

**[After]**
```php
<?php
namespace App\Config;

class AppConfiguration {
    public const PATHS = [
        'CSS_STYLE' => '/css/style.css'
    ];

    public const HTTP_STATUS = [
        'SUCCESS' => 200
    ];
}
?>
```

### 💡 기대 효과 및 결과
- 전역 오염을 막고 `AppConfiguration::PATHS['CSS_STYLE']` 형태로 구조적 호출이 가능해져 자동완성 및 유지보수성이 향상되었습니다.

---

## 2. 템플릿 포함 명세 파일 (ETC ISLS)
**대상 모듈:** `common_define_etc.isls.php`, `common_js_etc.isls.php`, `common_process_etc.isls.php`, `common_script_etc.isls.php` -> `ExtensionLoader.php`

### 🚨 기존의 문제점
- 제품 확장용 코드를 include하기 위해 각 파트마다 아무 기능이 없는 더미 `.isls.php` 파일 4개가 불필요하게 파일 시스템을 어지럽혔습니다.

### 🛠 개선 방법 (Before & After)

**[Before]**
```php
<?php
    // 빈 파일, 외부 스크립트 include 대기용 (각 모듈별로 4개 파일 파편화)
?>
```

**[After]**
```php
<?php
namespace App\Core;

class ExtensionLoader {
    public static function load(string $moduleType): void {
        $path = __DIR__ . "/extensions/{$moduleType}_extension.php";
        if (is_file($path)) require_once $path;
    }
}
?>
```

### 💡 기대 효과 및 결과
- 파편화된 빈 파일들을 삭제하고 단일 로더(Loader) 클래스로 통합시켜 파일 시스템 및 관리 포인트를 줄였습니다.

---

## 3. 백엔드 데이터 처리 (Process)
**대상 모듈:** `common_process.php`, `common_process_controller.php`, `common_process_fault.php` -> `AjaxRequestHandler.php`

### 🚨 기존의 문제점
- 컨트롤러, 폴트(Fault) 처리가 나뉘어 있었으나 코어 모듈 안에서 여전히 거대한 `if-else` 분기(`$_POST['type']`)가 중첩되어 있어 OCP 원칙에 위배되었습니다.

### 🛠 개선 방법 (Before & After)

**[Before]**
```php
<?php
if( $_POST['type'] == "log" ) {
    if($_POST['act'] == "write") { /* ... */ }
} else if( $_POST['type'] == "login" ) {
    /* ... */
}
// 하단에서 controller와 fault를 무조건 include
include_once "common_process_controller.php";
?>
```

**[After]**
```php
<?php
namespace App\Http;

class AjaxRequestHandler {
    private array $routes = [
        'log'        => LogController::class,
        'controller' => ProcessController::class,
        'fault'      => FaultController::class
    ];

    public function handle(string $type, string $action, array $data): void {
        if (!isset($this->routes[$type])) throw new Exception("Route Not Found");
        (new $this->routes[$type]())->execute($action, $data);
    }
}
?>
```

### 💡 기대 효과 및 결과
- 라우터(Router) 패턴 도입을 통해, API가 추가될 때 기존 코드 변경 없이 독립된 컨트롤러만 생성하면 되도록 확장성을 극대화했습니다.

---

## 4. 프론트엔드 공통 함수 스크립트 (JS)
**대상 모듈:** `common_js.php`, `common_js_controller.php`, `common_js_fault.php` -> `AppCore.js`

### 🚨 기존의 문제점
- `CommonFunc` 및 `NCSFunc` 등의 거대 클래스(God Class)에 HTTP 통신(`postArgs`)과 UI가 섞여 있고, 브라우저 스레드를 멈추는 동기화 AJAX 통신(`async: false`)을 사용했습니다.

### 🛠 개선 방법 (Before & After)

**[Before]**
```javascript
class CommonFunc {
    postArgs(_target, _args) {
        var result;
        $.ajax({ type: "POST", url: _target, data: _args, async: false, success: function(data){ result = data; }});
        return result;
    }
}
```

**[After]**
```javascript
const AppCore = {
    Http: {
        async fetchPost(targetUrl, payload) {
            const res = await fetch(targetUrl, {
                method: 'POST', body: new URLSearchParams(payload)
            });
            return await res.json();
        }
    }
};
```

### 💡 기대 효과 및 결과
- 비동기(`fetch`) API로 브라우저 프리징을 해결하고 모듈 분리를 통해 타 프로젝트에서의 재사용성을 높였습니다.

---

## 5. UI 렌더링용 스크립트 (Script)
**대상 모듈:** `common_script.php`, `common_script_controller.php` -> `UIComponentBuilder.js`

### 🚨 기존의 문제점
- 무려 100KB가 넘는 단일 파일 안에서 DOM 조작 로직과 이벤트 바인딩이 얽혀 스파게티 코드가 형성되었습니다. (Controller 모듈 또한 분리만 되어 있을 뿐 내부 결합도가 높음)

### 🛠 개선 방법 (Before & After)

**[Before]**
```javascript
// 수백 줄의 제이쿼리 체이닝과 DOM 바인딩이 섞여있음
$("#div_login_form_submit").click(function() {
    var fileName = $("#file_uploadFile").val();
    // 로직 처리...
});
```

**[After]**
```javascript
class UIComponentBuilder {
    constructor(elementId) {
        this.element = document.getElementById(elementId);
    }
    
    bindClickEvent(handler) {
        if(this.element) this.element.addEventListener('click', handler);
    }
}

// 명확하게 분리된 이벤트 초기화
const submitButton = new UIComponentBuilder('div_login_form_submit');
submitButton.bindClickEvent(AuthController.handleUpload);
```

### 💡 기대 효과 및 결과
- 컴포넌트 기반 아키텍처로 개선하여, DOM 제어와 비즈니스 로직을 완벽히 분리하고 테스팅 가능한 구조로 만들었습니다.

---

## 6. 프론트엔드 인증 파일 처리 (Auth)
**대상 모듈:** `common_auth.php` -> `AuthUIController.js`

### 🚨 기존의 문제점
- 인증 파일 검증 로직과 뷰포트(View) 에러 알림 기능이 하나의 거대한 이벤트 리스너 안에 중첩되어 역할 구분이 안 되었습니다.

### 🛠 개선 방법 (Before & After)

**[Before]**
```javascript
// 파일 확장자 검사, 용량 검사, AJAX 전송 로직이 한 함수에 묶임
$("#div_login_form_submit").click(function() {
    if( !fileName ) { alert("파일을 선택하세요"); return; }
    // ...용량 검사 및 formData.append() 전송
});
```

**[After]**
```javascript
class AuthUIController {
    static validateFile(file) {
        if (!file) throw new Error("파일을 선택하세요");
        if (file.size >= MAX_SIZE) throw new Error("용량 초과");
    }

    static async handleUpload(event) {
        try {
            const file = document.getElementById('file_uploadFile').files[0];
            this.validateFile(file);
            await AppCore.Http.uploadFile('/upload', file);
        } catch (error) {
            alert(error.message);
        }
    }
}
```

### 💡 기대 효과 및 결과
- 검증 로직(`validateFile`)과 실행 로직(`handleUpload`)을 분리하여 코드 가독성과 에러 처리의 일관성을 확보했습니다.

---

## 7. 백엔드 인증 파일 업로드 (Auth Upload)
**대상 모듈:** `common_auth_upload.php` -> `SecureFileUploader.php`

### 🚨 기존의 문제점
- 클라이언트가 보낸 파일명을 아무런 필터링 없이 리눅스 `tar` 명령어 인자로 직접 전달하여(Command Injection) 보안에 매우 취약했습니다.

### 🛠 개선 방법 (Before & After)

**[Before]**
```php
<?php
$filePath = "/tmp/" . $_FILES['file']['name'];
pclose(popen('sudo tar zxf ' . $filePath . ' -C /tmp/.', "r")); // 해킹 위험
?>
```

**[After]**
```php
<?php
namespace App\Security;

class SecureFileUploader {
    public function extractArchive(string $originalFileName): bool {
        // 1. 화이트리스트 검증
        if (!preg_match('/^[a-zA-Z0-9_\-\.]+\.tar\.gz$/', $originalFileName)) {
            throw new Exception("Invalid format");
        }
        
        // 2. 쉘 이스케이핑
        $safePath = escapeshellarg("/tmp/" . $originalFileName);
        exec("sudo tar zxf {$safePath} -C /tmp/.", $output, $returnVar);
        
        return $returnVar === 0;
    }
}
?>
```

### 💡 기대 효과 및 결과
- Command Injection 공격 벡터를 완벽하게 차단하고 업로드 무결성을 확보했습니다.

---

## 8. 로그인 및 암호화 (Login)
**대상 모듈:** `common_login.php` -> `CryptoService.js` / `LoginManager.js`

### 🚨 기존의 문제점
- RSA/AES 암호화 알고리즘 로직이 로그인 UI 스크립트 파일 내에 전역 함수로 묶여 있어 재사용이 불가능했습니다.

### 🛠 개선 방법 (Before & After)

**[Before]**
```javascript
class CryptFunc {
    // 암호화 파싱, 키 생성, UI 제어 모두 묶여있음
    rsaes_oaep_encrypt(_rsaKey, _m) { /* ... */ }
}
```

**[After]**
```javascript
export class CryptoService {
    static encryptRSA(publicKey, plainText) { /* 독립된 암호화 로직 */ }
}

import { CryptoService } from './CryptoService.js';
// 로그인 매니저에서는 오로지 서비스만 호출하여 사용
```

### 💡 기대 효과 및 결과
- 암호화 유틸리티를 결제창 등 다른 모듈에서도 쉽게 임포트(ES6 Modules)하여 재사용할 수 있도록 독립성을 보장했습니다.

---

## 9. 세션 관리 (Session)
**대상 모듈:** `common_session.php` -> `SessionManager.php`

### 🚨 기존의 문제점
- `/tmp/session_list`라는 텍스트 파일을 `fopen()`으로 직접 읽고 파싱하여 세션을 확인해, 동시 접속 시 Race Condition과 I/O 병목이 존재했습니다.

### 🛠 개선 방법 (Before & After)

**[Before]**
```php
<?php
$fd = fopen("/tmp/session_list", "r");
$str_session_list = fread($fd, filesize("/tmp/session_list"));
fclose($fd);
$sessions = json_decode($str_session_list);
?>
```

**[After]**
```php
<?php
namespace App\Auth;

class SessionManager {
    private SessionStorageInterface $storage;

    public function __construct(SessionStorageInterface $storage) {
        $this->storage = $storage;
    }
    
    public function getActiveSessions(): array {
        return $this->storage->fetchAll(); // Redis 또는 DB 구현체로 교체 가능
    }
}
?>
```

### 💡 기대 효과 및 결과
- 파일 잠금(Lock) 문제를 해결하고, 차후 세션 저장소를 Redis나 DB로 단 한 줄의 코드 수정 없이 교체(다형성)할 수 있도록 구조를 개선했습니다.

---

## 10. 데이터베이스 인터페이스 (SQLite Interface)
**대상 모듈:** `common_sqlite_interface.php` -> `DatabaseSocketClient.php`

### 🚨 기존의 문제점
- 에러 처리 기능 없이 소켓 쓰기 실패 시 `die()`를 호출하여 서버 스레드 전체를 죽여버리는 치명적 장애 요인을 품고 있었습니다.

### 🛠 개선 방법 (Before & After)

**[Before]**
```php
<?php
function query_interface($_path, $_query) {
    $socket = @socket_create(AF_INET, SOCK_STREAM, SOL_TCP);
    socket_write($socket, $msg, strlen($msg)) or die("Could not send data");
}
?>
```

**[After]**
```php
<?php
namespace App\Database;

class DatabaseSocketClient {
    public function executeQuery(string $path, string $query): array {
        $socket = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);
        if (socket_write($socket, json_encode([$path, $query])) === false) {
            // die() 대신 예외 던지기
            throw new ConnectionException("Socket write failed");
        }
        return json_decode(socket_read($socket, 4096), true);
    }
}
?>
```

### 💡 기대 효과 및 결과
- 네트워크 오류 시 애플리케이션이 멈추지 않고 예외(Exception)를 잡아 사용자에게 친절한 안내를 제공하도록(Graceful Degradation) 리팩토링했습니다.

---

## 11. 다국어 로그 처리 (Log)
**대상 모듈:** `common_log.php` -> `Logger.php`

### 🚨 기존의 문제점
- 매번 로그를 남길 때마다 정규식(`preg_match_all`)을 돌려 다국어 키를 찾아 치환하므로 성능 낭비가 컸습니다.

### 🛠 개선 방법 (Before & After)

**[Before]**
```php
<?php
preg_match_all("#\{(.*?)\}#", $text, $matches);
foreach( $matches[1] as $match ) {
$text = preg_replace(
    "다국어 패턴",
    constant("Lang\\".$match),
    $text
);
}
?>
```

**[After]**
```php
<?php
namespace App\Logging;

class Logger {
    private Translator $translator;
    public function __construct(Translator $translator) {
        $this->translator = $translator;
    }

    public function log(string $level, string $key): void {
        // 정규식 파싱을 걷어내고 Map(배열) 매핑으로 즉각 번역
        $translated = $this->translator->get($key);
        error_log("[{$level}] {$translated}");
    }
}
?>
```

### 💡 기대 효과 및 결과
- 무거운 정규표현식 연산을 O(1) 해시 테이블 검색으로 변경해 로깅 성능을 최적화하고 번역 모듈을 의존성 주입(DI) 방식으로 깔끔히 분리했습니다.
