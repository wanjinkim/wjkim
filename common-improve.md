# 공통 모듈 리팩토링 포트폴리오

기존 프로젝트에서 `common_` 접두사로 관리되던 공통 모듈을 기능별로 나누어 살펴보고, 중복된 책임과 강한 결합을 줄이는 방향으로 구조를 정리했습니다.

기존 코드는 한 파일 안에서 여러 역할을 처리하거나 전역 함수와 전역 변수에 의존하는 경우가 많았습니다. 리팩토링 과정에서는 각 기능의 책임을 분리하고, 필요한 부분에는 클래스와 인터페이스를 적용해 다른 기능에서도 재사용할 수 있는 형태로 변경했습니다.

---

## 1. 설정 값 관리 (Define)

**대상 모듈:** `common_define.php` → `AppConfiguration.php`

### 기존 문제점

기존에는 여러 종류의 상수를 하나의 네임스페이스에 계속 추가하는 방식으로 관리하고 있었습니다.

경로, HTTP 상태 코드처럼 성격이 다른 값들이 한곳에 나열되어 있어 어떤 값이 어디에서 사용되는지 한눈에 파악하기 어려웠고, 프로젝트 규모가 커질수록 상수 이름이 겹칠 가능성도 있었습니다.

### Before

```php
<?php

namespace Common\Def;

const PATH_WEB_CSS_STYLE = '/css/style.css';
const STATUS_SUCCESS = 200;
const STATUS_NOT_FOUND = 404;
const STATUS_ERROR = 500;
```

### After

```php
<?php

namespace App\Config;

final class AppConfiguration
{
    public const PATHS = [
        'CSS_STYLE' => '/css/style.css',
        'JS_MAIN' => '/js/main.js',
        'UPLOAD_DIR' => '/uploads',
    ];

    public const HTTP_STATUS = [
        'SUCCESS' => 200,
        'NOT_FOUND' => 404,
        'ERROR' => 500,
    ];
}

// 사용 예시
echo AppConfiguration::PATHS['CSS_STYLE'];
echo AppConfiguration::HTTP_STATUS['SUCCESS'];
```

### 변경한 부분

단순히 상수를 클래스 안으로 옮기는 것보다 용도별로 그룹을 나누는 데 중점을 두었습니다.

이제 경로 관련 설정은 `PATHS`, HTTP 상태 코드는 `HTTP_STATUS`를 통해 접근하기 때문에 사용하는 쪽에서도 값의 성격을 바로 알 수 있습니다.

---

## 2. 백엔드 요청 처리 (Process)

**대상 모듈:** `common_process.php`, `common_process_controller.php`, `common_process_fault.php` → `AjaxRequestHandler.php`

### 기존 문제점

요청 타입과 액션을 하나의 PHP 파일에서 직접 비교하면서 처리하고 있었습니다.

요청 종류가 늘어날수록 `if / elseif`가 계속 추가되는 구조였고, 로그 처리와 로그인 처리처럼 성격이 다른 기능이 하나의 흐름 안에 섞여 있었습니다.

### Before

```php
<?php

$type = $_POST['type'] ?? '';
$action = $_POST['act'] ?? 'write';

if ($type === 'log') {
    if ($action === 'write') {
        $message = $_POST['message'] ?? '';

        file_put_contents(
            '/tmp/app.log',
            $message . PHP_EOL,
            FILE_APPEND | LOCK_EX
        );
    }
} elseif ($type === 'login') {
    if ($action === 'write') {
        $user = $_POST['user'] ?? '';

        if ($user !== '') {
            session_start();
            $_SESSION['user'] = $user;
        }
    }
} elseif ($type === 'fault') {
    error_log($_POST['message'] ?? 'Unknown fault');
}
```

### After

