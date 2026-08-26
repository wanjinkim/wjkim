# 레거시 환경의 UI에서 실시간 통신과 상태 경계를 설계했습니다

레거시 환경의 UI에서는 단순한 화면 구현보다 **장치 상태가 여러 실행 컨텍스트를 거쳐 화면에 전달되는 과정**과 기존 환경과의 호환성을 함께 고려해야 했습니다.

## 프로젝트 정보
- 팀/소속: SW2팀
- 직접 설계·구현 범위: 아래 Frontend 사례
- 기술적 의사결정: 직접 수행

---

# 01. DOM 상태 변화를 Parent Controller까지 전달하는 구조를 만들었습니다

임베디드 제어 UI에서는 방송 서버 상태, iframe 내부 상태, Parent Controller 상태가 서로 다른 시점에 바뀔 수 있었습니다.

특히 `class`, `disabled` 같은 UI 속성이 바뀌었는데도 부모 Controller가 이를 알지 못하면 실제 상태와 조작 가능한 UI가 어긋날 수 있었습니다.

저는 **상태 변화 감지와 iframe 통신을 분리**했습니다.

```text
DOM / UI State
      ↓
MutationObserver
      ↓
postMessage
      ↓
Parent Controller
```

```javascript
const observer = new MutationObserver((mutations) => {
    mutations.forEach((mutation) => {
        if (mutation.attributeName === 'disabled') {
            const isDisabled = mutation.target.disabled;

            window.parent.postMessage(
                {
                    TTS_SOURCE_PLAYING_STATUS: isDisabled
                },
                controller_url
            );
        }
    });
});
```

`postMessage`가 상태의 원천이 아니라 DOM 변화가 원천이고, `MutationObserver`가 그 변화를 감지해 부모로 전달한다는 점이 핵심입니다.

### 무엇이 달라졌는가

- iframe → Parent 상태 전달
- 방송 서버 상태와 UI 상태 연결
- 재생 가능 여부를 Parent에 전달
- 전역 UI 상태 갱신

실제 latency 감소는 측정 후 추가할 수 있습니다.

---

# 02. Direct 전송을 도입하면서 기존 Legacy 방식도 함께 유지했습니다

파일 업로드/다운로드 경로를 개선하면서 단순히 새로운 방식을 적용하는 것으로 끝내지 않고, **실행 환경에 따라 Direct 또는 Legacy 경로를 선택할 수 있도록** 구성했습니다.

구체적인 운영 배경은 환경별 지원 여부를 직접 확인한 뒤 보완할 수 있습니다.

### 선택한 구조

```text
Source Capability Check
        ↓
directUpload / directDownload
        ↓
┌───────────────────┐
│ 지원          미지원 │
↓                 ↓
Direct Strategy  Legacy Strategy
```

핵심은 신규 전송 경로를 기존 방식의 대체품으로만 보지 않고 **전송 전략 자체를 분리한 것**입니다.

### Capability 기반 선택

```javascript
function check_source_capa() {
    let capa_response = commonFunc.postArgs(
        "modules/birdview/html/common/birdview_process.php",
        submitArgs
    );

    try {
        let json_data = JSON.parse(capa_response);

        if (
            json_data &&
            json_data.code === 0 &&
            json_data.capabilities
        ) {
            child_source_capa = json_data.capabilities;
        } else {
            child_source_capa = {
                directUpload: false,
                directDownload: false
            };
        }
    } catch (_e) {
        child_source_capa = {
            directUpload: false,
            directDownload: false
        };
    }
}
```

실제 실행 경로는 다음처럼 결정합니다.

```javascript
let executeUpload =
    child_source_capa.directUpload
        ? UploadStrategies.direct
        : UploadStrategies.legacy;
```

### Direct Upload

파일을 `ArrayBuffer`로 읽고 Transferable로 iframe에 전달합니다.

```javascript
const UploadStrategies = {
    direct: function(upload_source_list, storage) {
        let files_promises = [];
        let transferables = [];

        $.each(upload_source_list, function(_i, _e) {
            let promise = _e.arrayBuffer().then(function(buffer) {
                transferables.push(buffer);

                return {
                    name: _e.name,
                    type: _e.type,
                    buffer: buffer
                };
            });

            files_promises.push(promise);
        });

        Promise.all(files_promises).then(function(files_data) {
            document
                .getElementById('test')
                .contentWindow
                .postMessage(
                    {
                        action: "DIRECT_UPLOAD",
                        files: files_data,
                        storage
                    },
                    source_url,
                    transferables
                );
        });
    }
};
```

iframe에서는 전달받은 바이너리 데이터를 다시 `File` 객체로 구성해 기존 업로드 API로 연결합니다.

```javascript
var files_data = event.data.files;

for (var i = 0; i < files_data.length; i++) {
    var file_info = files_data[i];

    var file = new File(
        [file_info.buffer],
        file_info.name,
        { type: file_info.type }
    );

    form_source_list.append("file_" + i, file);
}
```

