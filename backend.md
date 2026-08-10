# 백엔드 핵심 로직 및 유틸리티 포트폴리오

백엔드 개발 과정에서 처리한 통신, 데이터 관리, 파일 처리, 보안 및 예외 처리 로직을 기능별로 정리했습니다.

각 기능이 담당하는 역할과 처리 흐름을 중심으로 정리했으며, 임베디드 환경에서 발생할 수 있는 상태 동기화와 파일 무결성 문제까지 함께 다루었습니다.

## 1. 시스템 제어 및 버전 호환성 관리

### 개요

프론트엔드와 하드웨어 간의 통신을 중계하고, 컨트롤러와 장치 간의 버전 불일치로 인한 시스템 충돌을 방지하는 백엔드 제어 및 호환성 관리 로직을 구축했습니다.

### 핵심 내용

- **시스템 자원 연동**: TTS 즉시 송출로 생성된 임시 음원을 시스템 영구 음원(BGM)으로 안전하게 복사 및 변환하는 백엔드 프로세스 설계.
- **데이터 정규화**: `state.json`의 데이터 형식을 실시간 통신 상황에 맞춰 동적으로 변환하는 로직 구현.
- **버전 호환성 검사**: `version_compare`와 정규식을 활용하여 컨트롤러 및 장치의 기능 지원 여부를 동적으로 판별하고, 호환되지 않는 버전의 요청을 사전에 차단.
- **중복 작업 방지**: TTS 음원 생성 및 수정 시 파일 중복 여부를 백엔드 레벨에서 확인하여 잘못된 데이터 덮어쓰기를 줄였습니다.

### 실행 가능한 예제

```php
<?php
function isCompatibleVersion(string $current, string $supported): bool
{
    $current = preg_replace('/[^0-9.]/', '', $current);
    $supported = preg_replace('/[^0-9.]/', '', $supported);

    return $current !== '' && $supported !== ''
        && version_compare($current, $supported, '>=');
}

echo isCompatibleVersion('2.1.0-build12', '2.0.5') ? "supported\n" : "unsupported\n";
```

---

## 2. 데이터 무결성 관리

### 개요

임베디드 리눅스 환경에서 설정값의 안정적인 저장과 동기화를 보장하는 로직을 개발했습니다.

### 핵심 내용

- **물리적 기록 보장**: 설정값 변경 후 `shell_exec("sync")`를 명시적으로 호출하여 비정상 종료 시 데이터 손실 가능성을 줄였습니다.
- **SQLite 동적 관리**: 가상 테이블을 활용한 실시간 상태값 저장소 설계 및 쿼리 최적화.
- **불필요 로직 제거**: 기존 시스템의 중복된 오디오 출력 체크 로직을 제거하여 API 응답 속도 및 리소스 효율성을 개선했습니다.

### 실행 가능한 예제

```php
<?php
if (!in_array('sqlite', PDO::getAvailableDrivers(), true)) {
    exit("SQLite PDO driver is required for this example.\n");
}

$db = new PDO('sqlite::memory:');
$db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$db->exec('CREATE TABLE settings (key TEXT PRIMARY KEY, value TEXT)');

try {
    $db->beginTransaction();

    $stmt = $db->prepare(
        'INSERT INTO settings(key, value) VALUES(:key, :value)'
    );
    $stmt->execute([':key' => 'volume', ':value' => '80']);

    $db->commit();
    echo "saved\n";
} catch (Throwable $e) {
    if ($db->inTransaction()) {
        $db->rollBack();
    }
    echo "save failed: {$e->getMessage()}\n";
}
```

---

## 3. 비동기 중복 방지 시스템

### 개요

네트워크 지연이나 사용자 중복 클릭으로 인해 발생할 수 있는 데이터 오염을 방지하기 위해 타임스탬프 기반의 비동기 토큰 검증 시스템을 도입했습니다.

### 핵심 내용

- **Race Condition 방지**: TTS 음원 생성과 같은 긴 시간이 소요되는 비동기 작업 시 최신 요청 토큰만 유효하게 처리하여 구형 요청에 의한 데이터 덮어쓰기를 차단.
- **안전한 파일 삭제**: 작업이 취소되거나 토큰이 불일치할 경우 서버에 임시 생성된 TTS 파일을 즉시 삭제하는 정리 로직 연동.

### 실행 가능한 예제

```php
<?php
function finishTask(string $token, string $latestToken, string $tempFile): bool
{
    if ($token !== $latestToken) {
        if (is_file($tempFile)) {
            unlink($tempFile);
        }
        return false;
    }

    file_put_contents(__DIR__ . '/task_result.json', json_encode(
        ['temp_file' => $tempFile],
        JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE
    ));
    return true;
}

$tempFile = __DIR__ . '/temp_result.txt';
file_put_contents($tempFile, 'temporary result');

echo finishTask('task-002', 'task-002', $tempFile) ? "accepted\n" : "discarded\n";
```

---

## 4. 멀티채널 대응 아키텍처

### 개요

단일 채널 장비부터 4채널 이상의 장비까지 동일한 코드로 대응할 수 있는 동적 데이터베이스 참조 시스템을 설계했습니다.

### 핵심 내용

- **동적 테이블 매핑**: 장치 설정(`device-config.json`)을 분석하여 채널별 저장소 테이블을 자동으로 식별하는 쿼리 생성.
- **SQLite 가상 연산**: 복잡한 비즈니스 로직을 PHP가 아닌 DB 레벨에서 처리하기 위해 서브쿼리와 가상 컬럼을 활용하여 쿼리와 데이터 처리를 단순화했습니다.

### 실행 가능한 예제

```php
<?php
if (!in_array('sqlite', PDO::getAvailableDrivers(), true)) {
    exit("SQLite PDO driver is required for this example.\n");
}

$db = new PDO('sqlite::memory:');
$db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$db->exec('CREATE TABLE channel_status (
    channel INTEGER PRIMARY KEY,
    is_run INTEGER NOT NULL,
    is_play INTEGER NOT NULL
)');

$stmt = $db->prepare(
    'INSERT INTO channel_status(channel, is_run, is_play) VALUES(?, ?, ?)'
);

for ($channel = 1; $channel <= 4; $channel++) {
    $stmt->execute([$channel, $channel === 1 ? 1 : 0, $channel === 1 ? 1 : 0]);
}

$rows = $db->query('SELECT channel, is_run, is_play FROM channel_status ORDER BY channel')
            ->fetchAll(PDO::FETCH_ASSOC);

print_r($rows);
```

---

## 5. 기본적인 보안 및 세션 관리

### 개요

외부 공격으로부터 시스템을 보호하고 실시간 세션 정합성을 유지하기 위해 보안 필터링 시스템을 구축했습니다.

### 핵심 내용

- **Replay Attack 방어**: `Nonce`와 타임스탬프 기반의 티켓 검증 로직을 도입하여 동일한 패킷을 재전송하는 공격을 방지.
- **세션 정합성 보장**: `session_write_close`를 명시적으로 사용하여 세션 데이터를 파일 시스템에 즉시 동기화하고 동시성 문제를 줄였습니다.

### 실행 가능한 예제

```php
<?php
session_start();

$_SESSION['used_nonces'] ??= [];

$nonce = $_GET['nonce'] ?? bin2hex(random_bytes(8));

if (in_array($nonce, $_SESSION['used_nonces'], true)) {
    http_response_code(409);
    exit("replay request\n");
}

$_SESSION['used_nonces'][] = $nonce;
$_SESSION['used_nonces'] = array_slice($_SESSION['used_nonces'], -10);

session_write_close();

echo "request accepted: {$nonce}\n";
```

---

## 6. API 보안 및 장치 검증 자동화

### 개요

장치 간 통신 시 발생할 수 있는 무단 접근을 방지하기 위해 통신 전 단계에서 장치 등록 여부와 API 키 유효성을 자동으로 검사하는 보안 모듈을 설계했습니다.

### 핵심 내용

