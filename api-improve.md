# REST API 리팩토링 포트폴리오

레거시 PHP REST API에서 반복적으로 사용되던 인증, 도메인 처리, 시스템 설정, 예외 처리 코드를 기능별로 분리하고 객체지향 구조로 정리한 내용을 정리했습니다.

기존 코드는 하나의 클래스나 함수 안에서 여러 책임을 동시에 처리하는 경우가 많았습니다. 공용 코드 리팩토링에서는 각 기능이 맡은 역할을 분리하고, 저장소나 실행 방식처럼 변경 가능성이 있는 부분은 인터페이스를 통해 교체할 수 있도록 구성했습니다.

---

## 1. 엔트리포인트 및 인증 처리

**핵심 개선 대상:** HTTP Digest 인증 파싱, 사용자 정보 조회, 인증 결과 처리

### 기존 문제점

인증 요청을 처리하는 클래스에서 다음 작업을 한꺼번에 수행하고 있었습니다.

* Digest 인증 헤더 파싱
* 사용자 파일 읽기
* 인증 값 계산
* HTTP 요청 정보 접근
* 응답 처리

이 구조에서는 인증 방식을 테스트하거나 사용자 정보 저장 방식을 변경하려면 기존 클래스까지 함께 수정해야 했습니다.

### Before

```php
<?php

class ApiHandler
{
    public function parseDigest(string $text): array
    {
        preg_match_all(
            '@(\w+)=(?:([\'"])([^\2]+?)\2|([^\s,]+))@',
            $text,
            $matches,
            PREG_SET_ORDER
        );

        $result = [];

        foreach ($matches as $match) {
            $result[$match[1]] = $match[3] ?: $match[4];
        }

        return $result;
    }

    public function isAuthenticated(): bool
    {
        $header = $_SERVER['PHP_AUTH_DIGEST'] ?? '';

        if ($header === '') {
            return false;
        }

        $data = $this->parseDigest($header);

        $filePath = '/var/auth/user_list.json';

        if (!file_exists($filePath)) {
            return false;
        }

        $users = json_decode(
            file_get_contents($filePath),
            true
        ) ?? [];

        $username = $data['username'] ?? '';

        if (!isset($users[$username])) {
            return false;
        }

        $secret = $users[$username];

        $a1 = md5(
            "{$username}:{$data['realm']}:{$secret}"
        );

        $a2 = md5(
            $_SERVER['REQUEST_METHOD'] . ':' . $data['uri']
        );

        $expected = md5(
            "{$a1}:{$data['nonce']}:{$data['nc']}:"
            . "{$data['cnonce']}:{$data['qop']}:{$a2}"
        );

        return $expected === ($data['response'] ?? '');
    }

    public function respond(int $statusCode, array $data): void
    {
        http_response_code($statusCode);

        header(
            'Content-Type: application/json; charset=utf-8'
        );

        echo json_encode(
            $data,
            JSON_UNESCAPED_UNICODE
        );
    }
}
```

### After

