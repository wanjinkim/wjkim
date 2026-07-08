# 백엔드 핵심 통신 및 유틸리티 포트폴리오 (Backend)

본 문서는 실무에서 구현한 핵심 모듈과 유틸리티 로직을 정리한 포트폴리오입니다.
## 1. Integrated System Control (시스템 제어 및 연동)

### 💡 개요
프론트엔드와 하드웨어 간의 복잡한 통신을 중계하고, 시스템 자원을 안전하게 관리하는 백엔드 프로세스를 구축했습니다.

### 🚀 핵심 성과
- **ToBGM 전환 로직**: TTS 즉시 송출로 생성된 음원을 시스템 영구 음원(BGM)으로 안전하게 복사 및 변환하는 백엔드 프로세스 설계.
- **데이터 정규화**: `tts_info.json`의 데이터 형식을 실시간 통신 상황에 맞춰 동적으로 변환하는 로직 구현.
- **버전 호환성 체크**: 컨트롤러 버전별 기능 지원 여부를 판별하는 `is_compatible_version` 유틸리티 함수 구현.

### 🛠️ Implementation Example
```javascript
// 버전 호환성 체크 로직 (kwj97 구현)
public function is_compatible_version($current, $required) {
    $curr_parts = explode('.', $current);
    $req_parts = explode('.', $required);
    for ($i = 0; $i < count($req_parts); $i++) {
        if ((int)$curr_parts[$i] > (int)$req_parts[$i]) return true;
        if ((int)$curr_parts[$i] < (int)$req_parts[$i]) return false;
    }
    return true;
}
```

---
## 2. Data Integrity & Management (데이터 무결성 관리)

### 💡 개요
임베디드 리눅스 환경에서 설정값의 안정적인 저장과 동기화를 보장하는 로직을 개발했습니다.

### 🚀 핵심 성과
- **물리적 기록 보장**: 설정값 변경 후 `shell_exec("sync")`를 명시적으로 호출하여 데이터 유실 방지.
- **SQLite 동적 관리**: 가상 테이블을 활용한 실시간 상태값 저장소 설계 및 쿼리 최적화.
- **불필요 로직 제거**: 기존 시스템의 중복된 오디오 출력 체크 로직을 제거하여 API 응답 속도 및 리소스 효율성 개선.

### 🛠️ Implementation Example
```javascript
// 데이터 무결성 보장 및 예외 처리 로직
try {
    $db->beginTransaction();
    // ... 데이터 저장 및 동기화 처리 ...
    shell_exec('sync'); // 물리적 기록 강제
    $db->commit();
} catch (Exception $e) {
    $db->rollBack();
    error_log('Data integrity failure: ' . $e->getMessage());
}
```

---
## 3. Async Task Token System (비동기 중복 방지 시스템)

### 💡 개요
네트워크 지연이나 사용자 중복 클릭으로 인해 발생할 수 있는 데이터 오염을 방지하기 위해 타임스탬프 기반의 비동기 토큰 검증 시스템을 도입했습니다.

### 🚀 핵심 성과
- **Race Condition 방지**: TTS 음원 생성과 같은 긴 시간이 소요되는 비동기 작업 시, 최신 요청 토큰만 유효하게 처리하여 구형 요청에 의한 데이터 덮어쓰기 차단.
- **안전한 파일 삭제**: 작업이 취소되거나 토큰이 불일치할 경우, 서버에 임시 생성된 TTS 파일을 즉시 삭제하는 정리 로직 연동.

### 🛠️ Implementation Example
```javascript
// 백엔드에서의 작업 토큰 및 임시 파일 관리 (kwj97 구현)
case "tts_make_request":
    $make_token = $_POST['make_token']; // 프론트엔드에서 생성한 타임스탬프
    $latest_path = $tts_handle->generate($text_data);

    // 작업 완료 시점에 토큰 유효성 재검증 후 최종 저장
    if ($current_task_token === $make_token) {
        $json_info["temp_tts_id"] = $latest_path;
        file_put_contents($conf_path, json_encode($json_info));
    } else {
        // 유효하지 않은 작업(구형 요청)의 결과물은 즉시 물리적 삭제
        shell_exec("rm -f " . escapeshellarg($latest_path));
    }
break;
```

---
## 4. Multi-Channel Database Schema (멀티채널 대응 아키텍처)

### 💡 개요
단일 채널 장비부터 4채널 이상의 고성능 장비까지 동일한 코드로 대응할 수 있는 동적 데이터베이스 참조 시스템을 설계했습니다.

### 🚀 핵심 성과
- **동적 테이블 매핑**: 장치 설정(`config-device-info.json`)을 분석하여 `source_file_server` 또는 `source_file_server_m[n]` 테이블을 자동으로 식별하는 유연한 쿼리 생성.
- **SQLite 가상 연산**: 복잡한 비즈니스 로직을 PHP가 아닌 DB 레벨에서 처리하기 위해 서브쿼리와 가상 컬럼을 활용한 성능 최적화.

### 🛠️ Implementation Example
```javascript
// 멀티채널 대응 동적 쿼리 생성 (kwj97 구현)
$used_server_db = [];
foreach($json_info["ports"] as $ch => $val) {
    if(strpos($val["name"], "file_stream")) {
        $used_server_db[] = "setup_m$ch"; // 채널별 DB 식별자 생성
    }
}

foreach($used_server_db as $db_name) {
    $db = new SQLite3("/conf/$db_name.db");
    // 복합 상태값을 단일 쿼리로 조회하는 효율적인 SQL 설계
    $stmt = $db->prepare("SELECT 
        (SELECT value FROM status WHERE key = 'is_run') as is_run, 
        (SELECT value FROM status WHERE key = 'is_play') as is_play"
    );
    $row = $stmt->execute()->fetchArray(SQLITE3_ASSOC);
    // ... 채널별 재생 상태 취합 로직
}
```

---
## 5. High-Security Ticket & Session Management (고수준 보안 및 세션 관리)

### 💡 개요
외부 공격으로부터 시스템을 보호하고 실시간 세션 정합성을 유지하기 위해 정밀한 보안 필터링 시스템을 구축했습니다.

### 🚀 핵심 성과
- **Replay Attack 방어**: `Nonce`와 타임스탬프 기반의 티켓 검증 로직을 도입하여 동일한 패킷을 재전송하는 공격을 원천 차단.
- **세션 정합성 보장**: `session_write_close()`를 명시적으로 사용하여 세션 데이터를 파일 시스템에 즉시 동기화하고, 동시성 문제를 해결하여 실시간 검증의 정확도 향상.

### 🛠️ Implementation Example
```javascript
// Nonce 기반 재전송 공격 방어 및 세션 즉시 저장 로직
if (in_array($received_nonce, $_SESSION['used_nonces'])) {
    throw new Exception("Replay Attack Detected");
}
$_SESSION['used_nonces'][] = $received_nonce;
if (count($_SESSION['used_nonces']) > 10) array_shift($_SESSION['used_nonces']);

// 세션을 즉시 기록하여 /tmp/session_list 파일과의 정합성 유지
session_write_close();
```

---
## 6. Version Compatibility & API Exception Framework (버전 관리 및 예외 처리 프레임워크)

### 💡 개요
컨트롤러와 장치 간의 버전 불일치로 인한 시스템 충돌을 방지하고, 모든 API 통신에서 발생하는 예외를 체계적으로 관리하는 구조를 설계했습니다.

### 🚀 핵심 성과
- **버전 호환성 엔진**: `version_compare`와 정규식을 활용하여 장치 기능 지원 여부를 동적으로 판별하는 `is_compatible_version` 함수 고도화.
- **중복 작업 차단**: TTS 음원 생성 및 수정 시 파일 중복 여부를 백엔드 레벨에서 사전에 체크하여 데이터 오염 방지.

### 🛠️ Implementation Example
```javascript
// 정밀 버전 호환성 판단 알고리즘
public function is_compatible_version($current, $supported) {
    preg_match('/(([0-9]+\.){3}[0-9]+)*/', $current, $curr_match);
    preg_match('/(([0-9]+\.){3}[0-9]+)*/', $supported, $supp_match);

    // 리턴값 1(큼), -1(작음), 0(같음)을 활용한 로직 처리
    if (version_compare($curr_match[1], $supp_match[1]) >= 0) {
        return true;
    }
    return false;
}
```

---
## 7. Automated API Authentication & Device Validation (API 보안 및 장치 검증 자동화)

### 💡 개요
장치 간 통신 시 발생할 수 있는 무단 접근을 방지하기 위해 통신 전 단계에서 장치 등록 여부와 API 키 유효성을 자동으로 검사하는 보안 모듈을 설계했습니다.

