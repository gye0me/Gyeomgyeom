# 💾 11장 노드 서비스 테스트

## 11.1 테스트 준비하기

개발 완료 후 서비스가 의도대로 동작하는지 확인하기 위해 **Jest** 프레임워크를 사용한다.

### 설치 및 설정

1. **패키지 설치**:
```bash
$ npm i -D jest

```


2. **package.json 스크립트 등록**:
```json
{
  "scripts": {
    "start": "nodemon app",
    "test": "jest",
    "coverage": "jest --coverage"
  }
}

```


3. **파일명 규칙**: `파일명.test.js` 또는 `파일명.spec.js` (예: `middlewares.test.js`)

---

## 11.2 유닛 테스트 (Unit Test)

함수나 미들웨어 등 개별 단위 로직을 독립적으로 검증한다.

### 핵심: 모킹 (Mocking)

* **정의**: 실제 객체 대신 가짜 객체/함수를 주입하는 행위.
* **메서드 체이닝 구현**: `res.status(403).send('...')`를 검증하려면 `res.status`가 다시 `res`를 반환하도록 설정해야 한다.
* `status: jest.fn(() => res)`



### 미들웨어 유닛 테스트 예시

```javascript
const { isLoggedIn } = require("./middlewares");

describe('isLoggedIn 미들웨어 테스트', () => {
    const res = {
        status: jest.fn(() => res),
        send: jest.fn(),
    };
    const next = jest.fn();

    test('로그인되어 있으면 next를 호출해야 함', () => {
        const req = { isAuthenticated: jest.fn(() => true) };
        isLoggedIn(req, res, next);
        expect(next).toBeCalledTimes(1);
    });
    
    test('로그인되어 있지 않으면 403 에러를 응답해야 함', () => {
        const req = { isAuthenticated: jest.fn(() => false) };
        isLoggedIn(req, res, next);
        expect(res.status).toBeCalledWith(403);
        expect(res.send).toBeCalledWith('로그인 필요');
    });
});

```

---

## 11.3 테스트 커버리지 (Test Coverage)

전체 코드 중 테스트가 수행된 코드의 비율을 분석한다.

### 실행 명령어

```bash
$ npm run coverage

```

### 지표 해석

* **% Stmts**: 구문 실행 비율
* **% Branch**: `if`문 등의 분기점 실행 비율 (핵심 지표)
* **% Funcs**: 함수 실행 비율
* **% Lines**: 코드 줄 수 실행 비율
* **Uncovered Line**: 테스트되지 않은 코드의 라인 번호

---

## 11.4 통합 테스트 (Integration Test)

여러 미들웨어, 라이브러리, 데이터베이스가 유기적으로 잘 작동하는지 테스트한다.

### Supertest 활용

1. **설치**: `$ npm i -D supertest`
2. **구조 분리**: `app.js`(로직)와 `server.js`(리스닝)를 분리하여 테스트 환경에서 서버를 직접 제어한다.
3. **테스트 코드**:

```javascript
const request = require('supertest');
const { sequelize } = require('../models');
const app = require('../app');

beforeAll(async () => {
    await sequelize.sync(); // 테스트 전 DB 동기화
});

describe('POST /auth/login', () => {
    test('로그인 성공 시 메인으로 리다이렉트 된다', (done) => {
        request(app)
            .post('/auth/login')
            .send({ email: 'test@test.com', password: 'password' })
            .expect('Location', '/')
            .expect(302, done);
    });
});

```

---

## 11.5 부하 테스트 (Load Test)

서버가 대량의 동시 요청을 얼마나 안정적으로 처리하는지 측정한다.

### Artillery 사용

* **설치**: `$ npm i -D artillery`
* **빠른 실행**:
```bash
$ npx artillery quick --count 100 -n 50 http://localhost:8001

```



### 주요 지표: Request Latency (응답 지연 시간)

* **Median**: 전체 응답의 중간값 (평균 속도)
* **p95 / p99**: 상위 95%, 99% 사용자가 느끼는 지연 시간.
* **분석**: Median과 p95의 격차가 작을수록 응답 속도가 균일하고 안정적인 서버이다.

### 지연 시간 발생 시 해결 방안

1. **Scale-up**: 서버 하드웨어 사양 업그레이드
2. **Scale-out**: 서버 대수 증설 (로드 밸런싱)
3. **코드 최적화**: 효율적인 알고리즘 및 DB 인덱싱 적용