- **장치 신뢰성 검증**: `is_regist_device` 메서드를 통해 Target IP의 관리 서버 등록 상태를 조회하고, 통신 실패 시 세부 에러 코드(101~103)를 반환하여 문제 원인 명확화.
- **권한 계층화**: `canUseAPI` 로직을 통해 401(인증 실패), 202(키 권한 부족) 등 HTTP 상태 코드별로 세분화된 보안 예외 처리를 수행.

### 예제

```php
<?php
function validateApiResponse(?array $response): array
{
    if ($response === null) {
        return ['ok' => false, 'code' => 503, 'error' => 'connect_fail'];
    }

    $status = (int)($response['status'] ?? 0);

    if ($status === 401) {
        return ['ok' => false, 'code' => 401, 'error' => 'auth_fail'];
    }

    if ($status !== 200) {
        return ['ok' => false, 'code' => $status ?: 500, 'error' => 'request_fail'];
    }

    return ['ok' => true, 'code' => 200];
}

print_r(validateApiResponse(['status' => 200]));
```

---

## 7. 계층형 저장소 정보 수집

### 개요

내부 메모리와 외부 SD 카드가 혼용되는 임베디드 환경에서, 통합된 저장소 정보를 제공하기 위한 동적 데이터 수집 아키텍처를 구성했습니다.

### 핵심 내용

- **구조적 데이터 반환**: 기존의 단순 문자열 반환 방식을 탈피하여, `internal`, `external`, `mount_status`가 포함된 구조화된 JSON 데이터를 생성하여 프론트엔드 제어 편의성 개선.
- **실시간 용량 추적**: API 요청 시점에 시스템 레벨의 `df` 정보를 실시간 파싱하여 요청 시점의 가용 용량 데이터를 보장.

### 예제

```php
<?php
function getStorageInfo(string $path): array
{
    if (!is_dir($path)) {
        return ['path' => $path, 'exists' => false];
    }

    return [
        'path' => $path,
        'exists' => true,
        'total_bytes' => disk_total_space($path),
        'free_bytes' => disk_free_space($path),
    ];
}

$result = [
    'internal' => getStorageInfo(__DIR__),
    'external' => getStorageInfo(__DIR__ . '/storage'),
];

echo json_encode($result, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
```

---

## 8. 자원 중복 방지 및 파일 무결성

### 개요

시스템 내 동일한 이름의 음원 파일이 중복 생성되어 설정이 꼬이는 문제를 방지하기 위해 서버 측 선검증 로직을 강화했습니다.

### 핵심 내용

- **사전 이름 충돌 검사**: 파일 복사(파일 복사 작업) 작업 전 대상 디렉토리의 파일 존재 여부를 전체 확인하여 "duplicate" 상태와 충돌 파일 리스트를 구조적으로 반환.
- **원자적 작업 처리**: 복사 실패나 중복 발생 시 임시 생성된 파일들을 즉시 물리적으로 삭제하여 저장 공간 낭비 차단.

### 예제

```php
<?php
function findDuplicateFiles(string $directory, array $names): array
{
    $duplicates = [];

    foreach ($names as $name) {
        $path = $directory . DIRECTORY_SEPARATOR . basename($name);
        if (is_file($path)) {
            $duplicates[] = basename($name);
        }
    }

    return $duplicates;
}

$directory = __DIR__ . '/audio';
if (!is_dir($directory)) {
    mkdir($directory, 0777, true);
}
file_put_contents($directory . '/sample.wav', 'example');

print_r(findDuplicateFiles($directory, ['sample.wav', 'new.wav']));
```

---

## 9. 동적 설정 오케스트레이션

### 개요

사용자의 UI 조작(정렬, 필터링 등) 결과를 시스템 설정 파일(`json`)에 즉시 반영하고 실시간으로 재색인하는 로직을 구성했습니다.

### 핵심 내용

- **설정 재색인 알고리즘**: 전달받은 정렬 순서 리스트를 기반으로 기존 JSON 배열을 재구성하고, `array_values`를 통해 인덱스를 재정렬하여 데이터 일관성 유지.

### 예제

```php
<?php
$config = [
    'items' => [
        ['file' => 'B.wav', 'title' => 'B'],
        ['file' => 'A.wav', 'title' => 'A'],
        ['file' => 'C.wav', 'title' => 'C'],
    ],
];

$order = ['A.wav', 'C.wav', 'B.wav'];

$indexed = [];
foreach ($config['items'] as $item) {
    $indexed[$item['file']] = $item;
}

$config['items'] = [];
foreach ($order as $file) {
    if (isset($indexed[$file])) {
        $config['items'][] = $indexed[$file];
    }
}

file_put_contents(
    __DIR__ . '/config.json',
    json_encode($config, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE)
);

print_r($config['items']);
```

---

## 10. 임시 자원 생명주기 관리

### 개요

TTS 즉시 송출을 위해 생성된 임시 음원 파일이 시스템 자원을 점유하지 않도록, 방송 종료 시점에 자동으로 파기하는 자동 정리 로직을 구성했습니다.

### 핵심 내용

- **유지 여부 플래그(`is_keep`)**: 사용자가 명시적으로 저장(Keep)을 요청하지 않은 임시 파일에 대해 방송 완료 즉시 물리적 삭제(`rm`)를 수행하여 불필요한 저장 공간 사용을 줄였습니다.
- **세션 기반 상태 추적**: 현재 방송 중인 음원이 임시 파일인지 여부를 서버 세션 및 JSON 상태 파일에 기록하여 프로세스 재시작 시에도 무결성 유지.

### 예제

```php
<?php
$state = [
    'keep' => false,
    'temp_file' => __DIR__ . '/temporary_audio.tmp',
];

file_put_contents($state['temp_file'], 'temporary data');

if ($state['keep'] === false && is_file($state['temp_file'])) {
    unlink($state['temp_file']);
}

$state['temp_file'] = null;
echo "temporary resource cleaned
";
```

---

## 11. JSON 기반 멀티 인스턴스 동기화

### 개요

여러 명의 관리자가 동시에 접속한 환경에서 방송 상태와 설정값이 모든 클라이언트에 동일하게 보이도록 중앙 집중형 상태 관리 엔진을 구현했습니다.

### 핵심 내용

- **실시간 상태 전파**: 각 채널별(`m1`~`m4`) 독립된 오디오 서버 상태를 JSON 파일로 중앙 집중화하고, 변경 발생 시 웹소켓 핸들러를 통해 모든 인스턴스에 즉시 전파.
- **원자적 파일 조작**: `LOCK_EX`를 활용한 파일 쓰기 제어로 여러 프로세스가 동시에 상태를 갱신할 때 발생할 수 있는 잘못된 데이터 덮어쓰기를 줄였습니다.

### 예제

```php
<?php
$path = __DIR__ . '/state.json';

$state = is_file($path)
    ? json_decode(file_get_contents($path), true)
    : [];

$state['current'] = [
    'id' => 101,
    'keep' => true,
    'updated_at' => date(DATE_ATOM),
];

file_put_contents(
    $path,
    json_encode($state, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE),
    LOCK_EX
);

echo file_get_contents($path);
```

---

## 12. 교차 컨트롤러 자원 검증 엔진

### 개요

동일한 음원 자원이 여러 컨트롤러의 스케줄러에 등록되어 사용 중일 경우, 실수로 삭제되는 것을 방지하기 위해 시스템 전체의 스케줄을 전체 확인하는 검증 엔진을 구성했습니다.

### 핵심 내용

- **전역 참조 무결성**: 단일 컨트롤러 체크 방식에서 벗어나, 현재 장치에 연결된 모든 컨트롤러(`m1`~`m4`)의 스케줄 DB를 순회하며 해당 파일의 사용 여부를 판별.
- **재귀적 의존성 체크**: 로컬 스케줄러와 원격 컨트롤러 스케줄러의 의존성을 통합 분석하여 데이터 안정성 개선.

### 예제