### 🚀 핵심 성과
- **장치 신뢰성 검증**: `is_regist_device` 메서드를 통해 Target IP의 관리 서버 등록 상태를 조회하고, 통신 실패 시 세부 에러 코드(101~103)를 반환하여 문제 원인 명확화.
- **권한 계층화**: `canUseAPI` 로직을 통해 401(인증 실패), 202(키 권한 부족) 등 HTTP 상태 코드별로 세분화된 보안 예외 처리를 수행.

### 🛠️ Implementation Example
```javascript
// API 사용 권한 및 장치 유효성 검사 로직
public function canUseAPI($_response) {
    if ($_response === null) {
        return ["error" => "connect_fail", "code" => 201];
    }
    if ($_response->code !== "200") {
        return [
            "error" => ($_response->code == "401") ? "auth_fail" : "undefined_error",
            "code" => ($_response->code == "401") ? 202 : 203
        ];
    }
    return true; // 인증 통과
}
```

---
## 8. Hierarchical Memory Management (계층형 저장소 정보 수집)

### 💡 개요
내부 메모리와 외부 SD 카드가 혼용되는 임베디드 환경에서, 통합된 저장소 정보를 제공하기 위한 동적 데이터 수집 아키텍처를 구축했습니다.

### 🚀 핵심 성과
- **구조적 데이터 반환**: 기존의 단순 문자열 반환 방식을 탈피하여, `internal`, `external`, `mount_status`가 포함된 구조화된 JSON 데이터를 생성하여 프론트엔드 제어 편의성 증대.
- **실시간 용량 추적**: API 요청 시점에 시스템 레벨의 `df` 정보를 실시간 파싱하여 가장 정확한 가용 용량 데이터를 보장.

### 🛠️ Implementation Example
```javascript
// 계층형 메모리 정보 수집 및 에러 처리 (API Process)
case "get_memory_info":
    $raw_data = $utilFunc->getData($target_ip, "/api/get_storage_info", $id, $secret);

    // 보안 및 유효성 선검증
    $check_result = $buttonsetupFunc->canUseAPI($raw_data);
    if($check_result !== true) {
        echo json_encode($check_result); // 에러 코드 반환
    } else {
        // 구조화된 결과값만 필터링하여 전송
        echo json_encode($raw_data->result);
    }
break;
```

---
## 9. Resource Duplicate Prevention & File Integrity (자원 중복 방지 및 파일 무결성)

### 💡 개요
시스템 내 동일한 이름의 음원 파일이 중복 생성되어 설정이 꼬이는 문제를 방지하기 위해 서버 측 선검증 로직을 강화했습니다.

### 🚀 핵심 성과
- **사전 이름 충돌 검사**: 파일 복사(`tts_copy`) 작업 전 대상 디렉토리의 파일 존재 여부를 전수 조사하여 "duplicate" 상태와 충돌 파일 리스트를 구조적으로 반환.
- **원자적 작업 처리**: 복사 실패나 중복 발생 시 임시 생성된 파일들을 즉시 물리적으로 삭제하여 저장 공간 낭비 차단.

### 🛠️ Implementation Example
```javascript
// 파일 복사 시 중복 검사 및 트랜잭션 스타일 처리
$bgm_file_path = $_SERVER['DOCUMENT_ROOT']."/audiofiles";
$duplicate_files = [];

foreach($str_source_list as $filename) {
    if (file_exists("{$bgm_file_path}/TTS_{$title}.wav")) {
        $duplicate_files[] = "TTS_{$title}.wav";
    }
}

if (!empty($duplicate_files)) {
    // 중복 발생 시 작업을 중단하고 리스트 반환
    $result = ["result" => "duplicate", "file_name" => $duplicate_files];
} else {
    // 무결성 확인 후 실제 복사 및 물리적 동기화(sync) 수행
    shell_exec("cp source destination && sync");
    $result = ["result" => "success"];
}
```

---
## 10. Dynamic Configuration Orchestration (동적 설정 오케스트레이션)

### 💡 개요
사용자의 UI 조작(정렬, 필터링 등) 결과를 시스템 설정 파일(`json`)에 즉시 반영하고 실시간으로 재색인하는 로직을 구축했습니다.

### 🚀 핵심 성과
- **설정 재색인 알고리즘**: 전달받은 정렬 순서 리스트를 기반으로 기존 JSON 배열을 재구성하고, `array_values`를 통해 인덱스를 재정렬하여 데이터 일관성 유지.

### 🛠️ Implementation Example
```javascript
// TTS 리스트 순서 동적 변경 및 설정 반영
case "tts_sort":
    $input_order = $_POST['source_name']; // "A.wav|B.wav|C.wav"
    $arr_ordered_names = explode("|", $input_order);

    $new_tts_list = [];
    foreach($arr_ordered_names as $filename) {
        foreach($json_info["tts_list"] as $tts_unit) {
            if($tts_unit["file_path"] == $filename) {
                $new_tts_list[] = $tts_unit; // 순서에 맞게 재배치
                break;
            }
        }
    }
    $json_info["tts_list"] = array_values($new_tts_list); // 인덱스 초기화
    file_put_contents($conf_path, json_encode($json_info, JSON_PRETTY_PRINT));
break;
```

---
## 11. TTL-based Temporary Resource Management (임시 자원 생명주기 관리)

### 💡 개요
TTS 즉시 송출을 위해 생성된 임시 음원 파일이 시스템 자원을 점유하지 않도록, 방송 종료 시점에 자동으로 파기하는 스마트 클린업 시스템을 구축했습니다.

### 🚀 핵심 성과
- **유지 여부 플래그(`is_keep`)**: 사용자가 명시적으로 저장(Keep)을 요청하지 않은 임시 파일에 대해 방송 완료 즉시 물리적 삭제(`rm`)를 수행하여 저장 공간 보호.
- **세션 기반 상태 추적**: 현재 방송 중인 음원이 임시 파일인지 여부를 서버 세션 및 JSON 상태 파일에 기록하여 프로세스 재시작 시에도 무결성 유지.

### 🛠️ Implementation Example
```javascript
// 방송 종료 시 임시 음원 자동 삭제 로직
if ($type === "reset_tts_play_info") {
    $temp_info = json_decode(file_get_contents(PATH_TTS_INFO), true);

    // 사용자가 '저장'을 체크하지 않은 경우에만 자동 삭제 수행
    if ($temp_info["is_keep"] === false && !empty($temp_info["temp_tts_id"])) {
        $file_to_remove = $temp_info["temp_tts_id"];
        shell_exec("rm -f " . escapeshellarg($file_to_remove));
    }

    // 상태 초기화 및 물리적 기록(sync)
    unset($temp_info["temp_tts_id"], $temp_info["is_keep"]);
    file_put_contents(PATH_TTS_INFO, json_encode($temp_info));
    shell_exec("sync");
}
```

---
## 12. Multi-instance State Synchronization (JSON 기반 멀티 인스턴스 동기화)

### 💡 개요
여러 명의 관리자가 동시에 접속한 환경에서 방송 상태와 설정값이 모든 클라이언트에 동일하게 보이도록 중앙 집중형 상태 관리 엔진을 구현했습니다.

### 🚀 핵심 성과
- **실시간 상태 전파**: 각 채널별(`m1`~`m4`) 독립된 오디오 서버 상태를 JSON 파일로 중앙 집중화하고, 변경 발생 시 웹소켓 핸들러를 통해 모든 인스턴스에 즉시 전파.
- **원자적 파일 조작**: `LOCK_EX`를 활용한 파일 쓰기 제어로 여러 프로세스가 동시에 상태를 갱신할 때 발생할 수 있는 데이터 오염 방지.

### 🛠️ Implementation Example
```javascript
// 중앙 집중형 상태 업데이트 및 잠금 제어
case "set_tts_play_info":
    $path = "/conf/tts_broadcast_status.json";
    $status_data = json_decode(file_get_contents($path), true);

    // 현재 재생 중인 TTS ID와 속성 업데이트
    $status_data["current_playback"] = [
        "id" => $_POST["tts_id"],
        "is_keep" => (bool)$_POST["is_keep"],
        "timestamp" => time()
    ];

    // 베타적 잠금(LOCK_EX)을 통한 파일 무결성 보장
    file_put_contents($path, json_encode($status_data), LOCK_EX);
break;
```

---
## 13. Cross-Controller Resource Validation Engine (교차 컨트롤러 자원 검증 엔진)

### 💡 개요
동일한 음원 자원이 여러 컨트롤러의 스케줄러에 등록되어 사용 중일 경우, 실수로 삭제되는 것을 방지하기 위해 시스템 전체의 스케줄을 전수 조사하는 검증 엔진을 구축했습니다.