```php
<?php

namespace App\Http;

interface RequestControllerInterface
{
    public function handle(string $action, array $data): void;
}

final class LogController implements RequestControllerInterface
{
    public function handle(string $action, array $data): void
    {
        if ($action !== 'write') {
            throw new \InvalidArgumentException(
                "지원하지 않는 로그 액션입니다: {$action}"
            );
        }

        $message = trim((string) ($data['message'] ?? ''));

        file_put_contents(
            '/tmp/app.log',
            date('Y-m-d H:i:s') . ' ' . $message . PHP_EOL,
            FILE_APPEND | LOCK_EX
        );
    }
}

final class LoginController implements RequestControllerInterface
{
    public function handle(string $action, array $data): void
    {
        if ($action !== 'write') {
            throw new \InvalidArgumentException(
                "지원하지 않는 로그인 액션입니다: {$action}"
            );
        }

        $user = trim((string) ($data['user'] ?? ''));

        if ($user === '') {
            throw new \InvalidArgumentException('사용자 정보가 필요합니다.');
        }

        if (session_status() !== PHP_SESSION_ACTIVE) {
            session_start();
        }

        $_SESSION['user'] = $user;
    }
}

final class FaultController implements RequestControllerInterface
{
    public function handle(string $action, array $data): void
    {
        if ($action !== 'write') {
            throw new \InvalidArgumentException(
                "지원하지 않는 오류 액션입니다: {$action}"
            );
        }

        error_log(
            '[FAULT] ' . (string) ($data['message'] ?? 'Unknown fault')
        );
    }
}

final class AjaxRequestHandler
{
    /**
     * 실제 프로젝트에서는 설정 파일이나 DI 컨테이너에서
     * 이 매핑을 관리하도록 확장할 수 있습니다.
     */
    private array $controllers;

    public function __construct()
    {
        $this->controllers = [
            'log' => new LogController(),
            'login' => new LoginController(),
            'fault' => new FaultController(),
        ];
    }

    public function handle(
        string $type,
        string $action,
        array $data
    ): void {
        if (!isset($this->controllers[$type])) {
            throw new \RuntimeException(
                "등록되지 않은 요청 타입입니다: {$type}"
            );
        }

        $this->controllers[$type]->handle($action, $data);
    }
}

// 사용 예시
$handler = new AjaxRequestHandler();

try {
    $handler->handle(
        (string) ($_POST['type'] ?? ''),
        (string) ($_POST['act'] ?? 'write'),
        $_POST
    );

    header('Content-Type: application/json; charset=utf-8');

    echo json_encode(
        ['success' => true],
        JSON_UNESCAPED_UNICODE
    );
} catch (\Throwable $e) {
    http_response_code(400);

    header('Content-Type: application/json; charset=utf-8');

    echo json_encode(
        [
            'success' => false,
            'message' => $e->getMessage(),
        ],
        JSON_UNESCAPED_UNICODE
    );
}
```

### 변경한 부분

요청을 처리하는 코드와 실제 업무 로직을 분리했습니다.

기존에는 새로운 요청 타입을 추가할 때 기존 분기문에 조건을 계속 추가해야 했지만, 변경 후에는 각각의 컨트롤러가 자기 요청을 처리하도록 만들었습니다.

라우팅 자체는 한 곳에서 관리하기 때문에 새로운 요청을 추가하더라도 기존 컨트롤러의 코드를 건드리지 않고 확장할 수 있습니다.

---

## 3. 프론트엔드 공통 HTTP 통신 (JS)

**대상 모듈:** `common_js.php`, `common_js_controller.php`, `common_js_fault.php` → `AppCore.js`

### 기존 문제점

기존 AJAX 함수는 요청 결과를 동기적으로 반환하는 방식이었습니다.

특히 `async: false`를 사용하고 있어 서버 응답을 기다리는 동안 브라우저 UI가 멈출 수 있었고, HTTP 통신 코드와 화면 처리 코드가 서로 강하게 연결되어 있었습니다.

### Before

```javascript
function postArgs(url, data) {
    let result;

    $.ajax({
        type: 'POST',
        url: url,
        data: data,
        async: false,
        success: function (response) {
            result = response;
        }
    });

    return result;
}
```

### After

```javascript
const AppCore = {
    Http: {
        async post(url, payload = {}) {
            const response = await fetch(url, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
                },
                body: new URLSearchParams(payload)
            });

            if (!response.ok) {
                throw new Error(`HTTP 요청 실패: ${response.status}`);
            }

            return response.json();
        },

        async uploadFile(url, file) {
            if (!(file instanceof File)) {
                throw new TypeError('업로드할 파일이 필요합니다.');
            }

            const formData = new FormData();
            formData.append('file', file);

            const response = await fetch(url, {
                method: 'POST',
                body: formData
            });

            if (!response.ok) {
                throw new Error(`파일 업로드 실패: ${response.status}`);
            }

            return response.json();
        }
    }
};

// 사용 예시
async function sendLog() {
    try {
        const result = await AppCore.Http.post('/api/log', {
            type: 'log',
            message: 'test'
        });

        console.log(result);
    } catch (error) {
        console.error(error);
    }
}

sendLog();
```