### 무엇이 달라졌는가

- Capability에 따라 전송 경로 선택
- Direct / Legacy 전략 독립
- Parent ↔ iframe 바이너리 데이터 전달
- 기존 환경의 fallback 경로 유지

Transferable은 브라우저 간 바이너리 전달에서 불필요한 데이터 복사를 줄이기 위한 경로로 사용했으며, 모든 메모리 복사가 사라지는 `Zero-copy`로 확대해서 표현하지 않습니다.

실제 전송 시간, 메모리, CPU, 서버 Proxy 부담, 사용자 대기시간 변화는 측정 후 추가할 수 있습니다.

---

# 03. Binary WebSocket 데이터를 Header 기준으로 해석했습니다

실시간 방송 제어와 장치 상태 데이터는 빈번하게 전달되기 때문에 수신된 바이너리 데이터의 종류를 먼저 구분할 필요가 있었습니다.

8-byte Header를 기준으로 Command ID, Binary 여부, Length 등을 해석합니다.

```javascript
const buffer = msg.data;

const cmdId =
    new Int8Array(
        buffer.slice(0, 1)
    )[0];

const isBinary =
    new Int8Array(
        buffer.slice(2, 3)
    )[0];

const length =
    new Int32Array(
        buffer.slice(4, 8)
    )[0];
```

이후 Command ID에 따라 Alive 정보와 실시간 상태 데이터를 분기합니다.

### 무엇이 달라졌는가

WebSocket으로 들어오는 여러 장치 데이터를 Command 단위로 구분해 필요한 처리 흐름으로 연결할 수 있게 했습니다.

실제 처리량과 latency는 측정 후 추가할 수 있습니다.

---

# 04. iframe과 Parent 사이의 상태 요청/응답 경계를 만들었습니다

Parent Controller와 여러 제어 모듈 iframe 사이에 직접 DOM을 공유하지 않고 `postMessage`를 통신 경계로 사용했습니다.

```text
Parent
  ↕ postMessage
Iframe
```

Parent에서 상태를 요청할 때는 일회성 listener를 등록하고 응답을 받으면 제거하는 Promise wrapper를 사용했습니다.

```javascript
function send_to_iframe_is_play() {
    return new Promise((resolve) => {
        const listener = (event) => {
            if (
                event.data &&
                typeof event.data.play_status !== 'undefined'
            ) {
                window.removeEventListener(
                    'message',
                    listener
                );

                resolve(event.data.play_status);
            }
        };

        window.addEventListener(
            'message',
            listener
        );

        targetIframe.contentWindow.postMessage(
            { ask_playing: true },
            source_url
        );
    });
}
```

수신 측에서는 `origin`을 확인해 메시지 수신 경계를 검증합니다.

```javascript
window.addEventListener('message', (event) => {
    if (event.origin !== source_url) {
        return;
    }

    // expected message only
});
```

### 무엇이 달라졌는가

- Parent → iframe 상태 요청
- iframe → Parent 상태 보고
- 응답 listener의 수명 관리
- iframe 간 상태 전달 경계 명확화

---

# 05. TTS 요청의 오래된 결과를 UI에 반영하지 않았습니다

TTS 생성 요청이 연속으로 발생하면 요청 시작 순서와 완료 순서가 달라질 수 있습니다.

그래서 작업마다 Token을 만들고 응답 시 현재 Token과 비교했습니다.

```javascript
let currentToken = null;

const token = Date.now();
currentToken = token;

fetch(target, {
    method: 'POST',
    body: args
})
.then(res => res.text())
.then(() => {
    if (currentToken === token) {
        send_to_iframe_tts_data_tts(
            latest_tts_path
        );
    }
});
```

이 구조가 보장하는 것은 **stale-result 방지**입니다.

### 무엇이 달라졌는가

늦게 도착한 이전 요청의 결과가 현재 UI 상태를 덮어쓰지 않도록 결과 반영 시점에서 한 번 더 유효성을 확인했습니다.

---

# 06. TTS 생성 후 재생까지 하나의 사용자 흐름으로 연결했습니다

TTS 음원이 생성된 뒤 목록에서 다시 찾아 사용자가 직접 재생해야 했던 흐름을 자동화했습니다.

```javascript
if (broadcast_tts_id !== '') {
    document.querySelectorAll(
        '#select_tts option'
    ).forEach((option) => {
        if (option.value === broadcast_tts_id) {
            option.selected = true;

            document
                .getElementById(
                    'tts_button_tts_play'
                )
                ?.click();

            broadcast_tts_id = '';
        }
    });
}
```

### 무엇이 달라졌는가

TTS 생성 → 목록 반영 → 재생이라는 흐름이 연결되어 별도의 수동 선택 단계를 줄일 수 있는 구조가 됐습니다.

실제 사용자 클릭 횟수 변화는 측정 후 추가할 수 있습니다.

---

# 07. 필요한 Zone 상태만 UI에 반영했습니다