```php
<?php
function isResourceInUse(PDO $db, string $filename): bool
{
    $stmt = $db->prepare(
        'SELECT COUNT(*) FROM schedule_items WHERE source_name = :name'
    );
    $stmt->execute([':name' => $filename]);

    return (int)$stmt->fetchColumn() > 0;
}

if (!in_array('sqlite', PDO::getAvailableDrivers(), true)) {
    exit("SQLite PDO driver is required for this example.\n");
}

$db = new PDO('sqlite::memory:');
$db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$db->exec('CREATE TABLE schedule_items (source_name TEXT)');
$db->exec("INSERT INTO schedule_items VALUES ('sample.wav')");

var_export(isResourceInUse($db, 'sample.wav'));
```

---

## 13. 대용량 파일 업로드 대응

### 개요

TTS 음원이나 대용량 로그 파일을 시스템 간 전송할 때 데이터 유실을 방지하기 위해 `CURL`과 `multipart/form-data`를 통합한 효율적인 통신 모듈을 구성했습니다.

### 핵심 내용

- **CURL 파일 전송 최적화**: 바이너리 데이터를 안전하게 전송하기 위해 `CURLFile` 객체를 활용하고, 수신 측에서 `$_FILES`와 `php://input`을 유연하게 처리하도록 로직 개선.
- **스트리밍 데이터 처리**: 메모리 부하를 줄이기 위해 파일 데이터를 직접 스트림으로 전달하는 통신 파이프라인 구현.

### 예제

```php
<?php
$filePath = __DIR__ . '/sample.txt';
file_put_contents($filePath, "multipart example
");

if (!class_exists('CURLFile')) {
    exit("cURL extension is required
");
}

$upload = new CURLFile($filePath, 'text/plain', basename($filePath));

$request = [
    'type' => 'file_upload',
    'file' => $upload,
];

echo "multipart payload prepared
";
echo "file: " . $request['file']->getPostFilename() . "
";
```

---

## 14. 교차 모듈 자원 소유권 관리

### 개요

특정 음원이 '삭제 가능'한 상태인지 판단하기 위해, 소스 관리 모듈 뿐만 아니라 버튼 설정, 스케줄러, EM 등 전역 모듈의 의존성을 역추적하는 관계형 검증 시스템을 구성했습니다.

### 핵심 내용

- **통합 자원 잠금**: 타 모듈(예: Scheduler)에서 사용 중인 음원에 대해 "삭제 불가" 플래그를 실시간으로 연동하여 데이터 정합성 파괴 방지.
- **분산 DB 순회**: 각기 다른 파일로 존재하는 SQLite 데이터베이스들을 순차적으로 열어 음원 ID를 검색하는 재귀적 쿼리 엔진 구현.

### 예제

```php
<?php
function canDelete(string $filename, array $dependencies): bool
{
    foreach ($dependencies as $items) {
        if (in_array($filename, $items, true)) {
            return false;
        }
    }

    return true;
}

$dependencies = [
    'preset' => ['notice.wav'],
    'schedule' => ['sample.wav', 'notice.wav'],
    'playlist' => [],
];

var_export(canDelete('notice.wav', $dependencies));
```

---

## 15. 임시 자원 원자적 관리

### 개요

TTS 즉시 송출과 같이 생명주기가 짧은 임시 자원의 생성부터 자동 파기까지의 전 과정을 일관되게 관리하는 트랜잭션과 유사한 로직을 설계했습니다.

### 핵심 내용

- **원자적 클린업**: 방송 실패나 중도 취소 시, 생성 중이던 임시 파일을 즉시 물리적으로 삭제하고 시스템 인덱스를 롤백하여 가용 용량 보존.
- **배타적 쓰기 잠금**: `LOCK_EX`와 `sync` 명령어를 조합하여 파일 시스템 수준의 데이터 안정성 개선.

### 예제

```php
<?php
function createTemporaryFile(string $content): ?string
{
    $path = tempnam(sys_get_temp_dir(), 'audio_');

    try {
        if ($path === false || file_put_contents($path, $content) === false) {
            throw new RuntimeException('temporary file creation failed');
        }

        return $path;
    } catch (Throwable $e) {
        if ($path !== false && is_file($path)) {
            unlink($path);
        }
        return null;
    }
}

$path = createTemporaryFile('temporary resource');
echo $path ? "created: {$path}
" : "failed
";

if ($path && is_file($path)) {
    unlink($path);
}
```

---

## 16. 효율적인 상태 폴링 및 동기화

### 개요

여러 사용자가 동시에 접속하는 상황에서도 시스템 부하를 최소화하면서 최신 상태를 유지하기 위해, 데이터 변경 시점만을 감지하여 전송하는 시퀀스 기반 폴링 시스템을 설계했습니다.

### 핵심 내용

- **델타 업데이트(Delta Update)**: 전체 데이터를 매번 전송하는 대신 `STATE_SEQ` 값이 변경된 경우에만 상세 데이터를 요청하도록 설계하여 네트워크 트래픽 불필요한 전송을 줄였습니다.
- **시퀀스 정합성 보장**: 서버 측 시퀀스 번호와 클라이언트의 번호를 대조하여 브라우저의 불필요한 DOM 조작을 차단, 렌더링 쿼리와 데이터 처리를 단순화했습니다.

### 예제

```php
<?php
function getSequence(PDO $db): int
{
    $stmt = $db->query(
        "SELECT value FROM sequence_table WHERE name = 'state'"
    );

    return (int)($stmt->fetchColumn() ?: 0);
}

if (!in_array('sqlite', PDO::getAvailableDrivers(), true)) {
    exit("SQLite PDO driver is required for this example.\n");
}

$db = new PDO('sqlite::memory:');
$db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$db->exec('CREATE TABLE sequence_table (name TEXT PRIMARY KEY, value INTEGER)');
$db->exec("INSERT INTO sequence_table VALUES ('state', 3)");

$previous = 2;
$current = getSequence($db);

echo $previous !== $current ? "state changed
" : "no change
";
```

---

## 17. 추상화된 오디오 서버 데이터 파이프라인

### 개요

다양한 종류의 오디오 서버(MP3, TTS, CDP 등)에서 발생하는 데이터를 단일한 규격으로 통합하여 프론트엔드로 전달하는 추상화된 데이터 파이프라인을 구성했습니다.

### 핵심 내용

- **데이터 정규화**: 각기 다른 포맷의 소스 정보를 `prefix`, `cmd_id`, `data_idx` 필드를 포함한 통합 JSON 객체로 변환하여 전송.
- **바이너리 데이터 라우팅**: 레벨 미터와 같은 실시간 스트리밍 데이터를 바이너리 패킷으로 캡슐화하여 전송 속도 개선 및 지연 시간 단축.

### 예제

```php
<?php
function buildPayload(array $source): array
{
    return [
        'type' => 'meter',
        'ip' => $source['ip'],
        'channel' => $source['channel'],
        'level' => $source['level'],
    ];
}

$source = [
    'ip' => '127.0.0.1',
    'channel' => 1,
    'level' => -12.5,
];

echo json_encode(buildPayload($source), JSON_PRETTY_PRINT);
```

---

## 18. 세션 및 JSON 하이브리드 보존

### 개요

사용자가 마지막으로 선택한 소스 장치나 방송 존의 설정을 페이지 새로고침 후에도 유지하기 위해 세션(휘발성)과 JSON(영구성)을 혼합한 상태 보존 레이어를 구성했습니다.

### 핵심 내용

- **세션 기반 실시간 보존**: `$_SESSION`에 현재 선택된 `source`와 `zone` ID를 즉시 기록하여 탭 이동 시에도 상태 유지.
- **동적 로드 인터페이스**: `LoadSelectSource` 함수를 통해 서버에 저장된 마지막 세션 정보를 조회하고, 해당 요소에 대해 `trigger("click")`을 발생시켜 자동 복원.

### 예제

```php
<?php
session_start();

if (isset($_POST['source'])) {
    $_SESSION['source'] = (string)$_POST['source'];
}

$result = ['source' => $_SESSION['source'] ?? ''];
session_write_close();

header('Content-Type: application/json; charset=utf-8');
echo json_encode($result, JSON_UNESCAPED_UNICODE);
```

---

## 19. 장치 간 API 프록시 및 오케스트레이션

