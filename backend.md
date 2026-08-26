# 백엔드에서 상태·비동기·자원 정합성을 다뤘습니다

제한된 컴퓨팅 환경에서는 웹 요청 하나의 성공 여부만으로 시스템이 정상이라고 보기 어렵습니다. 여러 클라이언트의 상태가 같아야 하고, 비동기 작업이 오래된 결과를 남기지 않아야 하며, 파일과 장치 자원의 생명주기도 함께 관리해야 했습니다.

## 프로젝트 정보

- 팀/소속: SW2팀
- 직접 설계·구현 범위: 상태 동기화, TTS, 자원 관리, 장치/API 통신, 데이터/정합성 관련 Backend 사례
- 기술적 의사결정: 직접 수행

---

# 01. 여러 관리자가 같은 방송 상태를 보도록 상태 전파 경로를 정리했습니다

여러 관리자가 동시에 접속하는 환경에서 방송 상태와 설정값이 서로 다르게 보이면 UI와 실제 시스템 상태가 어긋날 수 있었습니다. 상태 파일도 여러 요청에서 함께 수정해야 했습니다.

저는 채널별 상태를 JSON으로 중앙 관리하고, 변경된 상태를 WebSocket으로 여러 인스턴스에 전달하는 구조를 사용했습니다.

```text
State Change
     ↓
 JSON State
     ↓
 WebSocket
     ↓
Connected Instances
```

상태 파일 기록에는 `LOCK_EX`를 적용했습니다.

```php
file_put_contents(
    $statePath,
    json_encode($state, JSON_UNESCAPED_UNICODE),
    LOCK_EX
);
```

**기록 시점의 동시 쓰기 충돌 가능성을 줄이는 역할**로 사용했습니다.

Frontend에서는 WebSocket 수신과 상태 조회를 함께 사용해 화면 상태를 맞추는 흐름이 구성되어 있습니다.

### 무엇이 달라졌는가

- 채널별 상태를 하나의 기준으로 관리
- 여러 인스턴스에 동일한 상태 전파
- 상태 파일 기록 시 배타적 잠금 적용

실제 동기화 latency, 동시 연결 규모, Polling 주기와 reconnect recovery 결과는 별도로 확인해 기입할 수 있습니다.

---

# 02. TTS 요청의 완료 순서가 뒤집혀도 오래된 결과가 남지 않도록 했습니다

TTS 생성처럼 완료까지 시간이 걸리는 작업은 요청 순서와 완료 순서가 달라질 수 있습니다.

```text
Request A ─────────→ 완료
       \
        Request B ─→ 완료

완료 순서: B → A
```

A가 늦게 완료되어 B의 결과를 다시 덮으면 최신 상태가 과거 요청으로 돌아갈 수 있습니다.

그래서 작업마다 Token을 부여하고, **응답이 도착한 시점에도 현재 요청인지 다시 확인**하도록 했습니다.

```php
if ($token !== $latestToken) {
    if (is_file($tempFile)) {
        unlink($tempFile);
    }

    return false;
}
```

이 구조가 해결하는 문제는 일반적인 모든 Race Condition이 아니라 **비동기 완료 순서 역전에 따른 stale-result**입니다.

### 무엇이 달라졌는가

- 오래된 요청의 결과를 폐기
- 폐기된 요청의 임시 파일도 함께 정리
- 최신 요청을 기준으로 결과를 반영

연속 요청과 완료 순서 역전 상황을 실제로 테스트한 결과는 별도로 기입할 수 있습니다.

---

# 03. TTS 음원을 파일 하나가 아니라 lifecycle로 관리했습니다

TTS 임시 음원은 생성된 순간 끝나는 데이터가 아니었습니다.

```text
생성
 ↓
임시 파일
 ↓
방송
 ↓
방송 종료
 ↓
삭제 또는 Keep
```

사용자가 명시적으로 Keep을 요청하지 않은 임시 파일은 방송 종료 시 정리하고, 방송 상태와 모델 특성에 따라 처리 흐름을 분기했습니다.

```text
TTS Source
   ↓
Broadcast State
   ↓
Keep?
 ├─ Yes → Preserve
 └─ No  → Cleanup
```

### 구현에서 중요한 부분은 '언제 지우는가'를 명시하는 것이었습니다.

```php
if (!$isKeep && is_file($tempFile)) {
    unlink($tempFile);
}

$state['temp_file'] = null;
```

즉, 방송 종료 이후 보존 여부를 확인하고 사용하지 않는 임시 자원을 정리한 뒤 상태에서도 해당 파일을 제거합니다.

### 무엇이 달라졌는가

- TTS 생성과 파일 정리 시점을 연결
- Keep 여부를 lifecycle의 일부로 관리
- 방송 상태에 따른 모델별 처리 흐름 분리

실제 운영에서 파일 잔존량이나 삭제 지연이 얼마나 줄었는지는 별도 측정이 필요합니다.

---

# 04. 한 Controller의 판단만으로 음원을 삭제하지 않았습니다

하나의 음원이 여러 Controller의 Scheduler에서 참조될 수 있는 환경에서는 현재 Controller만 확인하고 삭제하면 다른 스케줄에 영향을 줄 수 있습니다.