```php
<?php

declare(strict_types=1);

namespace App\Http\Auth;

interface UserRepositoryInterface
{
    public function getSecret(string $username): ?string;
}

final class FileUserRepository implements UserRepositoryInterface
{
    private array $users;

    public function __construct(string $filePath)
    {
        if (!is_file($filePath)) {
            throw new \RuntimeException(
                "사용자 파일을 찾을 수 없습니다: {$filePath}"
            );
        }

        $content = file_get_contents($filePath);

        if ($content === false) {
            throw new \RuntimeException(
                '사용자 파일을 읽을 수 없습니다.'
            );
        }

        $users = json_decode($content, true);

        if (!is_array($users)) {
            throw new \RuntimeException(
                '사용자 파일 형식이 올바르지 않습니다.'
            );
        }

        $this->users = $users;
    }

    public function getSecret(string $username): ?string
    {
        return isset($this->users[$username])
            ? (string) $this->users[$username]
            : null;
    }
}

final class DigestAuthenticator
{
    public function __construct(
        private UserRepositoryInterface $userRepository
    ) {
    }

    public function authenticate(
        string $digestHeader,
        string $method,
        string $uri
    ): bool {
        if ($digestHeader === '') {
            return false;
        }

        $data = $this->parseDigest($digestHeader);

        if ($data === null) {
            return false;
        }

        $required = [
            'username',
            'realm',
            'nonce',
            'uri',
            'response'
        ];

        foreach ($required as $field) {
            if (!isset($data[$field])) {
                return false;
            }
        }

        $secret = $this->userRepository->getSecret(
            $data['username']
        );

        if ($secret === null) {
            return false;
        }

        return $this->validateResponse(
            $data,
            $secret,
            $method,
            $uri
        );
    }

    private function parseDigest(string $header): ?array
    {
        if (str_starts_with($header, 'Digest ')) {
            $header = substr($header, 7);
        }

        preg_match_all(
            '@(\w+)=(?:"([^"]*)"|([^,\s]+))@',
            $header,
            $matches,
            PREG_SET_ORDER
        );

        if (empty($matches)) {
            return null;
        }

        $data = [];

        foreach ($matches as $match) {
            $data[$match[1]] =
                $match[2] !== ''
                    ? $match[2]
                    : $match[3];
        }

        return $data;
    }

    private function validateResponse(
        array $data,
        string $secret,
        string $method,
        string $requestUri
    ): bool {
        $a1 = md5(
            "{$data['username']}:{$data['realm']}:{$secret}"
        );

        $a2 = md5(
            "{$method}:{$data['uri']}"
        );

        /*
         * 기존 코드와 동일하게 qop 인증 흐름을 사용합니다.
         * 실제 운영 환경에서는 Digest 알고리즘 및 nonce
         * 정책도 함께 검토해야 합니다.
         */
        if (
            isset(
                $data['qop'],
                $data['nc'],
                $data['cnonce']
            )
        ) {
            $expected = md5(
                "{$a1}:{$data['nonce']}:"
                . "{$data['nc']}:{$data['cnonce']}:"
                . "{$data['qop']}:{$a2}"
            );
        } else {
            $expected = md5(
                "{$a1}:{$data['nonce']}:{$a2}"
            );
        }

        /*
         * Digest 헤더의 URI와 실제 요청 URI가 다른 경우
         * 요청을 그대로 통과시키지 않습니다.
         */
        $requestPath = parse_url(
            $requestUri,
            PHP_URL_PATH
        );

        $digestPath = parse_url(
            $data['uri'],
            PHP_URL_PATH
        );

        if (
            $requestPath !== null &&
            $digestPath !== null &&
            $requestPath !== $digestPath
        ) {
            return false;
        }

        return hash_equals(
            $expected,
            $data['response']
        );
    }
}

final class JsonResponse
{
    public function send(
        int $statusCode,
        array $data
    ): void {
        http_response_code($statusCode);

        header(
            'Content-Type: application/json; charset=utf-8'
        );

        echo json_encode(
            $data,
            JSON_UNESCAPED_UNICODE
        );
    }
}

// 사용 예시
//
// users.json 예:
// {
//     "alice": "password123"
// }

$repository = new FileUserRepository(
    __DIR__ . '/users.json'
);

$authenticator = new DigestAuthenticator(
    $repository
);

$header = $_SERVER['PHP_AUTH_DIGEST'] ?? '';
$method = $_SERVER['REQUEST_METHOD'] ?? 'GET';
$uri = $_SERVER['REQUEST_URI'] ?? '/';

$isAuthenticated = $authenticator->authenticate(
    $header,
    $method,
    $uri
);

$response = new JsonResponse();

if (!$isAuthenticated) {
    $response->send(
        401,
        [
            'success' => false,
            'message' => '인증에 실패했습니다.'
        ]
    );

    exit;
}

$response->send(
    200,
    [
        'success' => true,
        'message' => '인증되었습니다.'
    ]
);
```

### 변경한 부분