### 변경한 부분

HTTP 요청 자체만 담당하도록 별도의 모듈로 분리했습니다.

동기 AJAX 대신 `fetch`와 `async / await`를 사용해 요청 중에도 브라우저가 다른 작업을 처리할 수 있도록 했습니다.

파일 업로드 역시 일반 POST와 분리해 `FormData`를 사용하는 별도의 메서드로 구성했습니다.

---

## 4. UI 이벤트 바인딩 (Script)

**대상 모듈:** `common_script.php`, `common_script_controller.php` → `UIComponentBuilder.js`

### 기존 문제점

하나의 JavaScript 파일에 DOM 탐색, 이벤트 등록, 서버 요청, 화면 처리 코드가 함께 들어가 있었습니다.

파일 크기가 커질수록 특정 버튼의 동작을 찾거나 수정하기 어려웠고, 같은 DOM 처리 코드가 여러 곳에서 반복되는 문제도 있었습니다.

### Before

```javascript
$('#btn-submit').click(function () {
    const fileName = $('#file-input').val();

    if (!fileName) {
        alert('파일을 선택하세요.');
        return;
    }

    $.ajax({
        url: '/upload',
        type: 'POST',
        data: {
            file: fileName
        },
        async: false
    });
});
```

### After

```javascript
class UIComponentBuilder {
    constructor(elementId) {
        this.element = document.getElementById(elementId);

        if (!this.element) {
            throw new Error(`요소를 찾을 수 없습니다: #${elementId}`);
        }
    }

    onClick(handler) {
        this.element.addEventListener('click', handler);
        return this;
    }

    setVisible(visible) {
        this.element.style.display = visible ? '' : 'none';
        return this;
    }

    setDisabled(disabled) {
        this.element.disabled = disabled;
        return this;
    }
}

// 사용 예시
const submitButton = new UIComponentBuilder('btn-submit');

submitButton.onClick(() => {
    console.log('submit 버튼 클릭');
});
```

### 변경한 부분

UI 객체가 담당할 기능을 DOM 접근과 이벤트 처리 정도로 제한했습니다.

실제 업로드나 로그인처럼 업무와 관련된 처리는 별도의 컨트롤러에서 담당하도록 구성해 화면 코드가 특정 업무 로직을 직접 알지 않도록 했습니다.

---

## 5. 프론트엔드 파일 업로드 검증 (Auth)

**대상 모듈:** `common_auth.php` → `AuthUIController.js`

### 기존 문제점

파일 선택부터 확장자 검사, 용량 검사, `FormData` 생성, 서버 요청까지 하나의 이벤트 핸들러 안에서 처리하고 있었습니다.

검증 조건이 추가될 때마다 이벤트 함수가 길어지는 구조였습니다.

### Before

```javascript
document
    .getElementById('btn-submit')
    .addEventListener('click', function () {
        const file = document.getElementById('file-input').files[0];

        if (!file) {
            alert('파일을 선택하세요.');
            return;
        }

        if (file.size > 10 * 1024 * 1024) {
            alert('파일 크기가 10MB를 초과합니다.');
            return;
        }

        const formData = new FormData();
        formData.append('file', file);

        // 서버 업로드 처리
    });
```

### After

```javascript
const MAX_FILE_SIZE = 10 * 1024 * 1024;

const ALLOWED_EXTENSIONS = [
    'tar.gz',
    'zip',
    'bin'
];

class AuthUIController {
    static validateFile(file) {
        if (!(file instanceof File)) {
            throw new Error('파일을 선택하세요.');
        }

        if (file.size > MAX_FILE_SIZE) {
            throw new Error('파일 크기는 10MB를 초과할 수 없습니다.');
        }

        const fileName = file.name.toLowerCase();

        const isAllowed = ALLOWED_EXTENSIONS.some(
            (extension) => fileName.endsWith(`.${extension}`)
        );

        if (!isAllowed) {
            throw new Error('허용되지 않는 파일 형식입니다.');
        }
    }

    static async handleUpload() {
        const input = document.getElementById('file-input');

        if (!input) {
            throw new Error('파일 입력 요소를 찾을 수 없습니다.');
        }

        const file = input.files[0];

        AuthUIController.validateFile(file);

        return AppCore.Http.uploadFile('/upload', file);
    }
}