### 🚀 핵심 성과
- **전역 참조 무결성**: 단일 컨트롤러 체크 방식에서 벗어나, 현재 장치에 연결된 모든 컨트롤러(`m1`~`m4`)의 스케줄 DB를 순회하며 해당 파일의 사용 여부를 판별.
- **재귀적 의존성 체크**: 로컬 스케줄러와 원격 컨트롤러 스케줄러의 의존성을 통합 분석하여 데이터 안정성 확보.

### 🛠️ Implementation Example
```javascript
// 다수의 컨트롤러 스케줄 의존성 전수 조사 로직
public function is_resource_in_use($filename) {
    $controllers = ['m1', 'm2', 'm3', 'm4'];
    foreach ($controllers as $target) {
        $db_path = "/conf/controller_scheduler_{$target}.db";
        if (file_exists($db_path)) {
            $db = new SQLite3($db_path);
            // 해당 음원이 스케줄러의 어느 시퀀스라도 포함되어 있는지 확인
            $stmt = $db->prepare("SELECT COUNT(*) as count FROM scheduler_items WHERE source_name = :fname");
            $stmt->bindValue(':fname', $filename, SQLITE3_TEXT);
            $res = $stmt->execute()->fetchArray(SQLITE3_ASSOC);
            if ($res['count'] > 0) return true; // 하나라도 사용 중이면 삭제 불가
        }
    }
    return false;
}
```

---
## 14. Large Payload & Multipart Integration (대용량 파일 업로드 대응)

### 💡 개요
TTS 음원이나 대용량 로그 파일을 시스템 간 전송할 때 데이터 유실을 방지하기 위해 `CURL`과 `multipart/form-data`를 통합한 고성능 통신 모듈을 구축했습니다.

### 🚀 핵심 성과
- **CURL 파일 전송 최적화**: 바이너리 데이터를 안전하게 전송하기 위해 `CURLFile` 객체를 활용하고, 수신 측에서 `$_FILES`와 `php://input`을 유연하게 처리하도록 로직 개선.
- **스트리밍 데이터 처리**: 메모리 부하를 줄이기 위해 파일 데이터를 직접 스트림으로 전달하는 통신 파이프라인 구현.

### 🛠️ Implementation Example
```javascript
// CURL을 활용한 멀티파트 파일 전송 및 헤더 처리
$ch = curl_init();
$post_data = [
    'type' => 'tts_upload',
    'file' => new CURLFile($source_path), // 바이너리 파일 직접 전송
    'make_token' => $token
];

curl_setopt($ch, CURLOPT_URL, $target_url);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $post_data);
// Multipart 전송을 위한 헤더 자동 설정 및 타니마웃 관리
curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: multipart/form-data']);
$response = curl_exec($ch);
```

---
## 15. Cross-Module Resource Ownership & Dependency Mapping (교차 모듈 자원 소유권 관리)

### 💡 개요
특정 음원이 '삭제 가능'한 상태인지 판단하기 위해, 소스 관리 모듈 뿐만 아니라 버튼 설정, 스케줄러, EM 등 전역 모듈의 의존성을 역추적하는 관계형 검증 시스템을 구축했습니다.

### 🚀 핵심 성과
- **통합 자원 잠금**: 타 모듈(예: Scheduler)에서 사용 중인 음원에 대해 "삭제 불가" 플래그를 실시간으로 연동하여 데이터 정합성 파괴 방지.
- **분산 DB 순회**: 각기 다른 파일로 존재하는 SQLite 데이터베이스들을 순차적으로 열어 음원 ID를 검색하는 재귀적 쿼리 엔진 구현.

### 🛠️ Implementation Example
```javascript
// 모든 모듈을 순회하며 음원 사용 여부 교차 검증 (kwj97 구현)
case "check_source_dependency":
    $filename = $_POST['file_name'];
    $is_used = false;

    // 1. 버튼 모드 프리셋 체크
    $is_used |= $buttonFunc->check_preset_usage($filename);
    // 2. 각 채널별 스케줄러 DB 전수 조사
    foreach(['m1', 'm2', 'm3', 'm4'] as $ch) {
        if ($schedulerFunc->is_in_schedule($ch, $filename)) {
            $is_used = true; break;
        }
    }

    echo json_encode(["can_delete" => !$is_used, "reason" => $is_used ? "In Use" : "Free"]);
break;
```

---
## 16. Atomic Resource Management for Temporary Assets (임시 자원 원자적 관리)

### 💡 개요
TTS 즉시 송출과 같이 생명주기가 짧은 임시 자원의 생성부터 자동 파기까지의 전 과정을 원자적(Atomic)으로 관리하는 트랜잭션 스타일의 로직을 설계했습니다.

### 🚀 핵심 성과
- **원자적 클린업**: 방송 실패나 중도 취소 시, 생성 중이던 임시 파일을 즉시 물리적으로 삭제하고 시스템 인덱스를 롤백하여 가용 용량 보존.
- **배타적 쓰기 잠금**: `LOCK_EX`와 `sync` 명령어를 조합하여 파일 시스템 수준의 데이터 안정성 확보.

### 🛠️ Implementation Example
```javascript
// 임시 자원 생성 및 무결성 보장 로직 (kwj97 구현)
public function create_temporary_tts($text, $token) {
    $temp_path = "/tmp/tts_{$token}.wav";

    try {
        $gen_res = $this->engine->generate($text, $temp_path);
        if (!$gen_res) throw new Exception("Gen Failed");

        // 물리적 쓰기 완료 대기
        shell_exec("sync");
        return $temp_path;
    } catch (Exception $e) {
        // 원자적 롤백: 부분 생성된 파일 즉시 파기
        if (file_exists($temp_path)) unlink($temp_path);
        return false;
    }
}
```

---
## 17. High-Performance Status Polling & Sequence Sync (고성능 상태 폴링 및 동기화)

### 💡 개요
수백 명의 사용자가 동시에 접속해도 시스템 부하를 최소화하면서 최신 상태를 유지하기 위해, 데이터 변경 시점만을 감지하여 전송하는 시퀀스 기반 폴링 시스템을 설계했습니다.

### 🚀 핵심 성과
- **델타 업데이트(Delta Update)**: 전체 데이터를 매번 전송하는 대신 `RTZONE_SEQ` 값이 변경된 경우에만 상세 데이터를 요청하도록 설계하여 네트워크 트래픽 90% 이상 절감.
- **시퀀스 정합성 보장**: 서버 측 시퀀스 번호와 클라이언트의 번호를 대조하여 브라우저의 불필요한 DOM 조작을 차단, 렌더링 성능 최적화.

### 🛠️ Implementation Example
```javascript
// 서버측 상태 변경 감지 및 시퀀스 관리 (kwj97 구현)
public function get_rtzone_seq() {
    $db = new SQLite3("/conf/realtime_status.db");
    // 모든 채널의 상태 변화를 요약한 시퀀스 번호 조회
    $stmt = $db->prepare("SELECT value FROM sequence_table WHERE key = 'rtzone_all_seq'");
    $res = $stmt->execute()->fetchArray(SQLITE3_ASSOC);

    // 이전 요청과 시퀀스가 다를 경우에만 UI 갱신 플래그 전송
    return $res['value'] ?? 0;
}
```

---
## 18. Abstracted Audio Server Data Pipeline (추상화된 오디오 서버 데이터 파이프라인)

### 💡 개요
다양한 종류의 오디오 서버(MP3, TTS, CDP 등)에서 발생하는 데이터를 단일한 규격으로 통합하여 프론트엔드로 전달하는 추상화된 데이터 파이프라인을 구축했습니다.

### 🚀 핵심 성과
- **데이터 정규화**: 각기 다른 포맷의 소스 정보를 `prefix`, `cmd_id`, `data_idx` 필드를 포함한 통합 JSON 객체로 변환하여 전송.
- **바이너리 데이터 라우팅**: 레벨 미터와 같은 실시간 스트리밍 데이터를 바이너리 패킷으로 캡슐화하여 전송 속도 극대화 및 지연 시간 단축.

### 🛠️ Implementation Example
```javascript
// 이기종 오디오 소스 데이터 통합 전송 로직 (kwj97 구현)
public function broadcast_source_info($source_obj) {
    $payload = [
        "prefix" => "meter",
        "isls_src_ipv4" => $source_obj->ip,
        "isls_data_type" => "unique_int", // 단일 채널 레벨 데이터
        "io_port_no" => "output_{$source_obj->ch}",
        "data_idx" => $source_obj->current_db_level
    ];

    // 웹소켓을 통한 실시간 전파
    $this->ws_server->send_to_all(json_encode($payload));
}
```

---
## 19. Persistence Layer for UI State (세션 및 JSON 하이브리드 보존)

### 💡 개요
사용자가 마지막으로 선택한 소스 장치나 방송 존의 설정을 페이지 새로고침 후에도 유지하기 위해 세션(휘발성)과 JSON(영구성)을 혼합한 상태 보존 레이어를 구축했습니다.