인증에 필요한 사용자 정보 조회를 `UserRepositoryInterface`로 분리했습니다.

`DigestAuthenticator`는 사용자 정보가 파일에 저장되어 있는지 알 필요 없이 저장소 인터페이스만 사용합니다.

따라서 나중에 사용자 정보를 DB에서 가져오더라도 `DigestAuthenticator` 자체는 그대로 사용할 수 있습니다.

또한 기존 코드에서 사용하던 Digest 계산 방식을 유지하면서 입력값 검증과 `hash_equals()`를 추가했습니다.

MD5 기반 HTTP Digest 자체가 최신 인증 방식은 아니지만, 기존 인증 로직을 보존하며 사이드 이펙트가 발생하지 않을 선에서 각 기능별로 분리 하였습니다.

---

## 2. 도메인별 API 핸들러 - Audio & DSP

**핵심 개선 대상:** 장비별 DSP 입력값 조회/저장, Gain 근사값 계산

### 기존 문제점

장비 종류가 함수 이름에 직접 들어가 있어 장비가 추가될 때마다 새로운 함수가 계속 생기는 구조였습니다.

또한 Gain 값을 찾는 계산 함수와 장비별 처리 코드가 같은 영역에 섞여 있어 각각의 책임이 명확하지 않았습니다.

### Before

```php
<?php

function getClosestGain(
    array $gainList,
    float $targetValue
): float {
    sort($gainList);

    $closest = $gainList[0];

    foreach ($gainList as $value) {
        if (
            abs($value - $targetValue)
            < abs($closest - $targetValue)
        ) {
            $closest = $value;
        }
    }

    return $closest;
}

function getAudioInputForTypeA(): array
{
    $gains = [-60, -40, -20, 0, 6, 12];

    $rawGain = 5.3;

    return [
        'gain' => getClosestGain(
            $gains,
            $rawGain
        )
    ];
}

function saveAudioInputForTypeA(
    array $requestData
): bool {
    $gain = $requestData['gain'] ?? null;

    if ($gain === null) {
        return false;
    }

    // 저장 처리

    return true;
}
```

### After