그래서 삭제 전에 연결된 Controller들의 Scheduler DB를 조회해 **전역 참조 여부를 확인**하도록 했습니다.

```text
Delete Request
     ↓
Check Controller A
     ↓
Check Controller B
     ↓
Check Controller C
     ↓
Referenced?
 ├─ Yes → Keep
 └─ No  → Delete
```

파일 복사 전에는 대상 이름의 충돌 여부도 확인하고, 작업 실패 시 임시 생성 파일을 정리합니다.

### 무엇이 달라졌는가

- 단일 화면 기준의 삭제 판단을 여러 Controller의 참조 관계로 확장
- 중복 파일 생성 전 사전 확인
- 실패 시 임시 자원 정리

---

# 05. 장치/API 통신 실패를 호출부가 제어할 수 있는 오류 흐름으로 바꿨습니다

장치 간 API나 소켓 통신이 실패했을 때 `die()`처럼 즉시 종료하는 방식은 호출부가 실패 상황을 판단하고 적절한 응답을 만들기 어렵습니다.

그래서 명시적인 예외 흐름으로 변경하고 송수신 timeout을 설정했습니다.

```php
socket_set_option(
    $socket,
    SOL_SOCKET,
    SO_RCVTIMEO,
    ['sec' => $timeout, 'usec' => 0]
);
```

연결 실패, 인증 실패, 요청 실패, 응답 오류를 구분할 수 있도록 하고 상위 계층에서 HTTP 응답으로 연결할 수 있게 했습니다.

### 무엇이 달라졌는가

- `die()` 기반 종료 → Exception 기반 오류 전달
- 송수신 timeout 설정
- 호출부에서 연결/전송/응답 실패를 구분

---

# 06. 임베디드 시스템 설정과 호환성도 같은 기준으로 다뤘습니다

핵심 사례 외에도 다음과 같은 시스템 수준의 경험이 있습니다.

### 시스템 / 호환성

- Controller와 장치 버전 비교
- 단일 채널부터 4채널 이상 장비를 고려한 동적 채널 참조
- 모델별 시스템 점검 주기와 기본 설정 분기
- 마이크 Delay 등 장치별 기본값과 마이그레이션 처리

### 데이터 / 저장소

- SQLite / JSON 기반 상태·설정 저장
- 내부 메모리 / 외부 SD 저장소 정보 구조화
- Session Storage Interface와 배타적 read-modify-write 보호
- 다국어 Logging / Translation 분리

### 보안 / 자원

- 장치 등록 및 API 응답 상태 검증
- 입력값 검증 / HTML escaping
- 중복 리소스 확인 / 임시 파일 관리
- Multipart / 대용량 파일 처리

정량적인 성능, 장애 감소, 운영 규모는 실제 측정 후 추가할 수 있습니다.

---

# 07. AI Event Preset Backend를 Transaction과 Batch Update로 구성했습니다

AI Event Preset은 Frontend에서 동적 상태를 관리하고 Backend에서 이를 저장하는 별도의 Full-Stack 사례입니다.

Backend에서는 삽입과 수정을 각각 Transaction 단위로 처리합니다.

```text
Request
 ↓
Validation
 ↓
Insert Transaction
 ↓
Commit

Update
 ↓
CASE-based UPDATE
 ↓
Commit
```

핵심 저장 흐름은 다음과 같습니다.

```php
$db->beginTransaction();

foreach ($rows as $row) {
    $insert->execute($row);
}

$db->commit();
```

수정 작업은 여러 row를 `CASE` 기반 UPDATE로 묶었습니다. 핵심은 개별 row마다 UPDATE를 반복하지 않고 **한 SQL 문에서 변경 규칙을 표현한 것**입니다.

```sql
UPDATE event_preset
SET name = CASE id
    WHEN :id_1 THEN :name_1
    WHEN :id_2 THEN :name_2
END
WHERE id IN (:id_1, :id_2);
```

또한 `source_name`으로 HMAC을 생성해 식별자 Token을 반환하고 원본 `source_name`은 응답에서 제거합니다.

### 무엇이 달라졌는가

- Insert / Update 저장 흐름을 분리된 Transaction으로 관리
- 여러 수정 row를 `CASE` 기반으로 일괄 처리
- 응답에서 원본 `source_name`을 제거하고 식별자 Token 반환

실제 DB 처리시간, payload 변화, bulk 처리 규모, rollback 테스트 결과는 측정 후 추가할 수 있습니다.

---

# 이 경험에서 남은 설계 원칙

백엔드에서 반복해서 해결한 문제는 크게 다섯 가지였습니다.

1. 상태를 어떻게 일관되게 전달할 것인가
2. 비동기 작업의 오래된 결과를 어떻게 차단할 것인가
3. 파일과 자원의 lifecycle을 어떻게 관리할 것인가
4. 장치/API 통신 실패를 어디에서 처리할 것인가
5. 변경되는 장비와 정책을 어디까지 분리할 것인가

결국 코드를 많이 추가하는 것보다 **상태·자원·오류의 경계를 명확하게 만드는 것**이 시스템의 안정성을 좌우한다는 경험을 쌓았습니다.