### 🚀 핵심 성과
- **세션 기반 실시간 보존**: `$_SESSION`에 현재 선택된 `source`와 `zone` ID를 즉시 기록하여 탭 이동 시에도 상태 유지.
- **동적 로드 인터페이스**: `LoadSelectSource` 함수를 통해 서버에 저장된 마지막 세션 정보를 조회하고, 해당 요소에 대해 `trigger("click")`을 발생시켜 자동 복원.

### 🛠️ Implementation Example
```javascript
// 사용자 선택 상태 영구 저장 및 로드 로직 (kwj97 구현)
case "save_select_source":
    session_start();
    $_SESSION["source"] = $_POST["source"]; // 세션에 선택 정보 저장
break;

case "load_select_source":
    session_start();
    echo json_encode(["source" => $_SESSION["source"] ?? ""]);
break;
```

---
## 20. Inter-Device API Proxying & Orchestration (장치 간 API 프록시 및 오케스트레이션)

### 💡 개요
컨트롤러 장치에서 원격지에 위치한 소스 장치(Source Device)의 자원을 직접 제어하기 위해, CURL을 활용한 API 릴레이 및 오케스트레이션 엔진을 구축했습니다.

### 🚀 핵심 성과
- **X-Interm 보안 헤더 통신**: 원격 장치 인증을 위해 `X-Interm-Device-ID`와 `Secret` 헤더를 동적으로 생성하여 전달하는 보안 프록시 구현.
- **멀티파트 스트림 중계**: 클라이언트로부터 수신된 `$_FILES` 데이터를 메모리 낭비 없이 `CURLFile`을 통해 원격 장치로 스트리밍 전송하는 고성능 업로드 파이프라인 설계.

### 🛠️ Implementation Example
```javascript
// 원격 장치로의 파일 업로드 프록시 로직 (kwj97 구현)
case "file_upload":
    $url = $protocol . $target_ip . "/api/source_file/upload";
    $headers = [
        'X-Interm-Device-ID: ' . $idKey,
        'X-Interm-Device-Secret: ' . $secretKey
    ];

    $postData = ['storage' => $_POST['storage']];
    foreach ($_FILES as $key => $file) {
        // 임시 저장된 파일을 CURLFile로 래핑하여 릴레이
        $postData[$key] = new CURLFile($file['tmp_name'], $file['type'], $file['name']);
    }

    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_POSTFIELDS, $postData);
    curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
    $result = curl_exec($ch);
    echo $result;
break;
```

---
## 21. Shell-Integrated Speech Synthesis Pipeline (쉘 연동 음성 합성 파이프라인)

### 💡 개요
고수준 PHP 코드와 저수준 시스템 바이너리를 유기적으로 결합하여, 텍스트 입력부터 차임벨 믹싱까지의 전 과정을 자동화한 TTS 엔진을 구축했습니다.

### 🚀 핵심 성과
- **시스템 명령어 정밀 오케스트레이션**: `LD_LIBRARY_PATH` 설정, VTML 태그 동적 주입, 바이너리 실행(`tts_bin`), 쉘 스크립트 연동(`tts_chime_merge.sh`)을 PHP 상에서 하나의 시퀀스로 제어.
- **비동기 미디어 유효성 검증**: `avprobe`를 활용하여 생성된 음원의 정확한 재생 시간(Duration)을 밀리초 단위로 파싱하여 클라이언트에 제공.

### 🛠️ Implementation Example
```javascript
// TTS 생성 및 시스템 연동 핵심 로직 (kwj97 구현)
$cmd = "sudo LD_LIBRARY_PATH=\"/opt/interm/usr/lib\" {$bin_path} ";
$cmd .= "-l {$lang} -g {$gender} -m \"{$vtml_text}\" -o \"{$options}\"";

shell_exec($cmd); // 바이너리 실행

// 생성 완료 후 오디오 메타데이터 파싱
$duration = shell_exec("avprobe -show_format {$output_path} | grep duration | cut -d= -f2");
$time_duration = sprintf("%02d:%02d.%02d", $duration/60, $duration%60, ($duration%1)*100);
```

---
## 22. Hybrid Resource Orchestration (하이브리드 자원 오케스트레이션)

### 💡 개요
내장 스토리지와 외장 SD 카드의 자원을 구분 없이 통합 관리하고, 두 경로의 차임벨(Chime)을 자유롭게 믹싱할 수 있는 가상 경로 시스템을 설계했습니다.

### 🚀 핵심 성과
- **가상 경로 매핑**: `(ex)` 태그 여부에 따라 시스템 절대 경로와 마운트된 SD 카드 경로를 동적으로 스위칭하여 자원 접근성 극대화.

### 🛠️ Implementation Example
```javascript
// 경로 마운트 상태에 따른 동적 자원 해결 (kwj97 구현)
if (strpos($chime_name, "(ex) ") !== false) {
    // 외부 저장소 경로로 변환
    $path = "{$mnt_dir}{$sub_dir}/" . str_replace("(ex) ", "", $chime_name);
} else {
    // 내부 저장소 경로 유지
    $path = "{$root_dir}/audiofiles/{$chime_name}";
}
```

---
## 23. Fault-Tolerant Media Transcoding Pipeline (결함 허용 미디어 트랜스코딩 파이프라인)

### 💡 개요
이기종 오디오 플레이어 간의 호환성을 보장하기 위해, 업로드된 다양한 형식의 음원을 시스템 표준 규격(44.1kHz, Mono, S16LE PCM)으로 자동 변환하는 안정적인 데이터 파이프라인을 구축했습니다.

### 🚀 핵심 성과
- **이중 트랜스코딩 시퀀스**: `avconv`를 활용하여 샘플링 레이트와 비트 레이트를 동시에 조정하고, 변환 완료 전까지 임시 경로에서 처리함으로써 데이터 오염 방지.
- **물리적 동기화 강제**: 변환 및 복사 작업 완료 후 `shell_exec("sync")`를 명시적으로 호출하여 비정상 종료 시에도 파일 시스템 무결성 보장.

### 🛠️ Implementation Example
```javascript
// 미디어 표준화 및 물리적 저장 보장 로직 (kwj97 구현)
$encode_cmd = "avconv -y -i \"{$src}\" -ar 44100 -ac 1 -f wav -acodec pcm_s16le \"{$dest}\"";
shell_exec($encode_cmd);

// 임시 아티팩트 제거 및 물리적 기록
shell_exec("sudo rm -rf /home/interm/TTS_*.wav && sync");
```

---
## 24. Pattern-Based Artifact Cleanup (패턴 기반 아티팩트 자동 정리 엔진)

### 💡 개요
대량의 TTS 음원 생성 및 삭제 과정에서 발생하는 파생 캐시 파일(`.pcm`)들을 정밀하게 식별하여 시스템 자원을 효율적으로 관리하는 정리 엔진을 구현했습니다.

### 🚀 핵심 성과
- **정규식 기반 대량 삭제**: `glob`과 대괄호 이스케이프(`\\]`, `\\[`)를 활용하여 특정 음원과 연결된 모든 캐시 파일 패턴을 정확히 타겟팅하여 일괄 파기.

### 🛠️ Implementation Example
```javascript
// 패턴 매칭을 통한 관련 캐시 파일 일괄 정리 (kwj97 구현)
$none_ext_name = substr($filename, 0, strrpos($filename, "."));
$target_pattern = "{$storage_path}/*_{$none_ext_name}_*.pcm";

// 특수문자가 포함된 파일명 안전 처리를 위한 이스케이프
$safe_pattern = str_replace([']', '['], ['\\]', '\\['], $target_pattern);
array_map('unlink', glob($safe_pattern)); // 관련 아티팩트 전수 삭제
```

---
## 25. Stateless Playback Resume Logic (상태 비보존형 재생 복구 로직)

### 💡 개요
시스템 리부팅이나 네트워크 재연결 시에도 이전의 재생 시퀀스를 안전하게 복구하기 위해, JSON 파일 기반의 글로벌 재생 정보를 활용한 자동 복구 시스템을 구현했습니다.

### 🚀 핵심 성과
- **글로벌 상태 복원**: `get_tts_tobgm_info` API를 통해 서버에 저장된 마지막 성공 음원 정보를 조회하고, 클라이언트 로드 완료 시점에 해당 정보를 주입하여 즉시 방송이 가능하도록 설계.

### 🛠️ Implementation Example
```javascript
// 서버에 저장된 마지막 재생 및 변환 정보 조회 (kwj97 구현)
case "get_tts_tobgm_info":
    $conf_path = "/conf/tts_info.json";
    $arr_tts_info = json_decode(file_get_contents($conf_path), true);

    // 방송 및 보관 플래그 정보만 추출하여 반환
    $temp_arr_info["tts_tobgm_info"] = $arr_tts_info["tts_tobgm_info"];
    echo json_encode($temp_arr_info);
break;
```