### 개요

컨트롤러 장치에서 원격지에 위치한 소스 장치(Source Device)의 자원을 직접 제어하기 위해, CURL을 활용한 API 릴레이 및 오케스트레이션 엔진을 구성했습니다.

### 핵심 내용

- **X- 보안 헤더 통신**: 원격 장치 인증을 위해 `X-Device-ID`와 `Secret` 헤더를 동적으로 생성하여 전달하는 보안 프록시 구현.
- **멀티파트 스트림 중계**: 클라이언트로부터 수신된 `$_FILES` 데이터를 메모리 낭비 없이 `CURLFile`을 통해 원격 장치로 스트리밍 전송하는 업로드 파이프라인을 구성.

### 예제

```php
<?php
function buildProxyHeaders(string $deviceId, string $secret): array
{
    return [
        'X-Device-ID: ' . $deviceId,
        'X-Device-Secret: ' . $secret,
    ];
}

$headers = buildProxyHeaders('device-001', 'example-secret');

print_r($headers);
// 실제 전송 시 curl_setopt($ch, CURLOPT_HTTPHEADER, $headers)를 사용합니다.
```

---

## 20. 쉘 연동 음성 합성 파이프라인

### 개요

기본적인 PHP 코드와 저수준 시스템 바이너리를 유기적으로 결합하여, 텍스트 입력부터 차임벨 믹싱까지의 전 과정을 자동화한 TTS 엔진을 구성했습니다.

### 핵심 내용

- **시스템 명령어 정밀 오케스트레이션**: `LD_LIBRARY_PATH` 설정, VTML 태그 동적 주입, 바이너리 실행(`tts_engine`), 쉘 스크립트 연동(`merge_audio.sh`)을 PHP 상에서 하나의 시퀀스로 제어.
- **비동기 미디어 유효성 검증**: `avprobe`를 활용하여 생성된 음원의 재생 시간(Duration)을 밀리초 단위로 파싱하여 클라이언트에 제공.

### 예제

```php
<?php
function runTtsCommand(string $text, string $output): bool
{
    $binary = trim((string)shell_exec('command -v ffmpeg 2>/dev/null'));

    if ($binary === '') {
        file_put_contents($output, "TTS placeholder: {$text}
");
        return true;
    }

    // 실제 TTS 엔진 대신 외부 명령 실행 구조만 보여주는 예제입니다.
    return true;
}

$output = __DIR__ . '/tts_result.txt';
runTtsCommand('Hello backend', $output);

echo file_get_contents($output);
unlink($output);
```

---

## 21. 하이브리드 자원 오케스트레이션

### 개요

내장 스토리지와 외장 SD 카드의 자원을 구분 없이 통합 관리하고, 두 경로의 차임벨(Chime)을 자유롭게 믹싱할 수 있는 가상 경로 시스템을 설계했습니다.

### 핵심 내용

- **가상 경로 매핑**: `(ex)` 태그 여부에 따라 시스템 절대 경로와 마운트된 SD 카드 경로를 동적으로 스위칭하여 자원 접근성 개선.

### 예제

```php
<?php
function resolveResourcePath(
    string $name,
    string $internalDir,
    string $externalDir
): string {
    if (str_starts_with($name, 'external/')) {
        return $externalDir . '/' . substr($name, strlen('external/'));
    }

    return $internalDir . '/' . $name;
}

echo resolveResourcePath(
    'external/chime.wav',
    __DIR__ . '/audio',
    __DIR__ . '/storage'
) . PHP_EOL;
```

---

## 22. 결함 허용 미디어 트랜스코딩 파이프라인

### 개요

이기종 오디오 플레이어 간의 호환성을 보장하기 위해, 업로드된 다양한 형식의 음원을 시스템 표준 규격(44.1kHz, Mono, S16LE PCM)으로 자동 변환하는 안정적인 데이터 파이프라인을 구성했습니다.

### 핵심 내용

- **이중 트랜스코딩 시퀀스**: `avconv`를 활용하여 샘플링 레이트와 비트 레이트를 동시에 조정하고, 변환 완료 전까지 임시 경로에서 처리함으로써 잘못된 데이터 덮어쓰기를 줄였습니다.
- **물리적 동기화 강제**: 변환 및 복사 작업 완료 후 `shell_exec("sync")`를 명시적으로 호출하여 비정상 종료 시에도 파일 시스템 무결성 보장.

### 예제

```php
<?php
function normalizeAudio(string $source, string $destination): bool
{
    if (!is_file($source)) {
        return false;
    }

    $ffmpeg = trim((string)shell_exec('command -v ffmpeg 2>/dev/null'));

    if ($ffmpeg === '') {
        // 실행 환경에 ffmpeg가 없으면 예제에서는 원본을 복사합니다.
        return copy($source, $destination);
    }

    $command = sprintf(
        '%s -y -i %s -ar 44100 -ac 1 -c:a pcm_s16le %s 2>&1',
        escapeshellcmd($ffmpeg),
        escapeshellarg($source),
        escapeshellarg($destination)
    );

    exec($command, $output, $code);
    return $code === 0;
}

$source = __DIR__ . '/input.wav';
$destination = __DIR__ . '/normalized.wav';
file_put_contents($source, 'sample');

var_export(normalizeAudio($source, $destination));

@unlink($source);
@unlink($destination);
```

---

## 23. 패턴 기반 아티팩트 자동 정리 엔진

### 개요

대량의 TTS 음원 생성 및 삭제 과정에서 발생하는 파생 캐시 파일(`.pcm`)들을 일관되게 식별하여 시스템 자원을 효율적으로 관리하는 정리 엔진을 구현했습니다.

### 핵심 내용

- **정규식 기반 대량 삭제**: `glob`과 대괄호 이스케이프(`\\]`, `\\[`)를 활용하여 특정 음원과 연결된 모든 캐시 파일 패턴을 정확히 타겟팅하여 일괄 파기.

### 예제

```php
<?php
function cleanupArtifacts(string $directory, string $baseName): int
{
    $count = 0;
    foreach (glob($directory . '/' . $baseName . '_*.tmp') ?: [] as $file) {
        if (is_file($file) && unlink($file)) {
            $count++;
        }
    }
    return $count;
}

$dir = __DIR__ . '/cache';
mkdir($dir, 0777, true);
file_put_contents($dir . '/audio_001.tmp', 'cache');
file_put_contents($dir . '/audio_002.tmp', 'cache');

echo cleanupArtifacts($dir, 'audio') . " files removed
";
@rmdir($dir);
```

---

## 24. 상태 비보존형 재생 복구 로직

### 개요

시스템 리부팅이나 네트워크 재연결 시에도 이전의 재생 시퀀스를 안전하게 복구하기 위해, JSON 파일 기반의 글로벌 재생 정보를 활용한 자동 복구 시스템을 구현했습니다.

### 핵심 내용

- **글로벌 상태 복원**: `get_playback_info` API를 통해 서버에 저장된 마지막 성공 음원 정보를 조회하고, 클라이언트 로드 완료 시점에 해당 정보를 주입하여 즉시 방송이 가능하도록 설계.

### 예제

```php
<?php
$path = __DIR__ . '/playback.json';

$data = [
    'source' => 'sample.wav',
    'position' => 12.5,
    'updated_at' => date(DATE_ATOM),
];

file_put_contents(
    $path,
    json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE),
    LOCK_EX
);

$restored = json_decode(file_get_contents($path), true);
echo "resume source: {$restored['source']}, position: {$restored['position']}s
";

unlink($path);
```

---

## 25. 범용 API 페이로드 협상가

### 개요

기존의 JSON 전용 API 구조를 확장하여, 일반 데이터와 대용량 바이너리 파일(Multipart)을 동시에 수용할 수 있는 단순한 요청 처리 레이어를 구성했습니다.

### 핵심 내용

- **컨텐츠 타입 자동 감지**: `CONTENT_TYPE` 헤더를 분석하여 `application/json`과 `multipart/form-data`를 구분하여 판별하고, 각각의 형식에 최적화된 파싱 로직을 실행.
- **하이브리드 통신 지원**: 단일 엔드포인트에서 단순 설정 변경 요청과 파일 업로드 요청을 모두 처리할 수 있도록 REST 통신 규격 고도화.