실시간 상태 데이터가 여러 Zone으로 들어오는 환경에서는 화면에 필요하지 않은 데이터까지 같은 방식으로 렌더링할 이유가 없었습니다.

그래서 현재 선택한 Zone과 관련된 상태만 UI에 반영하는 방향으로 처리했습니다.

대상은 다음과 같습니다.

- 실시간 레벨 미터
- Zone 상태
- 방송 여부
- 장치별 상태
- UI 속성 변화

### 무엇이 달라졌는가

화면에서 실제로 필요한 상태를 중심으로 렌더링 범위를 좁혔습니다.

CPU/Memory 사용량이나 실제 렌더링 시간 변화는 측정 후 추가할 수 있습니다.

---

# 08. AI Event Preset의 화면 상태를 서버 저장 단위와 연결했습니다

AI Event Preset은 행을 동적으로 추가/삭제하면서 기존 데이터와 신규 데이터를 구분해 저장해야 했습니다.

저는 UI에서 Insert / Update 상태를 구분하고 변경된 데이터를 중심으로 서버에 전달하도록 구성했습니다.

```javascript
function make_insert_ai_event_list_div(type) {
    return `
        <div class="ai_event_row_item ${type} insert_item">
            ...
        </div>
    `;
}

document.addEventListener('click', (event) => {
    if (event.target.classList.contains('xi-plus')) {
        const row =
            event.target.closest('.ai_event_row_item');

        const type =
            row.classList.contains('sound')
                ? 'sound'
                : 'voice';

        const container =
            event.target.closest(
                '.div_ai_event_preset_item'
            );

        container?.insertAdjacentHTML(
            'beforeend',
            make_insert_ai_event_list_div(type)
        );
    }
});
```

저장 흐름은 다음과 같습니다.

```text
Rows
 ↓
Validation
 ↓
insert[] / update[]
 ↓
Fetch
 ↓
Backend
```

### 무엇이 달라졌는가

- Dynamic CRUD
- Insert / Update 상태 구분
- Event Delegation
- Client Validation
- Fetch 기반 비동기 통신

Backend의 저장 구조는 별도 Backend 사례에서 이어집니다.

---

# 09. 다국어 문자열 때문에 깨지는 레이아웃을 CSS로 대응했습니다

프랑스어·러시아어처럼 문자열 길이가 다른 환경에서는 고정 width 레이아웃이 깨질 수 있었습니다.

JavaScript로 매번 크기를 변경하기보다 CSS의 `min-width`, `width: auto`, `overflow-wrap` 등을 이용해 언어별 차이를 선언적으로 처리했습니다.

```css
.div_contents_cell_title {
    width: 100px;
    overflow-wrap: break-word;
}

body[class$="_Français"]
.div_contents_cell_title {
    width: 210px;
}
```

긴 문자열은 목록에서는 ellipsis, 상세 영역에서는 `break-word`를 사용하는 방식으로 용도에 따라 처리했습니다.

### 무엇이 달라졌는가

- 언어별 레이아웃 대응
- 가변 문자열 처리
- 반응형 컴포넌트 구조
- Flexbox 기반 layout

---

# 10. 파일 선택 단계의 검증을 실제 업로드 로직과 분리했습니다

파일 선택, 크기 검사, 확장자 검사, 업로드 요청이 한 handler에 섞여 있던 부분을 `validateFile()`과 upload 처리로 분리했습니다.

```javascript
AuthUIController.validateFile(file);

return AppCore.Http.uploadFile(
    '/upload',
    file
);
```

클라이언트 검증은 1차 UX 경계이며, 서버 검증을 대체하지 않습니다.

---

# 추가 Frontend 경험

다음 사례는 실제 구현 경험으로 보존하되 핵심 프로젝트와 같은 깊이로 반복하지 않습니다.

### UI / Layout
- Dynamic Z-index
- Tooltip / Ellipsis
- Flexbox
- Responsive layout
- Property inspection UI
- 접근성 관련 UI 보정

### Communication
- Promise 기반 iframe status inquiry
- Dynamic sorting + state persistence
- Protocol-adaptive resource resolution
- Metadata exchange

### Security / Data
- DOM 기반 output escaping
- File Upload validation
- Blob 기반 asset retrieval

### Device Integration
- Device-specific UI logic
- BGM/TTS source integration
- Model-specific UI handling

---

# 이 경험에서 보여주는 Frontend 역량

### Real-time State Synchronization
MutationObserver · WebSocket · postMessage · Binary Protocol

### Async UI / TTS
Fetch · Promise · Token · Automatic Playback

### Full-Stack AI Event UI
Dynamic CRUD · insert/update state · Fetch · Validation

### Embedded UI Adaptation
Responsive · i18n · Flexbox

### Cross-context Communication
iframe · postMessage · status inquiry · event propagation

프론트엔드에서 가장 중요한 경험은 UI를 많이 만든 것이 아니라 **실시간 장치 상태가 여러 실행 컨텍스트를 거쳐 사용자 화면에 반영되는 문제를 직접 다뤘다는 것**입니다.