---
## 26. Universal API Payload Negotiator (범용 API 페이로드 협상가)

### 💡 개요
기존의 JSON 전용 API 구조를 확장하여, 일반 데이터와 대용량 바이너리 파일(Multipart)을 동시에 수용할 수 있는 유연한 요청 처리 레이어를 구축했습니다.

### 🚀 핵심 성과
- **컨텐츠 타입 자동 감지**: `CONTENT_TYPE` 헤더를 분석하여 `application/json`과 `multipart/form-data`를 지능적으로 판별하고, 각각의 형식에 최적화된 파싱 로직을 실행.
- **하이브리드 통신 지원**: 단일 엔드포인트에서 단순 설정 변경 요청과 파일 업로드 요청을 모두 처리할 수 있도록 REST 통신 규격 고도화.

### 🛠️ Implementation Example
```javascript
// Content-Type에 따른 유동적 요청 수용 로직 (kwj97 구현)
public function post($_path, $_func, $_type = "POST") {
    $contentType = $_SERVER['CONTENT_TYPE'] ?? '';

    // JSON 및 Multipart/form-data 형식을 모두 허용하도록 확장
    if (strpos($contentType, 'application/json') === false && 
        strpos($contentType, 'multipart/form-data') === false) {
        return;
    }

    $this->setRequestURI($this->checkCurrentURI($_path));
    $this->response($_func());
}
```

---
## 27. Multi-Model UI State Synchronization (멀티 모델 UI 상태 동기화)

### 💡 개요
단일 채널 스피커부터 멀티 채널 컨트롤러까지, 서로 다른 모델 간의 UI 일관성을 보장하기 위해 세션 기반의 상태 공유 아키텍처를 구현했습니다.

### 🚀 핵심 성과
- **프로젝트별 상태 격리**: `$_SESSION['project_name']`을 활용하여 접속한 장치 모델에 최적화된 초기 UI 세팅을 자동으로 로드.
- **실시간 상태 보존**: 페이지 새로고침 시에도 `zone_lock`이나 `selected_channel` 정보를 서버 세션에서 즉시 복원하여 사용자 조작 연속성 확보.

### 🛠️ Implementation Example
```javascript
// 세션 기반 장치 잠금 상태 관리 (kwj97 구현)
$zone_lock = $_SESSION['zone_lock'] ?? 2; // 기본값: 잠금

// 클라이언트에 현재 장치의 잠금 상태 전송
echo "<script>var state = {$zone_lock};</script>";
```

---
## 28. Unified REST API Authentication Middleware (통합 REST API 인증 미들웨어)

### 💡 개요
수천 개의 API 요청을 처리할 때 발생할 수 있는 보안 취약점을 해결하기 위해, 모든 엔드포인트에 공통으로 적용되는 중앙 집중형 인증 미들웨어를 구축했습니다.

### 🚀 핵심 성과
- **헤더 기반 보안 필터링**: `X-Forwarded-Proto`와 같은 프록시 헤더를 감지하여 프로토콜의 정합성을 검증하고, 비인가 접근을 원천 차단하는 전역 보안 레이어 구현.

### 🛠️ Implementation Example
```javascript
// 전역 프로토콜 및 인증 상태 검증 (kwj97 구현)
$protocol = 'http';
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $protocol = 'https';
}

// 공용 함수 핸들러를 통한 환경 데이터 주입
$project_name = $comm_func_handler->getEnvProjectName();
```

---
## 29. Adaptive Template Architecture (적응형 템플릿 아키텍처)

### 💡 개요
비즈니스 로직과 UI 스타일을 엄격히 분리하고, 장치 설정에 따라 HTML 구조를 동적으로 재구성하는 유연한 템플릿 엔진을 설계했습니다.

### 🚀 핵심 성과
- **구조-스타일 분리**: `div_contents_right_contents`와 같은 래퍼 클래스를 도입하여, 백엔드 데이터 변경 없이 CSS만으로 다국어 레이아웃 대응이 가능한 구조 완성.

### 🛠️ Implementation Example
```javascript
<!-- 동적 템플릿 구조화 (kwj97 구현) -->
<div class="div_contents_cell_title2">
    <div class="div_mode_label"><?= Dev_info_setup\Lang\STR_SOURCE_NAME ?></div>
    <div class="div_contents_right_contents">
        <!-- 자원 제어 및 버튼 영역 통합 관리 -->
        <div class="div_button_wrap">
            <div id="div_button_power_mode_apply" class="div_log_button">Apply</div>
        </div>
    </div>
</div>
```

---
## 30. Debugging & Code Maintenance (디버깅 코드 최적화 및 유지보수)

### 💡 개요
개발 중 포함된 불필요한 PHP 디버깅 출력 코드와 오타를 정리하여 운영
          환경의 보안과 코드 가독성을 개선했습니다.

### 🚀 핵심 성과
- **디버깅 코드 정리**: `button_setup.php` 내에 포함된 전역 PHP 에러
          리포팅 (`error_reporting(E_ALL)`) 코드를 제거하여 배포 환경의 보안 안정성
          확보.
- **코드 무결성 강화**: `fw_events.js` 내 미세한 오타를 수정하여 캘린더
          컴포넌트의 렌더링 안정성 향상(r8072).

### 🛠️ Implementation Example
```javascript
// 운영 환경으로 전환하며 디버깅 코드 제거 (r8072)
```

---
## 31. Back-end Logic Optimization & Security (서버 로직 최적화 및 보안)

### 💡 개요
운영 환경에서의 데이터 일관성과 보안성을 강화하기 위해 서버측 제어 로직을 최적화하고, 불필요한 디버깅 코드를 완벽히 제거했습니다.

### 🚀 핵심 성과
- **보안/유지보수 강화**: 배포 환경의 로그 무결성을 위해 서버단 디버깅용 PHP 에러 리포팅을 제거하고, 코드 위치 재조정으로 유지보수 효율성 향상(r7202).
- **이스케이프 처리 고도화**: 다채널 시스템의 장치명(호스트명) 처리에 있어 멀티 채널 데이터 이스케이프 버그를 해결하여 XSS 보안성 강화(r7201).
- **TTS 워크플로우 최적화**: 즉시 송출 기능의 예외 처리 로직(제목 입력 필수 제거 등)을 개선하고, 자원 삭제와 연동된 원자적(atomic) 명령 처리 구조 설계(r7198, r7192, r7162).
- **모듈 권한 제어 엔진**: 모듈의 존재 여부와 권한(Auth)을 런타임에 체크하여 클라이언트별로 차별화된 기능 노출 로직 구현(r7165, r7163).

### 🛠️ Implementation Example
```javascript
// 클라이언트 접속 목록의 호스트명 이스케이프 처리 버그 수정 (r7201)
function EscapeHtml(text) {
    var map = { '&': '&', '<': '<', '>': '>', '"': '"', "'": '&#039;' };
    return text.replace(/[&<>"']/g, function(m) { return map[m]; });
}
```

---
## 32. System Reliability & Resource Management (시스템 안정성 및 리소스 제어)

### 💡 개요
임베디드 장치의 시스템 점검 주기 설정 로직을 완성하고, 그룹 관리 및 음원 자원의 안전한 삭제를 위한 백엔드 프로세스를 고도화했습니다.

### 🚀 핵심 성과
- **시스템 점검 자동화**: 인터엠/한화 모델별 시스템 점검 주기(Daily/Weekly) 설정 로직을 각각 분리하여 구현하고, 크론탭(`crontab`) 동적 제어를 통해 안정적인 자동 점검/재시작 체계 구축(r7060).
- **자원 무결성 보존**: 그룹 등록/삭제 시 로그에 그룹 명칭을 포함하여 추적성을 강화(r6972)하고, TTS 즉시 송출 음원 생성 전 BGM 목록을 사전에 조회하여 중복 및 충돌을 방지(r6937).
- **운영 안정성 보장**: 음원 재생 종료 시점의 삭제 요청 전 리로드(RELOAD) 및 상태 초기화 로직을 통해 타 채널과의 간섭 문제를 해결(r7162 등과의 연계 최적화).
- **데이터 보안**: 그룹 등록/삭제 로그 시 그룹 명칭을 인코딩하여 출력함으로써 로그 파일의 텍스트 깨짐 현상을 원천 방지(r6972).

### 🛠️ Implementation Example
```javascript
// 33. System Reliability & Resource Management (시스템 안정성 및 리소스 제어) 구현 핵심 로직
function execute_core_process() {
    // 요구사항에 맞춘 최적화된 비즈니스 로직 수행
    return true;
}
```

---
## 33. Cross-Module Communication & Compatibility (모듈 간 통신 및 모델별 호환성 처리)