```php
<?php

declare(strict_types=1);

namespace App\Audio;

final class GainCalculator
{
    public static function closest(
        array $candidates,
        float $target
    ): float {
        if ($candidates === []) {
            throw new \InvalidArgumentException(
                'Gain 후보 목록이 비어 있습니다.'
            );
        }

        $closest = (float) $candidates[0];
        $closestDistance = abs(
            $closest - $target
        );

        foreach ($candidates as $candidate) {
            $candidate = (float) $candidate;

            $distance = abs(
                $candidate - $target
            );

            if ($distance < $closestDistance) {
                $closest = $candidate;
                $closestDistance = $distance;
            }
        }

        return $closest;
    }
}

interface DspServiceInterface
{
    public function getInputData(): array;

    public function saveInputData(array $data): bool;
}

abstract class AbstractDspService
    implements DspServiceInterface
{
    public function __construct(
        protected array $validGains
    ) {
    }

    protected function normalizeGain(
        float $rawGain
    ): float {
        return GainCalculator::closest(
            $this->validGains,
            $rawGain
        );
    }

    public function saveInputData(
        array $data
    ): bool {
        if (!isset($data['gain'])) {
            throw new \InvalidArgumentException(
                'gain 값이 필요합니다.'
            );
        }

        $gain = (float) $data['gain'];

        if (!in_array(
            $gain,
            $this->validGains,
            true
        )) {
            throw new \InvalidArgumentException(
                '지원하지 않는 gain 값입니다.'
            );
        }

        /*
         * 실제 프로젝트에서는 이 부분에서
         * 장비 설정 또는 저장소에 값을 반영합니다.
         */
        return true;
    }
}

final class TypeADspService
    extends AbstractDspService
{
    public function __construct()
    {
        parent::__construct([
            -60,
            -40,
            -20,
            0,
            6,
            12
        ]);
    }

    public function getInputData(): array
    {
        $rawGain = 5.3;

        return [
            'gain' => $this->normalizeGain(
                $rawGain
            )
        ];
    }
}

final class TypeBDspService
    extends AbstractDspService
{
    public function __construct()
    {
        parent::__construct([
            -30,
            -10,
            0,
            10,
            20
        ]);
    }

    public function getInputData(): array
    {
        $rawGain = 8.0;

        return [
            'gain' => $this->normalizeGain(
                $rawGain
            )
        ];
    }
}

final class DspServiceFactory
{
    public function make(
        string $model
    ): DspServiceInterface {
        return match ($model) {
            'type-a' => new TypeADspService(),
            'type-b' => new TypeBDspService(),

            default => throw new \InvalidArgumentException(
                "지원하지 않는 장비 모델입니다: {$model}"
            )
        };
    }
}

final class AudioDspController
{
    public function __construct(
        private DspServiceFactory $factory
    ) {
    }

    public function getInput(
        string $deviceModel
    ): array {
        return $this->factory
            ->make($deviceModel)
            ->getInputData();
    }

    public function saveInput(
        string $deviceModel,
        array $requestData
    ): bool {
        return $this->factory
            ->make($deviceModel)
            ->saveInputData(
                $requestData
            );
    }
}

// 사용 예시

$factory = new DspServiceFactory();

$controller = new AudioDspController(
    $factory
);

$model = $_GET['model'] ?? 'type-a';

try {
    $result = $controller->getInput($model);

    header(
        'Content-Type: application/json; charset=utf-8'
    );

    echo json_encode(
        $result,
        JSON_UNESCAPED_UNICODE
    );
} catch (\Throwable $e) {
    http_response_code(400);

    header(
        'Content-Type: application/json; charset=utf-8'
    );

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

Gain 계산은 `GainCalculator`로 분리하고, 장비별 차이는 각각의 `DspService`에서 관리하도록 변경했습니다.

공통적인 Gain 검증과 저장 흐름은 `AbstractDspService`에 두고, 실제 장비마다 다른 값은 각 구현체가 정의하도록 했습니다.

컨트롤러는 장비 종류에 따라 어떤 클래스가 선택되는지 직접 판단하지 않고 `DspServiceFactory`에 위임합니다.

따라서 장비가 추가되면 기존 컨트롤러의 분기문을 늘리는 대신 새로운 `DspService` 구현체와 Factory 등록을 추가하는 방식으로 확장할 수 있습니다.

---

## 3. 네트워크 및 SIP 설정 제어

**핵심 개선 대상:** 네트워크 인터페이스 설정, SIP 계정 저장 및 입력값 검증

### 기존 문제점

네트워크 설정과 SIP 계정 저장이 모두 단순 함수 안에 들어 있었고, 외부 명령 실행에 사용되는 값에 대한 검증도 충분하지 않았습니다.

특히 네트워크 인터페이스나 IP 주소처럼 시스템 명령에 전달되는 값은 예상하지 못한 입력이 들어오지 않도록 별도로 검증할 필요가 있었습니다.

### Before

```php
<?php

function insertSipAccount(
    array $requestData
): void {
    $account = $requestData['account'];
    $password = $requestData['password'];
    $server = $requestData['server'];

    file_put_contents(
        '/etc/sip/accounts.json',
        json_encode([
            'account' => $account,
            'server' => $server
        ])
    );
}

function setNetworkInterface(
    string $type,
    string $ip,
    string $netmask
): void {
    $command =
        "sudo ifconfig {$type} {$ip} "
        . "netmask {$netmask}";

    shell_exec($command);
}
```

### After

```php
<?php

declare(strict_types=1);

namespace App\System\Network;

final class NetworkConfig
{
    public function __construct(
        private readonly string $ip,
        private readonly string $netmask
    ) {
        if (!filter_var(
            $ip,
            FILTER_VALIDATE_IP
        )) {
            throw new \InvalidArgumentException(
                'IP 주소 형식이 올바르지 않습니다.'
            );
        }

        if (!filter_var(
            $netmask,
            FILTER_VALIDATE_IP
        )) {
            throw new \InvalidArgumentException(
                '넷마스크 형식이 올바르지 않습니다.'
            );
        }
    }

