# 레거시 Common 모듈을 책임별로 분리했습니다

레거시 `common_` 모듈을 파일 단위로만 다시 나누는 대신, 설정·HTTP·UI 이벤트·파일 업로드·암호화·세션·소켓·로그가 서로 다른 책임이라는 기준으로 구조를 정리했습니다.

## 프로젝트 정보

- 팀/소속: SW2팀
- 직접 설계·구현 범위: 아래 Common module 리팩토링 사례
- 기술적 의사결정: 직접 수행

---

# 01. 설정과 요청 처리를 분리해 변경 지점을 줄였습니다

기존에는 설정 파일을 읽는 코드와 이를 사용하는 요청 처리가 여러 파일에 흩어져 있었습니다.

```php
$config = json_decode(
    file_get_contents('/path/config.json'),
    true
);
```

설정 접근 책임을 별도 객체로 이동했습니다.

```php
final class AppConfig
{
    public function __construct(
        private array $values
    ) {}

    public function get(string $key): mixed
    {
        return $this->values[$key] ?? null;
    }
}
```

설정 데이터를 사용하는 코드가 파일이나 전역 상태를 직접 다루지 않도록 경계를 만들었습니다.

### 무엇이 달라졌는가

설정 접근과 요청 처리 책임이 분리됐고, 설정 소스 변경 시 영향을 받는 범위를 줄일 수 있는 구조가 됐습니다.

---

# 02. 브라우저를 막던 동기 HTTP 요청을 공통 비동기 Client로 전환했습니다

기존 Frontend 요청에는 `async: false`를 사용하는 동기 AJAX가 존재했습니다.

```javascript
$.ajax({
    url,
    async: false,
    type: 'POST'
});
```

동기 요청은 브라우저의 실행 흐름을 기다리게 만들기 때문에 공통 통신 계층에서 비동기 Client를 두었습니다.

```javascript
class HttpClient {
    static async post(url, data) {
        const response = await fetch(url, {
            method: 'POST',
            body: data
        });

        if (!response.ok) {
            throw new Error(`HTTP ${response.status}`);
        }

        return response.text();
    }
}
```

### 무엇이 달라졌는가

- `async: false` 제거
- Fetch / async-await 기반 공통 처리
- HTTP 오류를 예외/Promise 흐름으로 전달

---

# 03. UI 이벤트에서 업무 로직을 분리했습니다

기존 이벤트 handler 안에는 DOM 조회, 입력 검증, 데이터 생성, API 호출, 결과 처리까지 한꺼번에 들어 있었습니다.

이벤트 계층은 사용자 입력에 집중하고, 검증과 실제 요청은 별도 메서드로 분리했습니다.

```javascript
class AuthUIController {
    static validateFile(file) {
        // input validation only
    }

    static async handleUpload() {
        const file = input.files[0];

        AuthUIController.validateFile(file);

        return AppCore.Http.uploadFile(
            '/upload',
            file
        );
    }
}
```

이 구조를 통해 검증 함수는 DOM이나 서버 통신과 독립적으로 다룰 수 있게 했습니다.

---

# 04. 파일 선택 단계에서 검증을 분리했습니다

파일 선택, 확장자 검사, 용량 검사, FormData 생성, 서버 요청을 하나의 handler가 담당하면 변경 지점이 커집니다.

그래서 `validateFile()`과 실제 업로드를 분리했습니다.

```javascript
const MAX_FILE_SIZE = 10 * 1024 * 1024;

const ALLOWED_EXTENSIONS = [
    'tar.gz',
    'zip',
    'bin'
];

static validateFile(file) {
    if (!(file instanceof File)) {
        throw new Error('파일을 선택하세요.');
    }

    if (file.size > MAX_FILE_SIZE) {
        throw new Error('파일 크기를 초과했습니다.');
    }

    const fileName = file.name.toLowerCase();

    const allowed = ALLOWED_EXTENSIONS.some(
        ext => fileName.endsWith(`.${ext}`)
    );

    if (!allowed) {
        throw new Error('허용되지 않는 파일 형식입니다.');
    }
}
```

클라이언트 검증은 UX를 위한 1차 경계이고, 서버 검증을 대신하지 않습니다.

---

# 05. 업로드 파일이 시스템 명령까지 직접 이어지지 않도록 경계를 바꿨습니다

기존 구조에서는 업로드된 파일명이 경로에 그대로 결합된 뒤 shell command에 전달됐습니다.