// 이벤트 등록
document
    .getElementById('btn-submit')
    .addEventListener('click', async () => {
        try {
            const result = await AuthUIController.handleUpload();

            console.log('업로드 완료:', result);
        } catch (error) {
            alert(error.message);
        }
    });
```

### 변경한 부분

파일 검증과 실제 업로드 과정을 분리했습니다.

`validateFile()`은 DOM이나 서버 통신을 알 필요가 없기 때문에 별도의 테스트 코드에서도 직접 호출할 수 있습니다.

또한 확장자를 단순히 `split('.')`으로 자르는 대신 `endsWith()`를 이용해 `tar.gz`와 같은 복합 확장자도 처리하도록 변경했습니다.

단, 클라이언트 검증은 우회할 수 있으므로 실제 서버에서도 파일 크기와 형식에 대한 검증을 별도로 수행해야 합니다.

---

## 6. 백엔드 파일 업로드 처리 (Auth Upload)

**대상 모듈:** `common_auth_upload.php` → `SecureFileUploader.php`

### 기존 문제점

업로드된 파일 이름을 그대로 경로에 붙이고, 그 값을 쉘 명령어에 전달하고 있었습니다.

파일 이름을 신뢰할 수 없는 상태에서 시스템 명령을 직접 실행하는 구조라 파일명 조작이나 경로 관련 문제를 방어하기 어려웠습니다.

### Before

```php
<?php

$filePath = '/tmp/' . $_FILES['file']['name'];

move_uploaded_file(
    $_FILES['file']['tmp_name'],
    $filePath
);

pclose(
    popen(
        'sudo tar zxf ' . $filePath . ' -C /tmp/',
        'r'
    )
);
```

### After

```php
<?php

namespace App\Security;

use PharData;
use RuntimeException;
use InvalidArgumentException;

final class SecureFileUploader
{
    private string $uploadDir;

    public function __construct(string $uploadDir)
    {
        $realDir = realpath($uploadDir);

        if ($realDir === false || !is_dir($realDir)) {
            throw new InvalidArgumentException(
                '업로드 디렉터리를 확인할 수 없습니다.'
            );
        }

        $this->uploadDir = $realDir;
    }

    public function extractArchive(array $uploadedFile): bool
    {
        if (($uploadedFile['error'] ?? UPLOAD_ERR_NO_FILE) !== UPLOAD_ERR_OK) {
            throw new RuntimeException('파일 업로드에 실패했습니다.');
        }

        $originalName = (string) ($uploadedFile['name'] ?? '');

        if (!preg_match(
            '/^[a-zA-Z0-9_-]+\.tar\.gz$/',
            $originalName
        )) {
            throw new InvalidArgumentException(
                '허용되지 않는 파일 형식입니다.'
            );
        }

        $temporaryPath = (string) ($uploadedFile['tmp_name'] ?? '');

        if (!is_uploaded_file($temporaryPath)) {
            throw new RuntimeException('유효한 업로드 파일이 아닙니다.');
        }

        $fileName = basename($originalName);
        $archivePath = $this->uploadDir . DIRECTORY_SEPARATOR . $fileName;

        if (!move_uploaded_file($temporaryPath, $archivePath)) {
            throw new RuntimeException('파일 저장에 실패했습니다.');
        }

        try {
            /*
             * shell 명령어를 직접 실행하지 않고 PHP의 PharData를 사용합니다.
             * tar.gz → tar 변환 후 압축을 해제합니다.
             */
            $tarPath = preg_replace(
                '/\.gz$/i',
                '',
                $archivePath
            );

            if ($tarPath === null) {
                throw new RuntimeException('파일 경로를 처리할 수 없습니다.');
            }

            $phar = new PharData($archivePath);
            $phar->decompress();

            $tar = new PharData($tarPath);

            if (!$tar->extractTo($this->uploadDir, null, true)) {
                throw new RuntimeException('압축 해제에 실패했습니다.');
            }

            return true;
        } finally {
            @unlink($archivePath);

            if (isset($tarPath)) {
                @unlink($tarPath);
            }
        }
    }
}
```

### 사용 예시

```php
<?php

require_once __DIR__ . '/SecureFileUploader.php';

use App\Security\SecureFileUploader;

header('Content-Type: application/json; charset=utf-8');

