# KingCong TTS — Full System Architecture

## 1. Tổng quan toàn hệ thống

```mermaid
flowchart TD
    subgraph CLIENTS["📱 Clients"]
        WEB["🌐 Web Browser\nkingcongstudio.com/tts3"]
        TOOL["🖥️ Electron Tool\ntoolkingcong"]
    end

    subgraph CPANEL["⚙️ cPanel — kingcongstudio.com"]
        PHP["tts3.php\nAPI Gateway"]
        WEBHOOK["webhook_tts3.php\nNhận callback từ Provider"]
        GET_TOKEN["get_wss_token.php\nPhát token 1 lần"]
        GET_VOICES["get_voices.php\nDanh sách giọng"]
        SRT_BG["tts3.php?action=generate_srt_for_tts\nBackground SRT job"]
    end

    subgraph VPS["🖧 VPS — wss.kingcongstudio.com"]
        subgraph WSS_SERVER["WSS Server :8443"]
            ROUTE_TTS["/tts — WebSocket\nroutes/tts.js"]
            ROUTE_GUIT["/guit — WebSocket\nroutes/guit.js"]
            PROXY_ENDPOINT["/proxy/ai84 HTTP POST - server.js"]
        end
        subgraph PROXY_LAYER["Proxy Layer"]
            KIOT["KiotProxy\nRotate IP mỗi 5 phút\n(Key: K658...)"]
            CF_WORKERS["CF Workers (random 1/3)\nvuavu / vuavuphaiemkhong / domixi\n.buicongxd11.workers.dev"]
        end
    end

    subgraph PROVIDERS["🎙️ TTS Providers"]
        AI84["AI84\napi.ai84.pro\n11Labs + Minimax"]
        AI33["AI33\napi.ai33.pro\n11Labs + Minimax"]
        KC["KingCong\nmaziao.com\nVN voices"]
        EDGE["Edge TTS\nMicrosoft Free"]
    end

    subgraph DB["🗄️ MySQL — LongVân\ncpanelhcm1.longvan.net"]
        T1[("tts_kingcong\ntask, status, audio_url, credits")]
        T2[("task_distribution\napi_key_id, server_type")]
        T3[("api_keys_pool\nkey, load, credits")]
        T4[("wss_tokens\nshort-lived 1-time")]
        T5[("Users\ncredits3, api_key")]
        T6[("voice_cloning\nclone voice → server_type")]
    end

    WEB -->|"① GET token"| GET_TOKEN
    GET_TOKEN -->|INSERT| T4
    WEB -->|"② WSS /tts + token"| ROUTE_TTS
    WEB -->|"fallback AJAX"| PHP

    TOOL -->|"① POST create_speech\nX-Client: electron-tool"| PHP
    TOOL -->|"② WSS /tts\n?api_key=..."| ROUTE_TTS

    PHP -->|"callAi84Proxy()"| PROXY_ENDPOINT
    PHP -->|"curl direct"| AI33
    PHP -->|"curl direct"| KC
    PHP -->|"curl direct"| EDGE

    PROXY_ENDPOINT --> KIOT & CF_WORKERS
    KIOT -->|"70%"| AI84
    CF_WORKERS -->|"30%"| AI84
    KIOT -->|"403 fallback"| CF_WORKERS

    AI84 -->|"webhook done"| WEBHOOK
    WEBHOOK -->|"UPDATE status=done"| T1

    ROUTE_TTS -->|"poll DB 1s"| T1
    ROUTE_TTS -->|"poll API"| KC
    ROUTE_TTS -->|"poll API"| AI33
    ROUTE_TTS -->|"push progress"| WEB
    ROUTE_TTS -->|"push progress"| TOOL

    PHP --> T1 & T2 & T3 & T4 & T5 & T6
    SRT_BG -->|"curl self-call"| PHP
```

---

