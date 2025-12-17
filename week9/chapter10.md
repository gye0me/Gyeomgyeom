# 💾 10장 웹 API 서버 만들기

## 1. API 서버의 이해

* **API (Application Programming Interface):** 다른 서비스의 기능이나 자원을 활용할 수 있도록 돕는 인터페이스.
* **웹 API 서버:** 데이터(게시글, 사용자 정보 등)를 JSON 형식으로 제공하여 다른 클라이언트가 사용할 수 있게 하는 창구.
* **장점:** 무분별한 크롤링 방지 및 서버 부하 감소, 필요한 정보만 선별적 제공 가능.

## 2. 프로젝트 설정 (nodebird-api)

* **목표:** 인증된 사용자에게만 API 호출을 허용하고 호출 할당량을 관리하는 서버 구축.
* **주요 패키지:**
* `uuid`: 고유한 API 비밀키(clientSecret) 생성을 위해 사용.
* `jsonwebtoken`: 토큰 기반 인증(JWT) 구현 시 사용 (본문 하단 흐름상 필요).


* **기본 구조:** 기존 NodeBird 서비스의 모델과 설정을 복사하여 활용하며, API 서버용 포트(예: 8002)를 별도로 할당.

## 3. 핵심 모델 설계 (Domain 모델)

API 사용 권한을 관리하기 위해 도메인 정보를 저장하는 모델을 추가함

```javascript
// models/domain.js 
module.exports = class Domain extends Sequelize.Model {
  static init(sequelize) {
    return super.init({
      host: { type: Sequelize.STRING(80), allowNull: false }, // 도메인 주소
      type: { type: Sequelize.ENUM('free', 'premium'), allowNull: false }, // 요금제 타입
      clientSecret: { type: Sequelize.UUID, allowNull: false }, // API 호출용 비밀키
    }, { ... });
  }
};

```

* **User 모델과의 관계:** 한 사용자가 여러 도메인을 등록할 수 있도록 `1:N` 관계로 설정

## 4. 주요 로직 흐름

1. **로그인 및 도메인 등록:** 사용자는 자신의 도메인을 등록하고 고유한 `clientSecret`을 발급받음
2. **인증 미들웨어:** 요청이 들어오면 등록된 도메인인지, `clientSecret`이 일치하는지 확인함
3. **데이터 응답:** 인증된 요청에 한해 게시글이나 해시태그 데이터를 JSON 형태로 전송함

## 5. 요약 포인트

* **포트 분리:** 기존 웹 서비스(8001)와 API 서버(8002)를 분리하여 운영.
* **보안:** ENUM 타입을 사용하여 서비스 등급을 나누고, UUID로 비밀키를 생성하여 보안 강화.
* **확장성:** 도메인 등록 시스템을 통해 API 사용자를 추적하고 제어할 수 있는 기반 마련.