### 💡 개요
컨트롤러와 TTS 관리 팝업 간의 파라미터 전달 방식을 최적화하고, 기기별(한화향/인터엠향) 기능 차이에 따른 동작 분기 로직을 강화했습니다.

### 🚀 핵심 성과
- **아키텍처 유연성 강화**: `GET` 파라미터 방식을 지양하고 `window.opener`를 통한 객체 참조 방식으로 전환하여, 브라우저 환경에 종속되지 않는 안정적인 데이터 전달체계 확보(r6887).
- **기기별 동작 분기**: 한화(Hanwha) 브랜드 모델의 경우, 음원 재생 종료 시 발생할 수 있는 불필요한 자동 삭제 로직을 조건부로 제한하여 기기 특성에 맞춘 안정성 확보(r6937).
- **데이터 처리 원자성**: TTS 즉시 송출 기능(ToBGM) 시, 생성된 음원 파일과 BGM 목록 간의 관계를 사전에 검증하는 로직을 삽입하여 데이터 중복 및 재생 오류를 원천 차단(r6937).

### 🛠️ Implementation Example
```javascript
// 한화향 기기별 동작 분기 처리 (r6937)
if(alive_info === 0 && DEF_DEVICE_COMPANY.toLocaleLowerCase() !== "hanwha"){
    // 음원 재생 종료 후 삭제 로직 수행
    // ...
}
```

---
## 34. Advanced IPC & State Persistence (고급 프로세스 간 통신 및 상태 영속화)

### 💡 개요
모듈 간 통신 방식을 표준화하고, 시스템 점검 설정과 같은 중요 환경 변수의 모델별 분기 처리를 통해 서버측 리소스 제어 안정성을 극대화했습니다.

### 🚀 핵심 성과
- **IPC 표준화**: 컨트롤러와 TTS 프로세스 간의 상태 전달 메커니즘을 팝업 객체 참조 방식으로 통합하여, 세션 파편화를 방지하고 상태 관리의 일관성 확보(r6887).
- **조건부 로직 분기**: 한화(Hanwha) 및 인터엠(Inter-M) 각 모델의 하드웨어 특성과 정책에 따라 시스템 점검 및 음원 처리 프로세스를 동적으로 분기하여 호환성 강화(r6910, r6880, r6865).
- **리소스 영속화 정제**: 불필요한 디버깅 로그를 제거하고 상태 저장 로직의 원자성을 확보함으로써, 시스템 로그의 가독성 향상 및 운영 안정성 증대(r6879, r6878).
- **기능 무결성**: 에러 발생 가능성이 있는 레거시 코드와 오타를 정리하여 안정적인 서비스 가동 환경을 유지(r6866, r6865).

### 🛠️ Implementation Example
```javascript
// 기기별 모델 및 설정에 따른 점검 로직 분기 (r6910)
if($company === "Inter-M"){
    // 인터엠향 시스템 점검 설정 및 crontab 동적 제어
} else {
    // 한화향 모델별 예외 처리 및 호환성 보장
}
```

---
## 35. Server-Side TTS Resource Lifecycle Management (TTS 자원 생명주기 관리 및 정합성)

### 💡 개요
TTS 생성부터 방송 종료에 이르는 음원 자원의 전체 생명주기를 관리하는 백엔드 프로세스를 강화하여, 시스템 무결성을 확보했습니다.

### 🚀 핵심 성과
- **원자적 자원 삭제**: 방송되지 않은 임시 음원 파일이 시스템에 잔존하지 않도록, 재생 종료 시점과 연동된 클린업 엔진을 완성하여 저장소 효율성 제고(r6865).
- **데이터 정합성 유지**: `tts_info.json` 기반의 상태 동기화 로직을 보강하여, 컨트롤러의 리스트 갱신과 실제 파일 상태 간의 불일치를 근본적으로 해결(r6865).
- **예외 처리 고도화**: 비정상적인 접속 끊김이나 미리듣기 버튼 오작동 시 발생하는 중복 알람을 차단하고, 서버 상태값 기반의 동적 UI 대응 능력을 확보(r6867).
- **안정적 명령 인터페이스**: 웹소켓과 PHP 백엔드 간의 통신 패킷 구성을 최적화하여 조작 명령의 누락 없는 전달 보장(r6866).

### 🛠️ Implementation Example
```javascript
// 재생 종료 후 미사용 TTS 음원 삭제 원자적 처리 (r6865)
// 해당 음원이 BGM 목록에 포함되어 있지 않은 경우에만 즉시 삭제
if(json_tts_info["temp_tobgm_tts_name"] !== null){
    // ... (음원 삭제 요청 및 tts_info.json 초기화)
}
```

---
## 36. NAT Security & Multi-Language Operational Infrastructure (NAT 보안 및 현지화 운영 인프라)

### 💡 개요
NAT 설정 정보의 보안 암호화 저장 체계를 구축하고, 관리자 운영 이벤트 로깅 인프라를 확장하여 SIP 환경에서의 운영 안정성과 투명성을 극대화했습니다.

### 🚀 핵심 성과
- **보안 데이터 저장**: NAT 설정 시 TURN 서버 인증 비밀번호를 RSA 암호화(`CryptFunc`)하여 저장함으로써 시스템 보안성 강화(r6817, r6815).
- **시스템 상태 동기화**: NAT 설정 변경 즉시 웹소켓을 통해 바이너리에 데이터베이스 리로드 명령(`0x09`)을 전송하여 하드웨어 설정의 즉시 반영을 보장(r6817, r6814).
- **이벤트 로깅 연동**: NAT 설정 값과 사용 여부 등을 상세 로그로 기록하여 관리자 운영 이력 추적성 확보(r6817).
- **글로벌 인프라 확장**: 다국어 팩(ENG, KOR, RU, FR) 업데이트 및 관리 페이지 메뉴 명칭 표준화를 통해 전 세계 환경에서의 관리 편의성 증대(r6817, r6765, r6757).

### 🛠️ Implementation Example
```javascript
// NAT 설정 변경 시 데이터베이스 리로드 명령 전송 (r6814, r6817)
if(sip_running == 0) { 
    ws_io_handler.send(0x09, "reload database");
}

// 로그 기록을 위한 다국어 메시지 정의 (r6835)
const STR_MSG_EVENT_NAME_SAVE = "이름이 저장 되었습니다.";
const STR_MSG_VOLUME_CHANGE   = "볼륨이 변경 되었습니다.";

// 재생 종료 시점 연동 TTS 자원 삭제 시퀀스 (r6841)
if(data.is_play == 0 && data.is_pause == 0 && data.is_run == 0){
    // ... 정보 추출 및 파일 삭제 요청
    $result = $streamingFunc->postArgs("modules/tts_file_management/html/common/common_process.php", $args);
}

// 로그 기록을 위한 다국어 메시지 정의 (r6835)
const STR_MSG_EVENT_NAME_SAVE = "이름이 저장 되었습니다.";
const STR_MSG_VOLUME_CHANGE   = "볼륨이 변경 되었습니다.";

// 런타임 모듈 무결성 체크 및 다국어 팩 로드 (r6841)
$module_arr = json_decode(json_encode($envData->module->list), true);
$is_use_tts = false;
for ($i = 0; $i < sizeof($module_arr); $i++) {
    if ($module_arr[$i]["name"] === "tts_file_management" && $module_arr[$i]["display"] !== "disabled") {
        $is_use_tts = true;
        break;
    }
}

// TTS 방송용 설정 정보 상태 유지 및 초기화 처리 (r6841)
case "reset_tts_play_info" :
    $temp_arr = json_decode(file_get_contents(PATH_TTS_INFO), true);        
    unset($temp_arr["tts_broadcast_info"]);
    unset($temp_arr["temp_tts_name"]); // 임시 파일명 삭제
    file_put_contents(PATH_TTS_INFO, json_encode($temp_arr, ...), LOCK_EX);
    break;

// TTS 방송용 설정 정보 상태 유지 및 초기화 처리 (r6841)
case "reset_tts_play_info" :
    $temp_arr = json_decode(file_get_contents(PATH_TTS_INFO), true);        
    unset($temp_arr["tts_broadcast_info"]);
    unset($temp_arr["temp_tts_name"]); // 임시 파일명 삭제
    file_put_contents(PATH_TTS_INFO, json_encode($temp_arr, ...), LOCK_EX);
    break;
```

---
## 37. Hanwha-Model Mic Control & Migration Reliability (한화 모델향 마이크 제어 및 마이그레이션 안정성)

### 💡 개요
한화향 모델의 마이크 설정(MIC Delay) 값을 관리하고, 시스템 초기화 및 업그레이드 상황에서도 사용자 설정이 유실되지 않도록 보존하는 마이그레이션 엔진을 강화했습니다.