try {
    $uploader = new SecureFileUploader('/tmp/uploads');

    $uploader->extractArchive($_FILES['file'] ?? []);

    echo json_encode(
        ['success' => true],
        JSON_UNESCAPED_UNICODE
    );
} catch (\Throwable $e) {
    http_response_code(400);

    echo json_encode(
        [
            'success' => false,
            'message' => $e->getMessage()
        ],
        JSON_UNESCAPED_UNICODE
    );
}
```

### 변경한 부분

기존처럼 파일명을 그대로 쉘 명령어에 연결하지 않고, 업로드 파일의 상태와 이름을 먼저 확인하도록 변경했습니다.

또한 `sudo tar ...`와 같은 외부 명령 실행 자체를 제거하고 PHP에서 제공하는 `PharData`를 사용하도록 변경했습니다.

다만 압축 파일 내부에 다른 경로를 지정하는 엔트리가 들어오는 경우까지 고려해야 하므로, 실제 서비스에서는 압축 해제 전에 **압축 파일 내부 경로에 대한 추가 검증**도 적용하는 것이 안전합니다.

---

## 7. 로그인 암호화 (Login)

**대상 모듈:** `common_login.php` → `CryptoService.js` / `LoginManager.js`

### 기존 문제점

암호화 클래스가 직접 DOM에서 비밀번호와 공개키를 읽고 있었습니다.

암호화 기능 자체는 로그인 화면과 관계가 없는데도 DOM에 의존하고 있어 다른 화면에서 재사용하기 어려운 구조였습니다.

### Before

```javascript
class CryptFunc {
    rsaEncrypt() {
        const publicKey =
            document.getElementById('public-key').value;

        const password =
            document.getElementById('input-password').value;

        const encryptor = new JSEncrypt();

        encryptor.setPublicKey(publicKey);

        return encryptor.encrypt(password);
    }
}
```

### After

```javascript
// CryptoService.js

export class CryptoService {
    static encryptRSA(publicKey, plainText) {
        if (!publicKey) {
            throw new Error('공개키가 필요합니다.');
        }

        if (!plainText) {
            throw new Error('암호화할 값이 필요합니다.');
        }

        const encryptor = new JSEncrypt();

        encryptor.setPublicKey(publicKey);

        const encrypted = encryptor.encrypt(plainText);

        if (!encrypted) {
            throw new Error('암호화에 실패했습니다.');
        }

        return encrypted;
    }
}
```

```javascript
// LoginManager.js

import { CryptoService } from './CryptoService.js';

export class LoginManager {
    static async handleSubmit(event) {
        event.preventDefault();

        const publicKeyElement =
            document.getElementById('public-key');

        const passwordElement =
            document.getElementById('input-password');

        if (!publicKeyElement || !passwordElement) {
            throw new Error('로그인 입력 요소를 찾을 수 없습니다.');
        }

        const publicKey = publicKeyElement.value;
        const password = passwordElement.value;

        try {
            const encryptedPassword =
                CryptoService.encryptRSA(
                    publicKey,
                    password
                );

            const result = await AppCore.Http.post(
                '/api/login',
                {
                    password: encryptedPassword
                }
            );

            console.log('로그인 처리 완료:', result);
        } catch (error) {
            alert(error.message);
        }
    }
}
```

```javascript
// 사용 예시

import { LoginManager } from './LoginManager.js';

const loginForm = document.getElementById('login-form');

if (loginForm) {
    loginForm.addEventListener(
        'submit',
        LoginManager.handleSubmit
    );
}
```

### 변경한 부분

암호화 자체는 `CryptoService`, 화면에서 값을 읽고 서버에 요청하는 과정은 `LoginManager`가 담당하도록 분리했습니다.

이렇게 하면 `CryptoService`는 DOM을 전혀 참조하지 않기 때문에 테스트하거나 다른 기능에서 재사용하기가 쉬워집니다.

---

## 8. 세션 관리 (Session)

**대상 모듈:** `common_session.php` → `SessionManager.php`

### 기존 문제점

세션 목록을 하나의 텍스트 파일에서 직접 읽고 수정하는 방식이었습니다.

또한 읽기와 쓰기가 별도의 작업으로 이루어져 있어 여러 요청이 동시에 들어오면 같은 데이터를 읽은 뒤 서로 덮어쓸 가능성이 있었습니다.

### Before

```php
<?php

$fp = fopen('/tmp/session_list.json', 'r');

$content = fread(
    $fp,
    filesize('/tmp/session_list.json')
);

fclose($fp);

$sessions = json_decode($content, true) ?? [];