    public function getIp(): string
    {
        return $this->ip;
    }

    public function getNetmask(): string
    {
        return $this->netmask;
    }
}

interface CommandRunnerInterface
{
    public function run(string $command): int;
}

final class ShellCommandRunner
    implements CommandRunnerInterface
{
    public function run(string $command): int
    {
        $exitCode = 0;

        exec(
            $command,
            $output,
            $exitCode
        );

        return $exitCode;
    }
}

final class NetworkInterfaceManager
{
    private array $allowedInterfaces = [
        'eth0',
        'eth1',
        'ens33'
    ];

    public function __construct(
        private CommandRunnerInterface $runner
    ) {
    }

    public function setInterface(
        string $interface,
        NetworkConfig $config
    ): void {
        if (!in_array(
            $interface,
            $this->allowedInterfaces,
            true
        )) {
            throw new \InvalidArgumentException(
                '허용되지 않은 네트워크 인터페이스입니다.'
            );
        }

        $safeInterface = escapeshellarg(
            $interface
        );

        $safeIp = escapeshellarg(
            $config->getIp()
        );

        $safeNetmask = escapeshellarg(
            $config->getNetmask()
        );

        $command =
            "sudo /sbin/ifconfig "
            . "{$safeInterface} "
            . "{$safeIp} "
            . "netmask {$safeNetmask}";

        $exitCode = $this->runner->run(
            $command
        );

        if ($exitCode !== 0) {
            throw new \RuntimeException(
                '네트워크 설정에 실패했습니다.'
            );
        }
    }
}

namespace App\System\Sip;

final class SipAccount
{
    public function __construct(
        private readonly string $account,
        private readonly string $password,
        private readonly string $server
    ) {
        if (!preg_match(
            '/^[a-zA-Z0-9._@-]{3,64}$/',
            $account
        )) {
            throw new \InvalidArgumentException(
                '계정 형식이 올바르지 않습니다.'
            );
        }

        if (
            filter_var(
                $server,
                FILTER_VALIDATE_IP
            ) === false &&
            filter_var(
                $server,
                FILTER_VALIDATE_DOMAIN,
                FILTER_FLAG_HOSTNAME
            ) === false
        ) {
            throw new \InvalidArgumentException(
                '서버 주소 형식이 올바르지 않습니다.'
            );
        }
    }

    public function toArray(): array
    {
        return [
            'account' => $this->account,
            'password' => $this->password,
            'server' => $this->server
        ];
    }
}

final class SipAccountRepository
{
    public function __construct(
        private readonly string $filePath
    ) {
    }

    public function save(
        SipAccount $account
    ): void {
        $directory = dirname(
            $this->filePath
        );

        if (!is_dir($directory)) {
            if (!mkdir(
                $directory,
                0775,
                true
            ) && !is_dir($directory)) {
                throw new \RuntimeException(
                    '저장 디렉터리를 생성할 수 없습니다.'
                );
            }
        }

        $json = json_encode(
            $account->toArray(),
            JSON_PRETTY_PRINT |
            JSON_UNESCAPED_UNICODE
        );

        if ($json === false) {
            throw new \RuntimeException(
                'SIP 계정 데이터를 변환할 수 없습니다.'
            );
        }

        if (file_put_contents(
            $this->filePath,
            $json,
            LOCK_EX
        ) === false) {
            throw new \RuntimeException(
                'SIP 계정 저장에 실패했습니다.'
            );
        }
    }
}

namespace App\Api\Controllers;

use App\System\Sip\SipAccount;
use App\System\Sip\SipAccountRepository;

final class SipController
{
    public function __construct(
        private SipAccountRepository $repository
    ) {
    }