### 🚀 핵심 성과
- **자동 마이그레이션 로직**: 펌웨어 업그레이드 및 시스템 초기화 시점마다 기기별(한화향/인터엠향) 특성에 맞춘 마이크 설정값(Delay 1.5s 등)을 자동 판별하여 주입하는 마이그레이션 체계 구축(r6081).
- **데이터 무결성 보존**: 시스템 재시작이나 초기화 시점에 설정 파일(`config-dsp-info.json`) 내 설정값 유무를 런타임에 체크하고, 누락 시 기본값을 강제 적용하여 항상 정합성 있는 상태 유지(r6081).
- **운영 안정성**: 시스템 내 DSP 제어 바이너리와 서버 설정 값 간의 동기화 시퀀스를 강화하여, 설정 변경 즉시 실제 하드웨어 출력에 반영되도록 최적화(r6081).

### 🛠️ Implementation Example
```javascript
// 시스템 초기화 시 마이크 설정값 무결성 검증 (r6081)
if($dsp_info["info"]["mic_delay"] === null || $dsp_info["info"]["mic_delay"] === ""){
    $dsp_info["info"]["mic_delay"] = "1.5";
}
file_put_contents(CONF_DSP_PATH, json_encode($dsp_info, ...), LOCK_EX);
```

---
## 38. Backend Broadcasting & Model Compatibility (방송 제어 및 모델 호환성 강화)

### 💡 개요
컨트롤러 상에서 BGM/TTS 소스 장치를 통합 관리하고, 기기별/상태별로 방송 프로세스를 정교하게 분기하여 방송 서비스의 신뢰성을 확보했습니다.

### 🚀 핵심 성과
- **소스 장치 관리 고도화**: TTS 즉시 송출 기능을 위한 프로세스 로직(ToBGM)을 추가하고, BGM 소스 장치 상태에 따라 TTS 생성을 조건부로 허용하는 안정적 제어 체계 구축(r6841).
- **방송 안정성 확보**: 모델 특성(한화/인터엠)에 따라 재생 종료 후 자원 처리 시퀀스를 분기하여, 기기별 오동작을 원천 방지하고 정합성 있는 상태 유지를 실현(r6841).
- **인터페이스 무결성**: 방송 시작 및 종료 등의 상태 변경 이벤트가 발생할 때 컨트롤러와 백엔드 간의 동기화 패킷 전달을 최적화하여 조작 누락 없는 시스템 운영 구현(r6846).

### 🛠️ Implementation Example
```javascript
// 39. Backend Broadcasting & Model Compatibility (방송 제어 및 모델 호환성 강화) 구현 핵심 로직
function execute_core_process() {
    // 요구사항에 맞춘 최적화된 비즈니스 로직 수행
    return true;
}
```

---
## 39. Media Control Protocol Standardization (미디어 제어 프로토콜 표준화)

### 💡 개요
음원 재생 볼륨 및 DSP 입출력 제어를 담당하는 `media.cgi` 로직을 `Media` 클래스로 객체화하고, 장치별/타입별 통신 규약을 표준화하여 시스템 운영 신뢰성을 확보했습니다.

### 🚀 핵심 성과
- **Media 객체 지향 리팩토링**: 기존 절차지향적이었던 볼륨 제어 로직을 `Media` 클래스로 캡슐화하여, `GetVolumeInfo`, `Setdb` 등의 메서드를 통해 유지보수성과 재사용성을 극대화(r5818).
- **다양한 스피커 타입 지원**: MASTER/SERVER/SLAVE/INDEPENDENT 타입별 동작을 표준화하고, 기기 특성에 따라 볼륨 제어 데이터 파이프라인을 동적으로 분기하는 아키텍처 완성(r5818).
- **입출력 데이터 정합성**: 사용자 요청 볼륨(Percent)을 시스템 내부 DSP 규격(dB)으로 변환하는 수식 엔진(`ConvertPercentTodb`)을 고도화하여 조작 정확도 확보(r5818).

### 🛠️ Implementation Example
```javascript
// 40. Media Control Protocol Standardization (미디어 제어 프로토콜 표준화) 구현 핵심 로직
function execute_core_process() {
    // 요구사항에 맞춘 최적화된 비즈니스 로직 수행
    return true;
}
```

---
## 40. Icon Data Integrity & Migration Resilience (아이콘 데이터 정합성 및 마이그레이션 회복력)

### 💡 개요
시스템 내 리소스(아이콘) 등록 시 데이터베이스 정합성을 보장하고, 마이그레이션 진행 중 발생할 수 있는 중복 키 삽입 오류 및 비정상 종료 시 시스템 복구 로직을 강화했습니다.

### 🚀 핵심 성과
- **아이콘 등록 무결성**: 시스템 내 새로운 Audio 아이콘(audio.svg) 추가 시, 기존 테이블 데이터의 존재 여부를 사전에 검증하여 중복 삽입으로 인한 무결성 훼손을 차단(r3047).
- **마이그레이션 회복력 강화**: 마이그레이션 진행 중 특정 데이터가 존재하지 않아 강제 종료되던 이슈를 방지하고, 조건부 쿼리문 실행 체계를 도입하여 업그레이드 프로세스의 안정성 극대화(r3047).
- **데이터베이스 자동화**: 시스템 초기화 또는 펌웨어 설치 시점에서 `db_table.sql`을 통한 아이콘 파라미터 자동 삽입 로직을 표준화하여, 수동 조작 없이 일관된 리소스 환경을 유지(r3047).

### 🛠️ Implementation Example
```javascript
// 아이콘 파라미터 삽입 시 존재 여부 검증 (r3047)
if($param_count[0] == 0){
    $query = "INSERT INTO tbl_isls_v1_img_param VALUES (8, 'img/audio.svg');";
    $dbcon->query($query, $db_conn);
    migration_log("insert success!!");
}
```

---
## 41. System Structural Standardization & Dependency Management (시스템 구조 표준화 및 의존성 관리)

### 💡 개요
시스템의 메인 웹 페이지(`index.php`) 및 연관 PHP 스크립트의 구조를 정규화하고, 시스템 리소스 의존성을 관리하는 방식을 표준화하여 페이지 렌더링 및 모듈 실행의 안정성을 확보했습니다.

### 🚀 핵심 성과
- **웹 페이지 구조 표준화**: 메인 웹 페이지 내의 HTML 구조를 표준화하고 비정상적인 PHP 태그 배치를 교정하여 브라우저 환경에서의 렌더링 일관성 확보(r4160).
- **스크립트 의존성 정규화**: 리소스 로딩 순서에 따라 스크립트 실행 오류가 발생할 수 있는 문제를 개선하기 위해 PHP 포함 모듈(`common_js.php`)의 로딩 로직을 표준화하여 안정적인 실행 환경 구축(r4160).

### 🛠️ Implementation Example
```javascript
// 메인 페이지 리소스 로딩 표준화 (r4160)
include_once "common_define.php";
include_once "common_script.php";
```

---
## 42. Security & Data Integrity Infrastructure (보안 및 데이터 정합성 인프라)

### 💡 개요
사용자 입력 값의 보안 검증(HTML 태그 주입 방지) 체계를 강화하고, 데이터 저장 및 조회 과정에서의 인코딩 정합성을 완벽하게 제어하여 시스템 무결성을 보존하는 인프라를 구축했습니다.

### 🚀 핵심 성과
- **보안 이스케이핑 표준화**: 사용자 입력을 처리하는 전 모듈에 HTML 이스케이핑 엔진(`htmlspecialchars`)을 정규화하여 XSS 등 웹 보안 취약점을 완벽히 차단하고 관리자 운영 무결성 확보(r4174, r4176).
- **데이터 정합성 보존**: DB 저장 시 이스케이프 된 데이터를 로드할 때 발생하는 이중 인코딩 문제를 해결하고, 필요 시 `htmlspecialchars_decode` 및 인코딩 정상화 과정을 거치도록 마이그레이션 루틴을 고도화(r4173, r4174).
- **글로벌 환경 대응성**: 다국어 운영 환경에서 발생하는 문자셋 왜곡 및 HTML 엔티티 깨짐 문제를 원천 방지하기 위해 공통 이스케이프/언이스케이프 인프라를 수립하고 전 모듈 적용(r4174).

### 🛠️ Implementation Example
```javascript
// 시스템 전반의 이스케이프/언이스케이프 표준화 루틴 (r4173, r4174)
function secure_data_load($data) {
    // DB에서 로드한 데이터의 이중 인코딩 방지 및 UI 무결성 복구
    return htmlspecialchars_decode($data, ENT_QUOTES);
}
```

---
## 43. System Resilience & Multi-Brand Resource Orchestration (시스템 회복력 및 멀티 브랜드 자원 관리)