if (isset($_SESSION['id'])) {
    $isActive = isset($sessions[$_SESSION['id']]);
}
```

### After

```php
<?php

namespace App\Auth;

interface SessionStorageInterface
{
    public function fetchAll(): array;

    public function save(
        string $sessionId,
        array $data
    ): void;

    public function delete(string $sessionId): void;
}

final class FileSessionStorage implements SessionStorageInterface
{
    private string $filePath;

    public function __construct(string $filePath)
    {
        $directory = dirname($filePath);

        if (!is_dir($directory)) {
            throw new \InvalidArgumentException(
                '세션 저장 디렉터리가 존재하지 않습니다.'
            );
        }

        $this->filePath = $filePath;

        if (!file_exists($this->filePath)) {
            file_put_contents(
                $this->filePath,
                '{}',
                LOCK_EX
            );
        }
    }

    public function fetchAll(): array
    {
        $fp = fopen($this->filePath, 'c+');

        if ($fp === false) {
            throw new \RuntimeException(
                '세션 파일을 열 수 없습니다.'
            );
        }

        try {
            flock($fp, LOCK_SH);

            rewind($fp);

            $content = stream_get_contents($fp);

            flock($fp, LOCK_UN);
        } finally {
            fclose($fp);
        }

        if ($content === false || trim($content) === '') {
            return [];
        }

        $data = json_decode($content, true);

        return is_array($data) ? $data : [];
    }

    public function save(
        string $sessionId,
        array $data
    ): void {
        $this->update(function (array $sessions) use (
            $sessionId,
            $data
        ) {
            $sessions[$sessionId] = $data;

            return $sessions;
        });
    }

    public function delete(string $sessionId): void
    {
        $this->update(function (array $sessions) use ($sessionId) {
            unset($sessions[$sessionId]);

            return $sessions;
        });
    }

    private function update(callable $callback): void
    {
        $fp = fopen($this->filePath, 'c+');

        if ($fp === false) {
            throw new \RuntimeException(
                '세션 파일을 열 수 없습니다.'
            );
        }

        try {
            flock($fp, LOCK_EX);

            rewind($fp);

            $content = stream_get_contents($fp);

            $sessions = json_decode(
                $content ?: '{}',
                true
            );

            if (!is_array($sessions)) {
                $sessions = [];
            }

            $sessions = $callback($sessions);

            $json = json_encode(
                $sessions,
                JSON_UNESCAPED_UNICODE
            );

            if ($json === false) {
                throw new \RuntimeException(
                    '세션 데이터를 JSON으로 변환하지 못했습니다.'
                );
            }

            rewind($fp);
            ftruncate($fp, 0);

            fwrite($fp, $json);
            fflush($fp);

            flock($fp, LOCK_UN);
        } finally {
            fclose($fp);
        }
    }
}

final class SessionManager
{
    public function __construct(
        private SessionStorageInterface $storage
    ) {
    }

    public function getActiveSessions(): array
    {
        return $this->storage->fetchAll();
    }

    public function isActive(string $sessionId): bool
    {
        return isset(
            $this->storage->fetchAll()[$sessionId]
        );
    }
}

// 사용 예시

$storage = new FileSessionStorage(
    '/tmp/session_list.json'
);

$sessionManager = new SessionManager($storage);

$sessionManager->save(
    'session-001',
    [
        'userId' => 100,
        'createdAt' => time()
    ]
);

$isActive = $sessionManager->isActive('session-001');

var_dump($isActive);
```

### 변경한 부분

단순히 `LOCK_SH`, `LOCK_EX`를 추가하는 것에서 끝내지 않고, 실제 수정 과정 전체를 하나의 배타적 잠금 안에서 처리하도록 변경했습니다.

이렇게 해야 두 요청이 동시에 같은 데이터를 읽고 각각 수정한 뒤 서로 덮어쓰는 문제를 줄일 수 있습니다.

또한 `SessionManager`는 실제 저장 방식에 직접 의존하지 않고 `SessionStorageInterface`만 바라보도록 했습니다.

따라서 이후 저장소를 파일에서 DB나 Redis 등으로 변경할 경우 `SessionManager`의 코드를 수정하지 않고 새로운 구현체를 연결할 수 있습니다.

---

## 9. 데이터베이스 소켓 통신 (SQLite Interface)

**대상 모듈:** `common_sqlite_interface.php` → `DatabaseSocketClient.php`

### 기존 문제점

소켓 연결이나 데이터 전송에 실패하면 `die()`를 호출하는 구조였습니다.

웹 요청을 처리하던 중 통신 오류가 발생했을 때 호출한 코드에서 예외 상황을 판단할 기회가 없고, 실패 상황에 따라 적절한 HTTP 응답을 만들기도 어려웠습니다.

### Before

```php
<?php

