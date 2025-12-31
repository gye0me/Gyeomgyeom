# 💾 12장 로그와 빅데이터

마이크로서비스 아키텍처(MSA)에서는 각 서비스가 독립적으로 로그를 발생시킴. 이를 개별적으로 관리하면 수집 비용이 크고 유지보수가 어렵기 때문에, **로그 전용 마이크로서비스**를 구축하여 중앙 집중식으로 관리함.

## 12.1 핵심 개념

* **통합 관리:** 모든 서비스의 로그를 한곳으로 모아 효율적으로 관리함.
* **유연한 확장:** 로그 저장소(파일, DB, Elasticsearch 등)가 변경되어도 개별 마이크로서비스를 수정할 필요가 없음.
* **데이터 분석:** 수집된 로그를 빅데이터 솔루션과 연동하여 시각화 및 분석이 가능함.

---

## 12.2 주요 소스 코드

### 1) server.js (공통 서버 클래스)

모든 마이크로서비스의 부모가 되는 클래스로, 로그 서비스로 데이터를 전송하는 로직이 추가됨.

```javascript
'use strict';
const net = require('net');
const tcpClient = require('./client.js');

class tcpServer {
    constructor(name, port, urls) {
        this.logTcpClient = null;  // 로그 서비스 연결용 클라이언트
        this.context = { port: port, name: name, urls: urls };
        this.merge = {};

        this.server = net.createServer((socket) => {
            this.onCreate(socket);

            socket.on('error', (exception) => { this.onClose(socket); });
            socket.on('close', () => { this.onClose(socket); });

            // 데이터 수신 시 로그 전송 및 로직 실행
            socket.on('data', (data) => {
                var key = socket.remoteAddress + ":" + socket.remotePort;
                var sz = this.merge[key] ? this.merge[key] + data.toString() : data.toString();
                var arr = sz.split('¶');
                for (var n in arr) {
                    if (sz.charAt(sz.length - 1) != '¶' && n == arr.length - 1) {
                        this.merge[key] = arr[n];
                        break;
                    } else if (arr[n] == "") {
                        break;
                    } else {
                        this.writeLog(arr[n]); // ➊ 로그 전송
                        this.onRead(socket, JSON.parse(arr[n]));
                    }
                }
            });
        });

        this.server.on('error', (err) => { console.log(err); });
        this.server.listen(port, () => {
            console.log('listen', this.server.address());
        });
    }

    onCreate(socket) { console.log("onCreate", socket.remoteAddress, socket.remotePort); }
    onClose(socket) { console.log("onClose", socket.remoteAddress, socket.remotePort); }

    // Distributor 접속 및 로그 서비스 정보 수신
    connectToDistributor(host, port, onNoti) {
        var packet = { uri: "/distributes", method: "POST", key: 0, params: this.context };
        var isConnectedDistributor = false;

        this.clientDistributor = new tcpClient(
            host, port,
            (options) => { // 접속 완료
                isConnectedDistributor = true;
                this.clientDistributor.write(packet);
            },
            (options, data) => { // 데이터 수신
                // ➋ 로그 마이크로서비스 접속 정보 확인 시 연결 시도
                if (this.logTcpClient == null && this.context.name != 'logs') {
                    for (var n in data.params) {
                        const ms = data.params[n];
                        if (ms.name == 'logs') {
                            this.connectToLog(ms.host, ms.port);
                            break;
                        }
                    }
                }
                onNoti(data);
            },
            (options) => { isConnectedDistributor = false; },
            (options) => { isConnectedDistributor = false; }
        );

        setInterval(() => {
            if (isConnectedDistributor != true) { this.clientDistributor.connect(); }
        }, 3000);
    }

    // 로그 서비스 연결 함수
    connectToLog(host, port) {
        this.logTcpClient = new tcpClient(
            host, port,
            (options) => { },
            (options) => { this.logTcpClient = null; },
            (options) => { this.logTcpClient = null; }
        );
        this.logTcpClient.connect();
    }

    // ➌ 로그 데이터 전송 실행
    writeLog(log) {
        if (this.logTcpClient) {
            const packet = { uri: "/logs", method: "POST", key: 0, params: log };
            this.logTcpClient.write(packet);
        } else {
            console.log(log); // 로그 서비스 미연결 시 콘솔 출력
        }
    }
}

module.exports = tcpServer;

```

### 2) microservice_logs_elasticsearch.js (로그 관리 서비스)

수신한 로그를 파일(`fs`)과 `Elasticsearch`에 동시에 저장하는 실제 서비스 코드임.

```javascript
'use strict';
const cluster = require('cluster');
const fs = require('fs');
const elasticsearch = new require('elasticsearch').Client({
    host: '127.0.0.1:9200',
    log: 'trace'
});

class logs extends require('./server.js') {
    constructor() {
        super("logs", process.argv[2] ? Number(process.argv[2]) : 9040, ["POST/logs"]);

        // ➊ 로그 파일 스트림 생성 (Append 모드)
        this.writestream = fs.createWriteStream('./log.txt', { flags: 'a' });

        this.connectToDistributor("127.0.0.1", 9000, (data) => {
            console.log("Distributor Notification", data);
        });
    }

    onRead(socket, data) {
        const sz = new Date().toLocaleString() + '\t' + socket.remoteAddress + '\t' +
                   socket.remotePort + '\t' + JSON.stringify(data) + '\n';
        
        console.log(sz);
        this.writestream.write(sz);                 // ➋ 파일 저장

        data.timestamp = new Date().toISOString();  // 타임스탬프 추가
        data.params = JSON.parse(data.params);      // 파라미터 변환
        
        // ➌ Elasticsearch 색인 저장
        elasticsearch.index({
            index: 'microservice',
            type: 'logs',
            body: data
        });
    }
}

// 클러스터 모드 실행
if (cluster.isMaster) {
    cluster.fork();
    cluster.on('exit', (worker, code, signal) => {
        console.log(`worker ${worker.process.pid} died`);
        cluster.fork();
    });
} else {
    new logs();
}

```

---

## 12.3 정리 및 결론

* **로그 관리 서비스**를 별도로 운영하면 전체 시스템의 유지보수 비용이 절감됨.
* `fs` 모듈의 스트림 기능을 활용해 효율적인 파일 로그 기록이 가능함.
* **Elasticsearch** 및 **Kibana**와 연동하여 단순 텍스트 로그를 넘어선 빅데이터 분석 환경을 쉽게 구축할 수 있음.