```php
$filePath = '/tmp/' . $_FILES['file']['name'];

pclose(
    popen(
        'sudo tar zxf ' . $filePath . ' -C /tmp/',
        'r'
    )
);
```

저는 외부 입력이 shell command로 직접 이어지는 경로를 제거하고 PHP `PharData`로 압축을 처리했습니다.

```text
Upload
 ↓
Upload Error Validation
 ↓
Filename Validation
 ↓
is_uploaded_file
 ↓
Controlled Directory
 ↓
PharData
 ↓
Temporary Archive Cleanup
```

### 무엇이 달라졌는가

- 파일명 형식 검증
- 업로드 파일 검증
- 업로드 경로 검증
- 외부 `tar` 실행 제거
- 임시 archive 삭제

---

# 06. 로그인 암호화를 UI에서 분리했습니다

기존에는 로그인 화면이 DOM에서 비밀번호와 공개키를 직접 읽으면서 암호화까지 수행했습니다.

암호화 기능을 `CryptoService`로 이동했습니다.

```javascript
export class CryptoService {
    static encryptRSA(publicKey, plainText) {
        if (!publicKey) {
            throw new Error('공개키가 필요합니다.');
        }

        if (!plainText) {
            throw new Error('암호화할 값이 필요합니다.');
        }

        // JSEncrypt 기반 RSA encryption
    }
}
```

`LoginManager`는 입력값을 읽고 `CryptoService`를 호출하는 역할에 집중합니다.

### 무엇이 달라졌는가

암호화 구현과 Login UI가 분리되어 다른 호출부에서도 사용할 수 있는 구조가 됐습니다.

실제 다른 화면에서 재사용한 결과가 있다면 추가로 기입할 수 있습니다.

---

# 07. 세션 관리와 저장소 구현을 분리했습니다

세션 목록 파일을 읽고 수정하는 코드와 세션 관리 로직이 한곳에 묶여 있으면 동시에 접근할 때 서로 다른 수정 결과가 덮어써질 수 있습니다.

구조를 다음처럼 분리했습니다.

```text
SessionManager
      ↓
SessionStorageInterface
      ↓
FileSessionStorage
```

수정 과정 전체를 하나의 배타적 lock 범위 안에서 처리했습니다.

```php
flock($fp, LOCK_EX);

$data = read();
$data = $callback($data);
write($data);

flock($fp, LOCK_UN);
```

### 무엇이 달라졌는가

- SessionManager와 저장 방식 분리
- read-modify-write 구간 보호
- 저장소 구현체 교체가 가능한 구조

실제 DB/Redis 교체 구현과 테스트는 별도로 확인할 수 있습니다.

---

# 08. Socket 오류를 `die()` 대신 예외 흐름으로 전달했습니다

기존에는 소켓 통신 실패 시 `die()`로 종료되어 호출부가 실패 상황을 세밀하게 처리하기 어려웠습니다.

그래서:

```text
Connect
 ↓
Timeout
 ↓
Send
 ↓
Receive
 ↓
JSON Validation
 ↓
ConnectionException
```

흐름으로 정리했습니다.

### 무엇이 달라졌는가

상위 계층에서 연결 실패, 전송 실패, 응답 실패를 구분하고 적절한 HTTP 응답으로 연결할 수 있게 됐습니다.

---

# 09. 로그 메시지와 번역 데이터를 분리했습니다

기존에는 로그 문자열 전체를 정규식으로 분석해 `{KEY}`를 찾은 뒤 번역값을 다시 조회했습니다.

새 구조에서는 메시지 Key를 직접 Translator에 전달합니다.

```text
Logger
  ↓
TranslatorInterface
  ↓
ArrayTranslator
```

이렇게 하면 로그 표현 방식과 번역 데이터의 책임이 분리되고 Translator 구현체를 교체할 수 있습니다.

---

# 이 리팩토링에서 공통으로 가져간 원칙

> **UI / 업무 로직 / 저장 / 통신 / 시스템 실행을 가능한 한 서로 다른 책임으로 분리**

그 결과 다음 경계를 만들었습니다.

- DOM ↔ 업무 로직
- Validation ↔ Upload
- SessionManager ↔ Storage
- Socket ↔ HTTP Response
- Crypto ↔ Login UI
- Logger ↔ Translator

공통 모듈 리팩토링의 핵심은 파일 수를 줄이는 것이 아니라 **서로 다른 변경 이유를 서로 다른 책임으로 분리하는 것**이었습니다.