function queryInterface(
    string $path,
    string $query
): ?array {
    $socket = socket_create(
        AF_INET,
        SOCK_STREAM,
        SOL_TCP
    );

    socket_connect(
        $socket,
        '127.0.0.1',
        9000
    );

    $message = json_encode([
        $path,
        $query
    ]);

    socket_write(
        $socket,
        $message,
        strlen($message)
    ) or die('Could not send data');

    $result = socket_read(
        $socket,
        4096
    );

    socket_close($socket);

    return json_decode(
        $result,
        true
    );
}
```

### After

```php
<?php

namespace App\Database;

class ConnectionException extends \RuntimeException
{
}

final class DatabaseSocketClient
{
    public function __construct(
        private string $host = '127.0.0.1',
        private int $port = 9000,
        private int $timeout = 5
    ) {
    }

    public function executeQuery(
        string $path,
        string $query
    ): array {
        $socket = socket_create(
            AF_INET,
            SOCK_STREAM,
            SOL_TCP
        );

        if ($socket === false) {
            throw new ConnectionException(
                '소켓을 생성할 수 없습니다.'
            );
        }

        try {
            socket_set_option(
                $socket,
                SOL_SOCKET,
                SO_RCVTIMEO,
                [
                    'sec' => $this->timeout,
                    'usec' => 0
                ]
            );

            socket_set_option(
                $socket,
                SOL_SOCKET,
                SO_SNDTIMEO,
                [
                    'sec' => $this->timeout,
                    'usec' => 0
                ]
            );

            if (!socket_connect(
                $socket,
                $this->host,
                $this->port
            )) {
                throw new ConnectionException(
                    '데이터베이스 서버에 연결하지 못했습니다.'
                );
            }

            $message = json_encode([
                'path' => $path,
                'query' => $query
            ]);

            if ($message === false) {
                throw new ConnectionException(
                    '요청 데이터를 생성하지 못했습니다.'
                );
            }

            $length = strlen($message);
            $written = 0;

            while ($written < $length) {
                $result = socket_write(
                    $socket,
                    substr($message, $written)
                );

                if ($result === false) {
                    throw new ConnectionException(
                        '소켓 데이터 전송에 실패했습니다.'
                    );
                }

                $written += $result;
            }

            $response = socket_read(
                $socket,
                4096
            );

            if ($response === false) {
                throw new ConnectionException(
                    '응답을 읽지 못했습니다.'
                );
            }

            $decoded = json_decode(
                $response,
                true
            );

            if (!is_array($decoded)) {
                throw new ConnectionException(
                    '잘못된 응답 형식입니다.'
                );
            }

            return $decoded;
        } finally {
            socket_close($socket);
        }
    }
}

// 사용 예시

use App\Database\DatabaseSocketClient;
use App\Database\ConnectionException;

$client = new DatabaseSocketClient();

try {
    $result = $client->executeQuery(
        '/db/users',
        'SELECT * FROM users LIMIT 10'
    );

    header('Content-Type: application/json; charset=utf-8');

    echo json_encode(
        $result,
        JSON_UNESCAPED_UNICODE
    );
} catch (ConnectionException $e) {
    http_response_code(503);

    echo json_encode(
        [
            'success' => false,
            'message' => $e->getMessage()
        ],
        JSON_UNESCAPED_UNICODE
    );
}
```

### 변경한 부분

`die()`를 제거하고 통신 오류를 예외로 전달하도록 변경했습니다.

호출하는 쪽에서는 연결 실패, 전송 실패, 응답 오류 등을 하나의 예외 흐름으로 처리할 수 있고, 상황에 맞는 HTTP 상태 코드도 반환할 수 있습니다.

또한 송수신 타임아웃을 설정해 외부 프로세스가 응답하지 않는 상황에서 요청이 계속 대기하는 문제도 방지했습니다.

---

## 10. 다국어 로그 처리 (Log)

**대상 모듈:** `common_log.php` → `Logger.php`

### 기존 문제점

로그를 기록할 때마다 메시지 전체를 정규식으로 검색해서 `{KEY}` 형태의 값을 찾고, 해당 키에 대응하는 문구를 다시 조회하는 방식이었습니다.

로그가 많이 발생하는 환경에서는 같은 패턴을 매번 분석하는 작업이 불필요하게 반복될 수 있었습니다.

### Before

```php
<?php