### 예제

```php
<?php
function parseRequest(string $contentType, string $body): array
{
    if (str_contains($contentType, 'application/json')) {
        return json_decode($body, true) ?: [];
    }

    if (str_contains($contentType, 'multipart/form-data')) {
        return ['type' => 'multipart', 'body_received' => true];
    }

    return ['error' => 'unsupported_content_type'];
}

print_r(parseRequest('application/json', '{"action":"save","value":10}'));
```

---

## 26. 멀티 모델 UI 상태 동기화

### 개요

단일 채널 스피커부터 멀티 채널 컨트롤러까지, 서로 다른 모델 간의 UI 일관성을 보장하기 위해 세션 기반의 상태 공유 아키텍처를 구현했습니다.

### 핵심 내용

- **프로젝트별 상태 격리**: `$_SESSION['project_name']`을 활용하여 접속한 장치 모델에 최적화된 초기 UI 세팅을 자동으로 로드.
- **실시간 상태 보존**: 페이지 새로고침 시에도 `zone_lock`이나 `selected_channel` 정보를 서버 세션에서 즉시 복원하여 사용자 조작 연속성 확보.

### 예제

```php
<?php
session_start();

$_SESSION['model'] = $_SESSION['model'] ?? 'standard';
$_SESSION['selected_channel'] = $_SESSION['selected_channel'] ?? 1;
$_SESSION['zone_lock'] = $_SESSION['zone_lock'] ?? true;

echo json_encode([
    'model' => $_SESSION['model'],
    'selected_channel' => $_SESSION['selected_channel'],
    'zone_lock' => $_SESSION['zone_lock'],
], JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);

session_write_close();
```

---

## 27. 통합 REST API 인증 미들웨어

### 개요

여러 API 요청을 처리할 때 발생할 수 있는 보안 취약점을 해결하기 위해, 모든 엔드포인트에 공통으로 적용되는 중앙 집중형 인증 미들웨어를 구성했습니다.

### 핵심 내용

- **헤더 기반 보안 필터링**: `X-Forwarded-Proto`와 같은 프록시 헤더를 감지하여 프로토콜의 정합성을 검증하고, 비인가 접근을 방지하는 전역 보안 레이어 구현.

### 예제

```php
<?php
function isSecureRequest(array $server): bool
{
    $proto = $server['HTTP_X_FORWARDED_PROTO'] ?? $server['REQUEST_SCHEME'] ?? 'http';
    return strtolower($proto) === 'https';
}

$server = [
    'HTTP_X_FORWARDED_PROTO' => 'https',
];

echo isSecureRequest($server) ? "secure request
" : "insecure request
";
```

---

## 28. 적응형 템플릿 아키텍처

### 개요

비즈니스 로직과 UI 스타일을 엄격히 분리하고, 장치 설정에 따라 HTML 구조를 동적으로 재구성하는 단순한 템플릿 엔진을 설계했습니다.

### 핵심 내용

- **구조-스타일 분리**: `div_contents_right_contents`와 같은 래퍼 클래스를 도입하여, 백엔드 데이터 변경 없이 CSS만으로 다국어 레이아웃 대응이 가능한 구조 완성.

### 예제

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <title>Simple Template</title>
  <style>
    .row { display: flex; align-items: center; gap: 12px; }
    .actions { margin-left: auto; }
  </style>
</head>
<body>
  <div class="row">
    <strong>Source Name</strong>
    <div class="actions">
      <button type="button" onclick="applySetting()">Apply</button>
    </div>
  </div>

  <script>
    function applySetting() {
      alert('설정을 적용했습니다.');
    }
  </script>
</body>
</html>
```

---

## 29. 디버깅 코드 최적화 및 유지보수

### 개요

개발 중 포함된 불필요한 PHP 디버깅 출력 코드와 오타를 정리하여 운영
          환경의 보안과 코드 가독성을 개선했습니다.

### 핵심 내용

- **디버깅 코드 정리**: 관리 기능 코드 내에 포함된 전역 PHP 에러
  리포팅 (`error_reporting(E_ALL)`) 코드를 제거하여 배포 환경의 보안 안정성
  확보.
- **코드 무결성 강화**: 이벤트 처리 코드 내 미세한 오타를 수정하여 캘린더
  컴포넌트의 렌더링 안정성 향상.

### 예제

```php
<?php
// 개발 환경에서는 오류를 확인하고, 운영 환경에서는 화면에 오류를 노출하지 않습니다.
$isProduction = false;

if ($isProduction) {
    error_reporting(0);
    ini_set('display_errors', '0');
} else {
    error_reporting(E_ALL);
    ini_set('display_errors', '1');
}

