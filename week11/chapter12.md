# 💾 12장 웹 소켓으로 실시간 데이터 전송하기

실시간성이 중요한 서비스를 구현하기 위한 기술들을 다룬다.

## 12.1 웹 소켓 이해하기

| 기술 | 특징 | 비고 |
| --- | --- | --- |
| **폴링 (Polling)** | 주기적으로 HTTP 요청을 보내 업데이트 확인 | 단순하지만 서버 리소스 낭비 심함 |
| **SSE (Server Sent Events)** | 서버에서 클라이언트로만 데이터를 전송 (단방향) | `EventSource` 객체 사용 |
| **웹 소켓 (Web Socket)** | 한 번 연결되면 유지되는 실시간 양방향 통신 | `WS` 프로토콜 사용 |

### 웹 소켓의 핵심 요약

* **양방향성:** 클라이언트와 서버가 서로 데이터를 주고받음.
* **효율성:** HTTP와 포트를 공유할 수 있으며 연결 오버헤드가 적음.
* **상태:** `CONNECTING` → `OPEN` → `CLOSING` → `CLOSED` (메시지는 `OPEN` 상태에서만 가능).

---

## 12.2 `ws` 모듈로 웹 소켓 사용하기

가벼운 서비스를 구축하거나 웹 소켓 표준에 가깝게 구현할 때 사용함.

### [Server] socket.js

```javascript
const WebSocket = require('ws');

module.exports = (server) => {
    const wss = new WebSocket.Server({ server }); // Express 서버 연결

    wss.on('connection', (ws, req) => {
        const ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
        console.log('새로운 클라이언트 접속', ip);

        ws.on('message', (message) => { console.log(message.toString()); });
        ws.on('error', (error) => { console.error(error); });
        ws.on('close', () => {
            console.log('클라이언트 접속 해제', ip);
            clearInterval(ws.interval);
        });

        // 3초마다 클라이언트로 상태 메시지 전송
        ws.interval = setInterval(() => {
            if (ws.readyState === ws.OPEN) {
                ws.send('서버에서 메시지를 보냅니다.');
            }
        }, 3000);
    });
};

```

---

## 12.3 Socket.IO 사용하기

웹 소켓을 지원하지 않는 브라우저에서도 **HTTP 폴링으로 자동 강하(Fallback)**하여 호환성을 확보함.

### 1) 서버 로직 (socket.js)

```javascript
const SocketIO = require('socket.io');

module.exports = (server) => {
    const io = SocketIO(server, { path: '/socket.io' });

    io.on('connection', (socket) => {
        const req = socket.request;
        const ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
        
        socket.on('disconnect', () => {
            console.log('접속 해제', socket.id);
            clearInterval(socket.interval);
        });
        
        socket.on('reply', (data) => { console.log(data); }); // 사용자 정의 이벤트
        
        socket.interval = setInterval(() => {
            socket.emit('news', 'Hello Socket.IO'); // 'news'라는 키로 데이터 전송
        }, 3000);
    });
};

```

### 2) 클라이언트 구현 (index.html)

```html
<script src="/socket.io/socket.io.js"></script>
<script>
    const socket = io.connect('http://localhost:8005', {
        path: '/socket.io',
        transports: ['websocket'], // 처음부터 웹소켓만 사용하고 싶을 때 설정
    });

    socket.on('news', (data) => {
        console.log(data);
        socket.emit('reply', 'Hello Node.JS');
    });
</script>

```

---

## 12.5 미들웨어와 소켓 연결하기

### 미들웨어와 세션 공유

* **세션 쿠키 직접 설정:** 서버 내부(socket.js)에서 axios 요청을 보낼 때는 브라우저와 달리 쿠키가 자동 포함되지 않음. `connect.sid` 세션 쿠키를 직접 헤더에 설정해야 요청자를 판단할 수 있음.
* **Express 연결:** `app.set('io', io)`를 사용하여 라우터에서 `req.app.get('io')`로 소켓 객체를 가져올 수 있음.

### Socket.IO 주요 API

* **특정인에게 전송:** `socket.to(소켓_아이디).emit(이벤트, 데이터)`
* **나를 제외한 전체 전송:** `socket.broadcast.emit(이벤트, 데이터)`
* **특정 방을 제외한 전송:** `socket.broadcast.to(방아이디).emit(이벤트, 데이터)`

---

## 12.7 핵심 정리

1. **포트 공유:** 웹 소켓과 HTTP는 동일한 포트를 사용함으로 별도 설정이 불필요함.
2. **브라우저 호환성:** Socket.IO는 구형 브라우저 대응이 용이함.
3. **네임스페이스와 방:** 데이터를 필요한 사용자(특정 그룹)에게만 타겟팅해서 보낼 수 있음.
4. **라우터 연동:** 복잡한 DB 조작이 동반되는 경우, 소켓 직접 통신보다 **HTTP 라우터를 거친 후 소켓으로 알림을 보내는 방식**이 더 안정적임.