function writeLog(string $text): void
{
    preg_match_all(
        '#\{(.*?)\}#',
        $text,
        $matches
    );

    foreach ($matches[1] as $key) {
        $constantName = 'Lang\\' . $key;

        $text = preg_replace(
            '#\{' . preg_quote($key, '#') . '\}#',
            constant($constantName),
            $text
        );
    }

    error_log($text);
}
```

### After

```php
<?php

namespace App\Logging;

interface TranslatorInterface
{
    public function get(string $key): string;
}

final class ArrayTranslator implements TranslatorInterface
{
    public function __construct(
        private array $messages
    ) {
    }

    public function get(string $key): string
    {
        return $this->messages[$key] ?? $key;
    }
}

final class Logger
{
    public function __construct(
        private TranslatorInterface $translator,
        private string $logFile = ''
    ) {
    }

    public function log(
        string $level,
        string $key,
        array $context = []
    ): void {
        $message = $this->translator->get($key);

        foreach ($context as $name => $value) {
            $message = str_replace(
                '{' . $name . '}',
                (string) $value,
                $message
            );
        }

        $formatted = sprintf(
            '[%s] [%s] %s',
            date('Y-m-d H:i:s'),
            strtoupper($level),
            $message
        );

        if ($this->logFile !== '') {
            file_put_contents(
                $this->logFile,
                $formatted . PHP_EOL,
                FILE_APPEND | LOCK_EX
            );

            return;
        }

        error_log($formatted);
    }

    public function info(
        string $key,
        array $context = []
    ): void {
        $this->log(
            'info',
            $key,
            $context
        );
    }

    public function warn(
        string $key,
        array $context = []
    ): void {
        $this->log(
            'warn',
            $key,
            $context
        );
    }

    public function error(
        string $key,
        array $context = []
    ): void {
        $this->log(
            'error',
            $key,
            $context
        );
    }
}

// 사용 예시

$translator = new ArrayTranslator([
    'LOGIN_SUCCESS' => '로그인에 성공했습니다.',
    'LOGIN_FAIL' => '로그인에 실패했습니다.',
    'USER_CREATED' => '{name} 사용자가 생성되었습니다.'
]);

$logger = new Logger(
    $translator,
    '/tmp/app.log'
);

$logger->info('LOGIN_SUCCESS');

$logger->error('LOGIN_FAIL');

$logger->info(
    'USER_CREATED',
    [
        'name' => 'Alice'
    ]
);
```

### 변경한 부분

로그 메시지 자체를 정규식으로 분석하는 대신 메시지 키를 바로 번역 테이블에서 조회하도록 변경했습니다.

예를 들어 `LOGIN_SUCCESS`라는 키를 받으면 배열에서 바로 값을 찾기 때문에 메시지 검색 과정에서 정규식을 사용할 필요가 없습니다.

또한 번역 기능을 `TranslatorInterface`로 분리했기 때문에 현재는 배열을 사용하더라도 나중에 파일이나 DB 기반 번역 구현체로 교체할 수 있습니다.

---

# 정리

이번 리팩토링에서는 기존 `common_` 모듈을 단순히 파일별로 다시 나누는 것보다 **각 코드가 어떤 책임을 가지고 있는지**를 기준으로 구조를 정리했습니다.

주요 변경 사항은 다음과 같습니다.

* 전역 상수를 용도별 설정 클래스로 정리
* 요청 타입별 처리 로직을 컨트롤러로 분리
* 동기 AJAX를 `fetch` 기반 비동기 통신으로 변경
* DOM 조작과 업무 로직 분리
* 파일 검증과 업로드 처리 분리
* 파일 업로드 과정에서 외부 쉘 명령 의존성 제거
* 암호화 기능을 DOM과 분리
* 세션 저장소를 인터페이스 기반으로 추상화
* 소켓 통신 오류를 `die()` 대신 예외로 처리
* 로그 메시지 조회와 번역 기능을 분리

결과적으로 공통 모듈을 수정할 때 다른 기능까지 함께 수정해야 하는 경우를 줄이고, 각각의 기능을 독립적으로 테스트하거나 교체할 수 있는 구조를 만드는 데 초점을 맞췄습니다.