## 2. Flow Web — Chi tiết từng bước

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant JS as tts3.js (Browser)
    participant PHP as tts3.php (cPanel)
    participant DB as MySQL DB
    participant VPS as WSS Server
    participant PROXY as Proxy Layer
    participant AI as AI Provider

    U->>JS: Nhập text + chọn voice + bấm Generate

    JS->>PHP: GET /ajaxs/get_wss_token.php
    PHP->>DB: INSERT wss_tokens (token, user_id, expire 60s)
    PHP-->>JS: { status: success, token: "abc123" }

    JS->>VPS: WebSocket wss://wss.kingcongstudio.com:8443/tts
    Note over JS,VPS: Kết nối WebSocket

    VPS->>DB: SELECT wss_tokens WHERE token=abc123
    VPS->>DB: DELETE token (dùng 1 lần, không reuse)
    VPS->>DB: SELECT Users WHERE id → xác thực user

    JS->>VPS: send { action: create_speech, apikey: token, payload }
    Note over VPS: Không xử lý create_speech ở WSS<br/>WSS chỉ watch_task<br/>→ tts3.js gọi AJAX tạo task

    JS->>PHP: AJAX POST /ajaxs/tts3.php action=create_speech
    Note over PHP: Xem Flow PHP chi tiết bên dưới
    PHP-->>JS: { task_id, cost }

    JS->>VPS: send { action: watch_task, task_id }

    loop Poll DB mỗi 1 giây
        VPS->>DB: SELECT tts_kingcong + task_distribution WHERE task_id
        alt Provider = AI84
            Note over VPS: KHÔNG poll AI84 trực tiếp<br/>(tránh block IP VPS)
            Note over VPS: Tăng fake progress +3~8%/s<br/>Chờ webhook cập nhật DB
        else Provider = AI33
            VPS->>AI: GET api.ai33.pro/v1/task/{id}
            AI-->>VPS: { status, progress, audio_url }
        else Provider = KingCong
            VPS->>AI: POST maziao.com/api/tts/status
            AI-->>VPS: { status, resultUrl, srtUrl }
        end
        VPS->>DB: UPDATE tts_kingcong SET progress, status
        VPS-->>JS: push { action: task_update, progress, status }
        JS->>JS: Update progress bar UI
    end

    AI->>PHP: Webhook POST /ajaxs/webhook_tts3.php\n{ task_id, audio_url, status: done }
    PHP->>DB: UPDATE tts_kingcong SET status=done, audio_url

    VPS->>DB: SELECT → status=done
    VPS-->>JS: push { status: done, audio_url, srt_url, cost }
    JS->>JS: Render audio player + update credits UI
    JS->>VPS: send { action: unwatch_task }
    VPS->>VPS: clearInterval, xóa khỏi activeWatches
```

---

## 3. Flow Tool Electron — Chi tiết

```mermaid
sequenceDiagram
    participant R as renderer/js/tool.js
    participant M as main.js (Electron)
    participant PHP as tts3.php (cPanel)
    participant DB as MySQL DB
    participant VPS as WSS Server
    participant AI as AI Provider

    R->>M: electronAPI.apiRequest(url, { action: create_speech, ...params })
    Note over M: Phát hiện Header X-Client=electron-tool<br/>Rate limit 100 req/min (thay vì 20)

    M->>PHP: POST /ajaxs/tts3.php\nHeader: X-Source=electron-tool\nBody: FormData (action, text, voice_id, ...)

    Note over PHP: Xem Flow PHP chi tiết bên dưới
    PHP->>DB: INSERT tts_kingcong + UPDATE credits3
    PHP-->>M: { status: success, task_id, cost }

    M->>VPS: WebSocket wss://wss.kingcongstudio.com/tts?api_key=PERMANENT_KEY
    Note over VPS: Auth bằng api_key cố định<br/>(khác Web dùng short-lived token)
    VPS->>DB: SELECT Users WHERE api_key=?

    M->>VPS: send { action: watch_task, task_id }

    loop Push mỗi 1 giây
        VPS->>DB: SELECT tts_kingcong
        VPS-->>M: { progress, status }
        M->>R: onProgress(progress) → update UI
    end

    VPS-->>M: { status: done, audio_url, srt_url }
    M->>R: resolve(response) → render audio
    M->>VPS: ws.close()