echo "debug configuration applied
";
```

---

## 30. 서버 로직 최적화 및 보안

### 개요

운영 환경에서의 데이터 일관성과 보안성을 강화하기 위해 서버측 제어 로직을 최적화하고, 불필요한 디버깅 코드를 안정적으로 제거했습니다.

### 핵심 내용

- **보안/유지보수 강화**: 배포 환경의 로그 무결성을 위해 서버단 디버깅용 PHP 에러 리포팅을 제거하고, 코드 위치 재조정으로 유지보수 효율성 향상.
- **이스케이프 처리 고도화**: 다채널 시스템의 장치명(호스트명) 처리에 있어 멀티 채널 데이터 이스케이프 버그를 해결하여 XSS 보안성 강화.
- **TTS 워크플로우 최적화**: 즉시 송출 기능의 예외 처리 로직(제목 입력 필수 제거 등)을 개선하고, 자원 삭제와 연동된 원자적(atomic) 명령 처리 구조 설계.
- **모듈 권한 제어 엔진**: 모듈의 존재 여부와 권한(Auth)을 런타임에 체크하여 클라이언트별로 차별화된 기능 노출 로직 구현.

### 예제

```php
<?php
function escapeHtml(string $text): string
{
    return htmlspecialchars($text, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

$input = '<script>alert("test")</script>';
echo escapeHtml($input) . PHP_EOL;
```

---

## 31. 시스템 안정성 및 리소스 제어

### 개요

임베디드 장치의 시스템 점검 주기 설정 로직을 완성하고, 그룹 관리 및 음원 자원의 안전한 삭제를 위한 백엔드 프로세스를 개선했습니다.

### 핵심 내용

- **시스템 점검 자동화**: 모델별 시스템 점검 주기(Daily/Weekly) 설정 로직을 각각 분리하여 구현하고, 크론탭(`crontab`) 동적 제어를 통해 안정적인 자동 점검/재시작 체계 구축.
- **자원 무결성 보존**: 그룹 등록/삭제 시 로그에 그룹 명칭을 포함하여 추적성을 강화하고, TTS 즉시 송출 음원 생성 전 BGM 목록을 사전에 조회하여 중복 및 충돌을 방지.
- **운영 안정성 유지**: 음원 재생 종료 시점의 삭제 요청 전 리로드(RELOAD) 및 상태 초기화 로직을 통해 타 채널과의 간섭 문제를 해결(r7162 등과의 연계 최적화).
- **데이터 보안**: 그룹 등록/삭제 로그 시 그룹 명칭을 인코딩하여 출력함으로써 로그 파일의 텍스트 깨짐 현상을 방지.

### 예제

```php
<?php
function buildCronEntry(string $schedule, string $command): string
{
    return match ($schedule) {
        'daily' => "0 3 * * * {$command}",
        'weekly' => "0 3 * * 0 {$command}",
        default => throw new InvalidArgumentException('unsupported schedule'),
    };
}

echo buildCronEntry('daily', '/usr/bin/php ' . __DIR__ . '/system_check.php') . PHP_EOL;
```

---

## 32. 모듈 간 통신 및 모델별 호환성 처리

### 개요

컨트롤러와 TTS 관리 팝업 간의 파라미터 전달 방식을 최적화하고, 기기별 기능 차이에 따른 동작 분기 로직을 강화했습니다.

### 핵심 내용

- **아키텍처 유연성 강화**: `GET` 파라미터 방식을 지양하고 `window.opener`를 통한 객체 참조 방식으로 전환하여, 브라우저 환경에 종속되지 않는 안정적인 데이터 전달체계 확보.
- **기기별 동작 분기**:  브랜드 모델의 경우, 음원 재생 종료 시 발생할 수 있는 불필요한 자동 삭제 로직을 조건부로 제한하여 기기 특성에 맞춘 안정성 개선.
- **데이터 처리 원자성**: TTS 즉시 송출 기능(ToBGM) 시, 생성된 음원 파일과 BGM 목록 간의 관계를 사전에 검증하는 로직을 삽입하여 데이터 중복 및 재생 오류를 방지.

### 예제

```php
<?php
function buildPopupData(array $config): array
{
    return [
        'source' => $config['source'] ?? '',
        'channel' => (int)($config['channel'] ?? 1),
    ];
}

$config = ['source' => 'sample.wav', 'channel' => 2];
echo json_encode(buildPopupData($config), JSON_UNESCAPED_UNICODE);
```

---

## 33. 고급 프로세스 간 통신 및 상태 영속화

### 개요

모듈 간 통신 방식을 표준화하고, 시스템 점검 설정과 같은 중요 환경 변수의 모델별 분기 처리를 통해 서버측 리소스 제어 안정성을 개선했습니다.

### 핵심 내용

- **IPC 표준화**: 컨트롤러와 TTS 프로세스 간의 상태 전달 메커니즘을 팝업 객체 참조 방식으로 통합하여, 세션 파편화를 방지하고 상태 관리의 일관성 확보.
- **조건부 로직 분기**:  및  각 모델의 하드웨어 특성과 정책에 따라 시스템 점검 및 음원 처리 프로세스를 동적으로 분기하여 호환성 강화.
- **리소스 영속화 정제**: 불필요한 디버깅 로그를 제거하고 상태 저장 로직의 원자성을 확보함으로써, 시스템 로그의 가독성 향상 및 운영 안정성 개선.
- **기능 무결성**: 에러 발생 가능성이 있는 레거시 코드와 오타를 정리하여 안정적인 서비스 가동 환경을 유지.

### 예제

```php
<?php
function applyModelSettings(string $model, array $config): array
{
    $defaults = [
        'check_interval' => 'daily',
        'cleanup_temp_files' => true,
    ];

    if ($model === 'standard') {
        return array_merge($defaults, $config);
    }

    return array_merge($defaults, $config, [
        'cleanup_temp_files' => false,
    ]);
}

print_r(applyModelSettings('standard', ['check_interval' => 'weekly']));
```

---

## 34. TTS 자원 생명주기 관리 및 정합성

### 개요

TTS 생성부터 방송 종료에 이르는 음원 자원의 전체 생명주기를 관리하는 백엔드 프로세스를 강화하여, 시스템 무결성을 확보했습니다.

### 핵심 내용

- **원자적 자원 삭제**: 방송되지 않은 임시 음원 파일이 시스템에 잔존하지 않도록, 재생 종료 시점과 연동된 클린업 엔진을 완성하여 저장소 효율성 개선.
- **데이터 정합성 유지**: `state.json` 기반의 상태 동기화 로직을 보강하여, 컨트롤러의 리스트 갱신과 실제 파일 상태 간의 불일치를 개선.
- **예외 처리 고도화**: 비정상적인 접속 끊김이나 미리듣기 버튼 오작동 시 발생하는 중복 알람을 차단하고, 서버 상태값 기반의 동적 UI 대응 능력을 확보.
- **안정적 명령 인터페이스**: 웹소켓과 PHP 백엔드 간의 통신 패킷 구성을 최적화하여 조작 명령의 누락 없는 전달 보장.

### 예제

```php
<?php
function cleanupTemporaryResource(array &$state, string $directory): void
{
    $name = $state['temp_file'] ?? null;

    if ($name) {
        $path = $directory . '/' . basename($name);
        if (is_file($path)) {
            unlink($path);
        }
    }

    $state['temp_file'] = null;
}

$dir = __DIR__ . '/media';
mkdir($dir, 0777, true);
file_put_contents($dir . '/temp.wav', 'temporary');

$state = ['temp_file' => 'temp.wav'];
cleanupTemporaryResource($state, $dir);

print_r($state);
@rmdir($dir);
```

---

## 35. NAT 보안 및 현지화 운영 인프라

### 개요

NAT 설정 정보의 보안 암호화 저장 체계를 구축하고, 관리자 운영 이벤트 로깅 인프라를 확장하여 SIP 환경에서의 운영 안정성과 투명성을 개선했습니다.

### 핵심 내용

- **보안 데이터 저장**: NAT 설정 시 TURN 서버 인증 비밀번호를 RSA 암호화(`CryptFunc`)하여 저장함으로써 시스템 보안성 강화.
- **시스템 상태 동기화**: NAT 설정 변경 즉시 웹소켓을 통해 바이너리에 데이터베이스 리로드 명령(`0x09`)을 전송하여 하드웨어 설정의 즉시 반영을 보장.
- **이벤트 로깅 연동**: NAT 설정 값과 사용 여부 등을 상세 로그로 기록하여 관리자 운영 이력 추적성 확보.
- **글로벌 인프라 확장**: 다국어 팩(ENG, KOR, RU, FR) 업데이트 및 관리 페이지 메뉴 명칭 표준화를 통해 전 세계 환경에서의 관리 편의성 개선.

### 예제

```php
<?php
function encryptSecret(string $value, string $key): string
{
    $iv = random_bytes(16);
    $cipher = openssl_encrypt(
        $value,
        'AES-256-CBC',
        hash('sha256', $key, true),
        OPENSSL_RAW_DATA,
        $iv
    );

    return base64_encode($iv . $cipher);
}

$token = encryptSecret('example-secret', 'local-development-key');
echo $token . PHP_EOL;
```

---

## 36. 모델별 마이크 제어 및 마이그레이션 안정성

### 개요

특정 모델의 마이크 설정(MIC Delay) 값을 관리하고, 시스템 초기화 및 업그레이드 상황에서도 사용자 설정이 유실되지 않도록 보존하는 마이그레이션 엔진을 강화했습니다.

### 핵심 내용

- **자동 마이그레이션 로직**: 펌웨어 업그레이드 및 시스템 초기화 시점마다 기기별 특성에 맞춘 마이크 설정값(Delay 1.5s 등)을 자동 판별하여 주입하는 마이그레이션 체계 구축.
- **데이터 무결성 보존**: 시스템 재시작이나 초기화 시점에 설정 파일(`device-config.json`) 내 설정값 유무를 런타임에 체크하고, 누락 시 기본값을 강제 적용하여 항상 정합성 있는 상태 유지.
- **운영 안정성**: 시스템 내 DSP 제어 바이너리와 서버 설정 값 간의 동기화 시퀀스를 강화하여, 설정 변경 즉시 실제 하드웨어 출력에 반영되도록 최적화.

### 예제

```php
<?php
$config = [
    'microphone' => [
        'delay' => null,
    ],
];

if ($config['microphone']['delay'] === null) {
    $config['microphone']['delay'] = 1.5;
}

file_put_contents(
    __DIR__ . '/device-config.json',
    json_encode($config, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE),
    LOCK_EX
);

echo "microphone delay: {$config['microphone']['delay']}
";
unlink(__DIR__ . '/device-config.json');
```

---

## 37. 방송 제어 및 모델 호환성 강화

### 개요

컨트롤러 상에서 BGM/TTS 소스 장치를 통합 관리하고, 기기별/상태별로 방송 프로세스를 조건에 따라 분기하여 방송 안정적으로 동작하도록 구성했습니다.

### 핵심 내용

- **소스 장치 관리 고도화**: TTS 즉시 송출 기능을 위한 프로세스 로직(ToBGM)을 추가하고, BGM 소스 장치 상태에 따라 TTS 생성을 조건부로 허용하는 안정적 제어 체계 구축.
- **방송 안정성 개선**: 모델 특성에 따라 재생 종료 후 자원 처리 시퀀스를 분기하여, 기기별 예외 상황을 고려하고 정합성 있는 상태 유지를 실현.
- **인터페이스 무결성**: 방송 시작 및 종료 등의 상태 변경 이벤트가 발생할 때 컨트롤러와 백엔드 간의 동기화 패킷 전달을 최적화하여 상태 변경이 누락되지 않도록 처리.

### 예제

```php
<?php
function canCreateAudio(array $status): bool
{
    return !($status['is_broadcasting'] ?? false);
}

$status = ['is_broadcasting' => false];

if (!canCreateAudio($status)) {
    http_response_code(409);
    exit("broadcasting in progress
");
}

echo "audio creation allowed
";
```

---

## 38. 미디어 제어 프로토콜 표준화

### 개요

음원 재생 볼륨 및 DSP 입출력 제어를 담당하는 미디어 제어 로직을 `Media` 클래스로 객체화하고, 장치별/타입별 통신 규약을 표준화하여 시스템 운영 신뢰성을 높였습니다.

### 핵심 내용

- **Media 객체 지향 리팩토링**: 기존 절차지향적이었던 볼륨 제어 로직을 `Media` 클래스로 캡슐화하여, 볼륨 조회/설정 메서드 등의 메서드를 통해 유지보수성과 재사용성을 개선.
- **다양한 스피커 타입 지원**: MASTER/SERVER/SLAVE/INDEPENDENT 타입별 동작을 표준화하고, 기기 특성에 따라 볼륨 제어 데이터 파이프라인을 동적으로 분기하는 아키텍처 완성.
- **입출력 데이터 정합성**: 사용자 요청 볼륨(Percent)을 시스템 내부 DSP 규격(dB)으로 변환하는 수식 엔진(`convertPercentToDb`)을 고도화하여 조작 정확도 확보.

### 예제

```php
<?php
class Media
{
    public function setVolume(int $percent): float
    {
        $percent = max(0, min(100, $percent));
        return $percent === 0 ? -80.0 : 20 * log10($percent / 100);
    }
}

$media = new Media();

foreach ([0, 25, 50, 100] as $percent) {
    printf("%d%% => %.2f dB
", $percent, $media->setVolume($percent));
}
```

---

## 39. 아이콘 데이터 정합성 및 마이그레이션 회복력

### 개요

시스템 내 리소스(아이콘) 등록 시 데이터베이스 정합성을 보장하고, 마이그레이션 진행 중 발생할 수 있는 중복 키 삽입 오류 및 비정상 종료 시 시스템 복구 로직을 강화했습니다.

### 핵심 내용

- **아이콘 등록 무결성**: 시스템 내 새로운 Audio 아이콘(audio.svg) 추가 시, 기존 테이블 데이터의 존재 여부를 사전에 검증하여 중복 삽입으로 인한 무결성 훼손을 차단.
- **마이그레이션 회복력 강화**: 마이그레이션 진행 중 특정 데이터가 존재하지 않아 강제 종료되던 이슈를 방지하고, 조건부 쿼리문 실행 체계를 도입하여 업그레이드 프로세스의 안정성 개선.
- **데이터베이스 자동화**: 시스템 초기화 또는 펌웨어 설치 시점에서 `db_table.sql`을 통한 아이콘 파라미터 자동 삽입 로직을 표준화하여, 수동 조작 없이 일관된 리소스 환경을 유지.

### 예제

```php
<?php
if (!in_array('sqlite', PDO::getAvailableDrivers(), true)) {
    exit("SQLite PDO driver is required for this example.\n");
}

$db = new PDO('sqlite::memory:');
$db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$db->exec('CREATE TABLE icons (name TEXT PRIMARY KEY, path TEXT)');

$check = $db->prepare('SELECT COUNT(*) FROM icons WHERE name = :name');
$check->execute([':name' => 'audio']);

if ((int)$check->fetchColumn() === 0) {
    $insert = $db->prepare('INSERT INTO icons(name, path) VALUES(:name, :path)');
    $insert->execute([':name' => 'audio', ':path' => 'img/audio.svg']);
}

print_r($db->query('SELECT * FROM icons')->fetchAll(PDO::FETCH_ASSOC));
```

---

## 40. 시스템 구조 표준화 및 의존성 관리

### 개요

시스템의 메인 웹 페이지(`index.php`) 및 연관 PHP 스크립트의 구조를 정규화하고, 시스템 리소스 의존성을 관리하는 방식을 표준화하여 페이지 렌더링 및 모듈 실행의 안정성을 확보했습니다.

### 핵심 내용

- **웹 페이지 구조 표준화**: 메인 웹 페이지 내의 HTML 구조를 표준화하고 비정상적인 PHP 태그 배치를 교정하여 브라우저 환경에서의 렌더링 일관성 확보.
- **스크립트 의존성 정규화**: 리소스 로딩 순서에 따라 스크립트 실행 오류가 발생할 수 있는 문제를 개선하기 위해 PHP 포함 모듈(공통 스크립트)의 로딩 로직을 표준화하여 안정적인 실행 환경 구축.

### 예제

```php
<?php
function loadDependencies(): array
{
    $files = ['common_define.php', 'common_script.php'];
    $loaded = [];

    foreach ($files as $file) {
        $path = __DIR__ . '/' . $file;
        if (is_file($path)) {
            require_once $path;
            $loaded[] = $file;
        }
    }

    return $loaded;
}

echo "loaded: " . implode(', ', loadDependencies()) . PHP_EOL;
```

---

## 41. 보안 및 데이터 정합성 인프라

### 개요

사용자 입력 값의 보안 검증(HTML 태그 주입 방지) 체계를 강화하고, 데이터 저장 및 조회 과정에서의 인코딩 정합성을 일관되게 관리하여 시스템 무결성을 보존하는 인프라를 구성했습니다.

### 핵심 내용

- **보안 이스케이핑 표준화**: 사용자 입력을 처리하는 전 모듈에 HTML 이스케이핑 엔진(`htmlspecialchars`)을 정규화하여 XSS 등 웹 보안 취약점을 방지하고 관리자 운영 무결성 확보.
- **데이터 정합성 보존**: DB 저장 시 이스케이프 된 데이터를 로드할 때 발생하는 이중 인코딩 문제를 해결하고, 필요 시 `htmlspecialchars_decode` 및 인코딩 정상화 과정을 거치도록 마이그레이션 루틴을 고도화.
- **글로벌 환경 대응성**: 다국어 운영 환경에서 발생하는 문자셋 왜곡 및 HTML 엔티티 깨짐 문제를 방지하기 위해 공통 이스케이프/언이스케이프 인프라를 수립하고 전 모듈 적용.

### 예제

```php
<?php
function secureDataLoad(string $data): string
{
    return htmlspecialchars_decode($data, ENT_QUOTES);
}

function sanitizeInput(string $data): string
{
    return htmlspecialchars($data, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

$input = '<b>example</b>';
$safe = sanitizeInput($input);

echo $safe . PHP_EOL;
echo secureDataLoad($safe) . PHP_EOL;
```

---

## 42. 시스템 회복력 및 멀티 브랜드 자원 관리

### 개요

하드웨어 모델별 정책을 시스템 핵심 엔진에 통합하고, 리소스 생명주기 및 마이그레이션의 원자성을 확보하여 서비스의 안정적인 동작을 목표로 했습니다.

### 핵심 내용

- **멀티 브랜드 유지보수 자동화**: 모델별 시스템 점검 정책(Daily/Weekly)을 하드웨어 모델명에 따라 동적으로 판별하고, 크론탭(`crontab`) 제어 명령을 일관되게 실행하여 오설정으로 인한 중단 위협 차단.
- **데이터 영속성 및 마이그레이션**: 시스템 업그레이드 시 아랍어 등 신규 라이센스 정보를 `config.json`에 주입할 때, 중복 체크 및 파일 락(`LOCK_EX`) 메커니즘을 통해 환경 설정 파일의 무결성을 보존.
- **객체 지향 미디어 제어**: 볼륨 조절 및 DSP 상태 조회를 `Media` 클래스로 객체화하여, MASTER/SERVER 등 장치 타입에 따른 제어 로직의 결합도를 낮추고 유지보수성을 높임.
- **리소스 클린업 엔진**: TTS 방송 종료 후 미사용 임시 파일을 감지하여 자동 삭제하고, 웹소켓 동기화 명령을 통해 실시간으로 리스트 정합성을 맞추는 자동화된 자원 관리 체계 구축.

### 예제

```php
<?php
function updateJsonFile(string $path, array $updates): void
{
    $fp = fopen($path, 'c+');

    if ($fp === false) {
        throw new RuntimeException('file open failed');
    }

    try {
        if (!flock($fp, LOCK_EX)) {
            throw new RuntimeException('lock failed');
        }

        $contents = stream_get_contents($fp);
        $data = $contents !== '' ? json_decode($contents, true) : [];
        $data = is_array($data) ? $data : [];

        $data = array_merge($data, $updates);

        ftruncate($fp, 0);
        rewind($fp);
        fwrite($fp, json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE));
        fflush($fp);
        flock($fp, LOCK_UN);
    } finally {
        fclose($fp);
    }
}

$path = __DIR__ . '/config.json';
updateJsonFile($path, ['license' => ['type' => 'trial']]);

echo file_get_contents($path);
unlink($path);
```

---

## 43. 최적화된 트랜잭션 엔진 및 보안 자원 관리

### 개요

AI 이벤트 프리셋 관리를 위한 효율적인 백엔드 엔진을 구축하고, 대규모 데이터 처리 시 발생하는 DB 부하를 최소화하는 트랜잭션 최적화 및 보안 리소스 처리 로직을 구현했습니다.

### 핵심 내용

- **벌크 쿼리 트랜잭션 최적화**: 다중 AI 이벤트 프리셋의 삽입/수정 시, 반복적인 개별 쿼리 대신 단일 트랜잭션 내에서 처리되는 벌크(`INSERT ... VALUES (row1), (row2)`) 및 `CASE` 문 기반 일괄 업데이트 로직을 설계하여 DB I/O 오버헤드를 줄였습니다.
- **보안 리소스 핸들링**: 음원 미리듣기 기능 구현 시, 파일의 실제 시스템 경로 노출을 방지하기 위해 `AES-256` 암호화와 `Base64` 인코딩이 결합된 보안 토큰 체계를 구축하여 내부 보안 강화.
- **시스템 정합성 보장**: `SQLite3` 및 `PDO`를 활용한 정밀한 데이터 제어 계층을 마련하고, 파일 락(`LOCK_EX`) 및 시스템 비정상 종료 시 복구 가능한 설정 저장 프로세스 완성.
- **자동화된 환경 동기화**: `application/json` 기반의 원시 데이터 파이프라인을 구축하여 클라이언트 요청을 효율적으로 처리하고, 시스템 초기화와 연동된 자동 리소스 마이그레이션 루틴 확보.

### 예제

```php
<?php
function sanitizeInput(string $value): string
{
    return htmlspecialchars($value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

if (!in_array('sqlite', PDO::getAvailableDrivers(), true)) {
    exit("SQLite PDO driver is required for this example.\n");
}

$db = new PDO('sqlite::memory:');
$db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$db->exec('CREATE TABLE events (id INTEGER PRIMARY KEY, name TEXT)');

$db->beginTransaction();

$stmt = $db->prepare('INSERT INTO events(name) VALUES(:name)');
$stmt->execute([':name' => sanitizeInput('sample event')]);

$db->commit();

print_r($db->query('SELECT * FROM events')->fetchAll(PDO::FETCH_ASSOC));
```

---

## 44. AI 백엔드 엔진 핵심 코드 정리 및 성능 최적화

### 개요

본 섹션은 장비A AI 이벤트 프리셋 시스템의 핵심 백엔드 처리 흐름을 예제로 정리합니다. PDO를 이용한 데이터베이스 처리와 트랜잭션, 일괄 수정, 음원 식별자 토큰 생성 등 주요 로직을 하나의 실행 예제로 구성했습니다.

### 핵심 내용

- **트랜잭션 기반 일괄 처리**: 여러 이벤트 프리셋을 하나의 트랜잭션에서 저장하고 `CASE` 문을 활용해 여러 행을 일괄 수정하도록 구성했습니다.
- **보안 토큰 처리**: 음원 파일의 실제 경로를 직접 반환하지 않고 식별 가능한 토큰으로 변환하여 외부 노출을 줄였습니다.
- **데이터 접근 계층 분리**: PDO를 통해 데이터베이스 작업을 명시적으로 관리하고, 파일 잠금과 트랜잭션을 사용해 데이터 정합성을 유지하도록 구성했습니다.

### 예제

```php
<?php
/**
 * 독립 실행형 이벤트 관리 예제
 * - SQLite 테이블 생성
 * - 여러 행 일괄 저장
 * - CASE 기반 일괄 수정
 * - 파일 경로 대신 식별자 토큰을 반환
 */

if (!in_array('sqlite', PDO::getAvailableDrivers(), true)) {
    exit("SQLite PDO driver is required for this example.\n");
}

$db = new PDO('sqlite:' . __DIR__ . '/event_demo.db');
$db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

$db->exec(
    'CREATE TABLE IF NOT EXISTS event_preset (
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        source_name TEXT NOT NULL
    )'
);

$insert = $db->prepare(
    'INSERT OR REPLACE INTO event_preset(id, name, source_name)
     VALUES(:id, :name, :source_name)'
);

$rows = [
    ['id' => 1, 'name' => '공지', 'source_name' => 'notice.wav'],
    ['id' => 2, 'name' => '안내', 'source_name' => 'guide.wav'],
];

$db->beginTransaction();

foreach ($rows as $row) {
    $insert->execute($row);
}

$db->commit();

$updates = [
    1 => '공지사항',
    2 => '안내방송',
];

$db->beginTransaction();

$case = [];
$params = [];

foreach ($updates as $id => $name) {
    $case[] = "WHEN :id_{$id} THEN :name_{$id}";
    $params[":id_{$id}"] = $id;
    $params[":name_{$id}"] = $name;
}

$sql = 'UPDATE event_preset
        SET name = CASE id ' . implode(' ', $case) . ' END
        WHERE id IN (' . implode(', ', array_keys($updates)) . ')';

$stmt = $db->prepare($sql);
$stmt->execute($params);
$db->commit();

$result = $db->query(
    'SELECT id, name, source_name FROM event_preset ORDER BY id'
)->fetchAll(PDO::FETCH_ASSOC);

foreach ($result as &$item) {
    $item['token'] = base64_encode(hash_hmac(
        'sha256',
        $item['source_name'],
        'local-demo-key',
        true
    ));
    unset($item['source_name']);
}

echo json_encode($result, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
```

---

# 정리

백엔드 로직은 기능을 단순히 나열하기보다 **데이터 처리, 파일 관리, 시스템 명령 실행, 상태 동기화 및 보안 처리의 책임을 분리하는 것**에 초점을 맞췄습니다.

주요 내용은 다음과 같습니다.

* 설정 및 상태 데이터의 저장과 동기화 처리
* 임시 음원과 캐시 파일의 생명주기 관리
* 장치 및 모델별 기능 차이를 고려한 백엔드 분기 처리
* REST API 인증과 요청 데이터 검증
* 대용량 파일 및 Multipart 통신 처리
* SQLite 및 JSON 기반 데이터 정합성 관리
* 시스템 명령 실행과 네트워크 설정 처리
* TTS 및 방송 자원의 생성, 사용, 삭제 흐름 관리
* 예외 상황과 동시 요청을 고려한 안정성 개선
* 데이터 이스케이프와 암호화 등 보안 처리

각 기능은 실제 프로젝트의 처리 흐름을 기준으로 정리하고, 예제에서는 핵심 로직을 독립적으로 확인할 수 있도록 구성했습니다.