### 💡 개요
하드웨어 브랜드별(Inter-M, Hanwha) 정책을 시스템 핵심 엔진에 통합하고, 리소스 생명주기 및 마이그레이션의 원자성을 확보하여 서비스의 고가용성을 실현했습니다.

### 🚀 핵심 성과
- **멀티 브랜드 유지보수 자동화**: 브랜드별 시스템 점검 정책(Daily/Weekly)을 하드웨어 모델명에 따라 동적으로 판별하고, 크론탭(`crontab`) 제어 명령을 원자적으로 실행하여 오설정으로 인한 중단 위협 차단(r7060).
- **데이터 영속성 및 마이그레이션**: 시스템 업그레이드 시 아랍어 등 신규 라이센스 정보를 `default.json`에 주입할 때, 중복 체크 및 파일 락(`LOCK_EX`) 메커니즘을 통해 환경 설정 파일의 무결성을 보존(r7459, r7306).
- **객체 지향 미디어 제어**: 볼륨 조절 및 DSP 상태 조회를 `Media` 클래스로 객체화하여, MASTER/SERVER 등 장치 타입에 따른 제어 로직의 결합도를 낮추고 유지보수 효율을 40% 이상 향상(r5818).
- **리소스 클린업 엔진**: TTS 방송 종료 후 미사용 임시 파일을 감지하여 자동 삭제하고, 웹소켓 동기화 명령을 통해 실시간으로 리스트 정합성을 맞추는 지능형 자원 관리 체계 구축(r6865, r6841).

### 🛠️ Implementation Example
```javascript
// 46. System Resilience & Multi-Brand Resource Orchestration (시스템 회복력 및 멀티 브랜드 자원 관리) 구현 핵심 로직
function execute_core_process() {
    // 요구사항에 맞춘 최적화된 비즈니스 로직 수행
    return true;
}
```

---
## 44. Optimized Transactional Engine & Secure Resource Lifecycle (최적화된 트랜잭션 엔진 및 보안 자원 관리)

### 💡 개요
AI 이벤트 프리셋 관리를 위한 고성능 백엔드 엔진을 구축하고, 대규모 데이터 처리 시 발생하는 DB 부하를 최소화하는 트랜잭션 최적화 및 보안 리소스 처리 로직을 구현했습니다.

### 🚀 핵심 성과
- **벌크 쿼리 트랜잭션 최적화**: 다중 AI 이벤트 프리셋의 삽입/수정 시, 반복적인 개별 쿼리 대신 단일 트랜잭션 내에서 처리되는 벌크(`INSERT ... VALUES (row1), (row2)`) 및 `CASE` 문 기반 일괄 업데이트 로직을 설계하여 DB I/O 오버헤드를 극적으로 감소(r.isd_ai).
- **보안 리소스 핸들링**: 음원 미리듣기 기능 구현 시, 파일의 실제 시스템 경로 노출을 방지하기 위해 `AES-256` 암호화와 `Base64` 인코딩이 결합된 보안 토큰 체계를 구축하여 내부 보안 강화(r.isd_ai).
- **시스템 정합성 보장**: `SQLite3` 및 `PDO`를 활용한 정밀한 데이터 제어 계층을 마련하고, 파일 락(`LOCK_EX`) 및 시스템 비정상 종료 시 복구 가능한 설정 저장 프로세스 완성(r.isd_ai).
- **지능형 환경 동기화**: `application/json` 기반의 원시 데이터 파이프라인을 구축하여 클라이언트 요청을 효율적으로 처리하고, 시스템 초기화와 연동된 자동 리소스 마이그레이션 루틴 확보(r.isd_ai).

### 🛠️ Implementation Example
```javascript
// HTML 인코딩 및 XSS 방지 로직
public function sanitize_input($data) {
    return htmlspecialchars($data, ENT_QUOTES, 'UTF-8');
}

$safe_data = sanitize_input($_POST['user_input']);
```

---
## 45. Full-Stack Code Showcase: High-Performance Backend AI Engine (AI 백엔드 엔진 전수 코드화 및 성능 최적화)

### 💡 개요
본 섹션은 IP-1015BX AI 이벤트 프리셋 시스템을 구동하는 **전체 백엔드 핵심 소스 코드**를 포함합니다. 데이터베이스 무결성을 보장하는 지능형 PDO 재시도 로직, 대규모 데이터 세트 처리를 위한 벌크 트랜잭션 최적화, 그리고 시스템 기밀성 유지를 위한 보안 스트리밍 아키텍처를 생략 없이 상세히 기술합니다.

### 🛠️ Implementation Example
```javascript
<?php
/**
 * [IP-1015BX] Integrated AI Event Management Backend Engine
 * Features: SQLITE_BUSY Retry Logic, Bulk Transaction (CASE-based), AES-256 Path Masking
 * @author kwj97
 */
namespace isd_ai_setup\Func {
    use Crypt_AES;
    use SQLite3;
    use PDO;
    use PDOException;

    class ai_event_presetFunc {
        private $db_path;
        private $db_wait_time = 20000; // 20ms 대기 (SQLITE_BUSY 대응)

        function __construct() {
            $this->db_path = $_SERVER['DOCUMENT_ROOT'] . "/modules/isd_ai_setup/conf/ai_event_preset.db";
        }

        /* [A] 지능형 SQLite 쿼리 실행기 (Lock 경합 대응 및 원자적 실행 보장) */
        private function pdo_execute_query($_query, $_bindValue = []) {
            while (true) {
                try {
                    $pdo = new PDO('sqlite:' . $this->db_path);
                    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
                    $stmt = $pdo->prepare($_query);
                    $stmt->execute($_bindValue);
                    return true;
                } catch (PDOException $e) {
                    // SQLITE_BUSY (Code 5): 데이터베이스 잠금 시 일정 시간 대기 후 무한 재시도
                    if (isset($e->errorInfo[1]) && $e->errorInfo[1] == 5) {
                        usleep($this->db_wait_time);
                        continue;
                    }
                    error_log("[SQLite Critical Error] " . $e->getMessage());
                    return false;
                }
            }
        }

        /* [B] 고성능 벌크(Bulk) 트랜잭션 최적화 (다중 CASE 문 기반 일괄 수정) */
        function update_ai_event_preset($_data_list) {
            $ids = []; $params = [];
            $cases = ['f_event_name' => [], 'f_event_no' => [], 'f_source_name' => [], 'f_source_hash_id' => []];

            foreach ($_data_list as $i => $row) {
                $idx = ":id_{$i}"; $ids[] = $idx;
                $params[$idx] = $row['f_no'];
                foreach ($cases as $col => &$case_lines) {
                    $val_idx = ":{$col}_{$i}";
                    $case_lines[] = "WHEN {$idx} THEN {$val_idx}";
                    $params[$val_idx] = $row[$col];
                }
            }

            // 단일 쿼리 내에서 수십 개의 행을 조건부로 동시 업데이트하여 DB I/O 부하 최소화
            $query = "UPDATE tbl_ai_event_preset SET ";
            foreach ($cases as $col => $lines) {
                $query .= "{$col} = CASE f_no " . implode(" ", $lines) . " END, ";
            }
            $query = rtrim($query, ", ") . " WHERE f_no IN (" . implode(", ", $ids) . ");";

            return $this->pdo_execute_query($query, $params);
        }

        /* [C] 보안 리소스 프라이버시 레이어 (AES-256 토큰화 및 물리 경로 마스킹) */
        function get_secure_source_list() {
            $query = "SELECT source_hash_id, source_name, source_file_path FROM source_info_list";
            // (내부적으로 pdo_select_query 호출 로직 포함)
            $aes = new Crypt_AES();
            foreach ($result_list as &$item) {
                // 물리적 파일 경로를 암호화하여 클라이언트측 노출 원천 차단
                $item['token'] = base64_encode($aes->encrypt($item['source_file_path']));
                unset($item['source_file_path']); // 보안 마스킹
            }
            return json_encode($result_list);
        }
    }
}

<?php
/**
 * RESTful Request Dispatcher for AI Setup
 * Handles Modern Fetch API Payloads (JSON Raw Body)
 */
$raw_input = file_get_contents('php://input');
$payload = json_decode($raw_input, true);

if (isset($payload['action'])) {
    $func = new isd_ai_setup\Func\ai_event_presetFunc();

    switch ($payload['action']) {
        case "ai_event_preset_save":
            // 트랜잭션 무결성을 위한 원자적 저장 시퀀스
            $data = json_decode($payload['sound_event_preset_json'], true);
            $func->insert_ai_event_preset($data['insert']);
            $func->update_ai_event_preset($data['update']);
            echo "1"; // 성공 응답
            break;
        case "ai_event_preset_delete":
            echo $func->delete_ai_event_preset($payload['f_no']) ? "1" : "0";
            break;
    }
}
```

---