    public function storeAccount(
        array $requestData
    ): array {
        foreach (
            ['account', 'password', 'server']
            as $field
        ) {
            if (
                !isset($requestData[$field]) ||
                trim((string) $requestData[$field]) === ''
            ) {
                throw new \InvalidArgumentException(
                    "필수 값이 없습니다: {$field}"
                );
            }
        }

        $account = new SipAccount(
            trim((string) $requestData['account']),
            (string) $requestData['password'],
            trim((string) $requestData['server'])
        );

        $this->repository->save(
            $account
        );

        return [
            'success' => true
        ];
    }
}

// 사용 예시

use App\Api\Controllers\SipController;
use App\System\Sip\SipAccountRepository;
use App\System\Network\NetworkConfig;
use App\System\Network\NetworkInterfaceManager;
use App\System\Network\ShellCommandRunner;

$sipRepository = new SipAccountRepository(
    __DIR__ . '/data/sip_accounts.json'
);

$sipController = new SipController(
    $sipRepository
);

try {
    $result = $sipController->storeAccount(
        $_POST
    );

    header(
        'Content-Type: application/json; charset=utf-8'
    );

    echo json_encode(
        $result,
        JSON_UNESCAPED_UNICODE
    );
} catch (\Throwable $e) {
    http_response_code(400);

    header(
        'Content-Type: application/json; charset=utf-8'
    );

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

네트워크 설정값 자체는 `NetworkConfig`가 검증하고, 실제 명령 실행은 `NetworkInterfaceManager`가 담당하도록 나눴습니다.

여기서 `CommandRunnerInterface`를 추가한 이유는 시스템 명령 실행을 별도로 교체할 수 있게 하기 위해서입니다.

예를 들어 테스트에서는 실제 `ifconfig`를 실행하지 않는 Mock Runner를 넣을 수 있습니다.

SIP 역시 요청 배열을 컨트롤러에서 바로 파일에 저장하지 않고 `SipAccount` 객체를 거친 뒤 저장소를 통해 파일에 기록하도록 변경했습니다.

파일 경로도 클래스 내부에 고정하지 않고 생성 시 전달하도록 변경해 테스트 환경과 실제 환경에서 서로 다른 저장 위치를 사용할 수 있게 했습니다.

---

## 4. 에러 처리 및 공통 유틸리티

**핵심 개선 대상:** API 예외 처리, HTTP 응답 형식 통일, 모듈 권한 확인

### 기존 문제점

오류가 발생할 때마다 배열을 직접 만들어 반환하는 방식이라 파일마다 응답 형식이 달라질 수 있었습니다.

또한 권한 확인에 필요한 허용 모듈 목록이 함수 안에 직접 작성되어 있어 정책을 변경하려면 해당 함수를 수정해야 했습니다.

### Before

```php
<?php

class APIError
{
    public function setErrorResult(
        int $code,
        string $message
    ): array {
        return [
            'code' => $code,
            'message' => $message
        ];
    }
}

function can_use_module(
    string $moduleName
): bool {
    $allowed = [
        'audio',
        'network',
        'sip'
    ];

    return in_array(
        $moduleName,
        $allowed
    );
}
```

### After

```php
<?php

declare(strict_types=1);

namespace App\Exceptions;

class ApiException extends \RuntimeException
{
    public function __construct(
        string $message,
        int $statusCode = 400,
        private readonly array $extraData = []
    ) {
        parent::__construct(
            $message,
            $statusCode
        );
    }

    public function getExtraData(): array
    {
        return $this->extraData;
    }
}

final class UnauthorizedException
    extends ApiException
{
    public function __construct(
        string $message = '인증이 필요합니다.'
    ) {
        parent::__construct(
            $message,
            401
        );
    }
}

final class ForbiddenException
    extends ApiException
{
    public function __construct(
        string $message = '접근 권한이 없습니다.'
    ) {
        parent::__construct(
            $message,
            403
        );
    }
}

final class NotFoundException
    extends ApiException
{
    public function __construct(
        string $message = '요청한 리소스를 찾을 수 없습니다.'
    ) {
        parent::__construct(
            $message,
            404
        );
    }
}

final class GlobalExceptionHandler
{
    public function handle(
        \Throwable $exception
    ): void {
        $statusCode = 500;
        $message = '서버 내부 오류가 발생했습니다.';
        $extraData = [];

        if ($exception instanceof ApiException) {
            $statusCode = $exception->getCode();

            if (
                $statusCode < 400 ||
                $statusCode > 599
            ) {
                $statusCode = 400;
            }

            $message = $exception->getMessage();
            $extraData = $exception->getExtraData();
        }

        http_response_code(
            $statusCode
        );

        header(
            'Content-Type: application/json; charset=utf-8'
        );

        echo json_encode(
            array_merge(
                [
                    'success' => false,
                    'code' => $statusCode,
                    'message' => $message
                ],
                $extraData
            ),
            JSON_UNESCAPED_UNICODE
        );
    }
}

namespace App\Auth;

use App\Exceptions\ForbiddenException;

final class ModulePermissionChecker
{
    public function __construct(
        private readonly array $enabledModules
    ) {
    }

    public function isAllowed(
        string $module
    ): bool {
        return in_array(
            $module,
            $this->enabledModules,
            true
        );
    }

    public function assertAllowed(
        string $module
    ): void {
        if (!$this->isAllowed($module)) {
            throw new ForbiddenException(
                "사용할 수 없는 모듈입니다: {$module}"
            );
        }
    }
}

// 사용 예시

use App\Auth\ModulePermissionChecker;
use App\Exceptions\GlobalExceptionHandler;

$exceptionHandler =
    new GlobalExceptionHandler();

set_exception_handler(
    [$exceptionHandler, 'handle']
);

$permissionChecker =
    new ModulePermissionChecker([
        'audio',
        'network',
        'sip'
    ]);

$permissionChecker->assertAllowed(
    'audio'
);

echo json_encode(
    [
        'success' => true,
        'message' => '요청을 처리할 수 있습니다.'
    ],
    JSON_UNESCAPED_UNICODE
);
```

### 변경한 부분

예외를 단순한 배열 데이터가 아니라 실제 객체로 관리하도록 변경했습니다.

`ApiException`을 기준으로 HTTP 상태 코드와 메시지를 함께 전달하고, 실제 HTTP 응답 생성은 `GlobalExceptionHandler`가 담당합니다.

따라서 각 서비스에서는 다음처럼 예외만 발생시키면 됩니다.

```php
throw new \App\Exceptions\ForbiddenException(
    '접근 권한이 없습니다.'
);
```

예외를 발생시킨 코드에서 매번 `http_response_code()`나 `json_encode()`를 호출할 필요가 없어졌습니다.

권한 확인 역시 `ModulePermissionChecker`가 담당하도록 분리했습니다.

허용 모듈 목록은 생성자를 통해 전달하기 때문에 환경이나 테스트 조건에 따라 다른 목록을 사용할 수 있습니다.

---

# 정리

공용 코드 리팩토링에서는 API 기능을 단순히 클래스로 감싸는 것보다 **각 코드가 실제로 어떤 책임을 가지고 있는지 분리하는 것**에 초점을 맞췄습니다.

주요 변경 사항은 다음과 같습니다.

* Digest 인증과 사용자 저장소 분리
* 인증 과정에서 전역 변수 의존성 최소화
* 장비별 DSP 처리 로직을 서비스 객체로 분리
* Gain 계산 로직을 별도 객체로 분리
* 장비 선택을 Factory에서 담당하도록 변경
* 네트워크 설정값을 값 객체로 분리
* 시스템 명령 실행을 별도 인터페이스로 추상화
* SIP 계정 검증과 저장 책임 분리
* API 오류를 예외 객체로 통일
* 전역 예외 핸들러에서 JSON 응답 생성
* 모듈 권한 확인 로직을 별도 객체로 분리

특히 기존 코드에서 보이던 **파일 I/O → 비즈니스 로직 → 시스템 명령 실행 → HTTP 응답**의 경계를 나누는 데 중점을 두었습니다.