```

---

## 4. PHP tts3.php — Logic chọn Server & API Key

```mermaid
flowchart TD
    START[POST action=create_speech] --> AUTH{Session hợp lệ?}
    AUTH -->|No| ERR401[401 Unauthorized]
    AUTH -->|Yes| RATELIMIT{Rate limit check\nWeb: 20 req/min\nTool: 100 req/min}
    RATELIMIT -->|Vượt| ERR429[429 Too Many Requests]
    RATELIMIT -->|OK| COST[Tính credits\nchar × base_rate × cost_factor × clone_multiplier]
    
    COST --> COST_NOTE["base_rate: 1.12 (normal) / 0.35 (EdgeTTS)\ncost_factor: theo model\nclone_multiplier: x1.15 nếu là clone voice\nSRT/transcript: x1.15"]
    COST_NOTE --> CHECK_BAL{credits3 đủ?}
    CHECK_BAL -->|No| ERR402[402 Không đủ credits]
    CHECK_BAL -->|Yes| VOICE_TYPE{Voice type?}

    VOICE_TYPE -->|Clone voice\nCó trong voice_cloning| CLONE_SERVER{Nhiều server?}
    CLONE_SERVER -->|1 server| USE_CLONE_KEY[Dùng api_key_id của clone]
    CLONE_SERVER -->|AI33 + AI84| CLONE_RAND{SRT upload?}
    CLONE_RAND -->|Yes| FORCE_AI33[Force AI33]
    CLONE_RAND -->|No| RAND7030{Random 1-100}
    RAND7030 -->|≤70| USE_AI33[AI33 70%]
    RAND7030 -->|>70| USE_AI84[AI84 30%]

    VOICE_TYPE -->|System voice\nKhông có clone| LB[Load Balancer\ngetAvailableServer]
    LB --> LB_SERVER{Server available?}
    LB_SERVER -->|ai33 / ai84| GET_KEY[getAvailableApiKey\ntheo load + credits]
    LB_SERVER -->|NULL| FB1{Thử server còn lại?}
    FB1 -->|OK| GET_KEY
    FB1 -->|Vẫn NULL| FB2[Final fallback: KingCong]

    GET_KEY --> SEND[Gửi request tới Provider]
    USE_CLONE_KEY --> SEND
    FORCE_AI33 --> SEND
    USE_AI33 --> SEND
    USE_AI84 --> SEND
    FB2 --> SEND

    SEND --> PROV{Provider?}
    PROV -->|elevenlabs/minimax AI84| PROXY[callAi84Proxy\n→ POST /proxy/ai84 trên VPS]
    PROV -->|elevenlabs/minimax AI33| DIRECT33[curl api.ai33.pro]
    PROV -->|kingcong| DIRECTKC[curl maziao.com]
    PROV -->|edge_tts| DIRECTEDGE[curl localhost:5001]

    PROXY --> DB_INSERT[INSERT tts_kingcong\nUPDATE credits3 -cost\nINSERT task_distribution]
    DIRECT33 --> DB_INSERT
    DIRECTKC --> DB_INSERT
    DIRECTEDGE --> DB_INSERT

    DB_INSERT --> SRT_CHECK{with_transcript\nor export_srt?}
    SRT_CHECK -->|Yes| SRT_BG[Background job\ncurl self → generate_srt_for_tts]
    SRT_CHECK -->|No| RESP[Return { task_id, cost }]
    SRT_BG --> RESP
```

---

## 5. Proxy Layer — AI84

```mermaid
flowchart LR
    PHP["tts3.php\ncPanel"] -->|"curl POST\nBearer kc_wss_internal_9x7q2mT4vLpR8nW3"| VPS["server.js\n/proxy/ai84"]

    VPS --> KIOT_CHECK{KiotProxy\ncache < 5 phút?}
    KIOT_CHECK -->|No / expired| KIOT_NEW["GET api.kiotproxy.com/new\n?key=K658..."]
    KIOT_NEW -->|fail < 60s| KIOT_CURR["GET api.kiotproxy.com/current"]
    KIOT_NEW & KIOT_CURR --> KIOT_CACHE["global.cachedProxyAgent\nglobal.lastKiotUpdate"]
    KIOT_CHECK -->|Yes| KIOT_CACHE

    KIOT_CACHE --> RAND{Math.random}
    RAND -->|"< 0.7 (70%)\n+ not failed"| KIOT_SEND["Fetch via HttpsProxyAgent\ntarget_url + api_key"]
    RAND -->|"≥ 0.7 (30%)"| CF_PICK

    KIOT_SEND -->|"HTTP 403"| MARK_FAIL["global.kiotFailed = true"] --> CF_PICK

    CF_PICK["Random 1 trong 3\nCF Workers"] --> CF1["vuavu...workers.dev"]
    CF_PICK --> CF2["vuavuphaiemkhong...workers.dev"]
    CF_PICK --> CF3["domixi...workers.dev"]

    CF1 & CF2 & CF3 -->|"Verify X-Proxy-Secret\nForward request"| AI84["api.ai84.pro"]
    KIOT_SEND --> AI84

    AI84 -->|"{ success, task_id }"| VPS
    VPS --> PHP
```

---

## 6. Database Schema — Các bảng liên quan TTS

```mermaid
erDiagram
    Users {
        int id PK
        string taikhoan
        string api_key
        bigint credits3
        string email
    }
    tts_kingcong {
        int id PK
        int user_id FK
        string task_id UK
        string provider
        string model_id
        string voice_id
        string status
        int credit_cost
        string audio_url
        string srt_url
        int progress
        string error_message
        datetime created_at
    }
    task_distribution {
        int id PK
        string task_id FK
        int api_key_id FK
        string server_type
        string status
        datetime completed_at
    }
    api_keys_pool {
        int id PK
        string api_key
        string server_type
        int current_load
        int max_concurrent
        bigint credits_remaining
        bool is_active
    }
    wss_tokens {
        int id PK
        int user_id FK
        string token UK
        datetime created_at
    }
    voice_cloning {
        int id PK
        int user_id FK
        string voice_id
        string server_type
        int api_key_id FK
        string status
    }

    Users ||--o{ tts_kingcong : "user_id"
    Users ||--o{ wss_tokens : "user_id"
    Users ||--o{ voice_cloning : "user_id"
    tts_kingcong ||--o| task_distribution : "task_id"
    task_distribution }o--|| api_keys_pool : "api_key_id"
    voice_cloning }o--|| api_keys_pool : "api_key_id"
```
