## 📝 8장. MongoDB 

-----

## 8-1. NoSQL (MongoDB) vs. SQL (MySQL) 비교

MongoDB는 **NoSQL** 데이터베이스의 대표이며, 유연성과 확장성에 중점을 둡니다.

| 특징 | **SQL (MySQL)** | **NoSQL (MongoDB)** |
| :--- | :--- | :--- |
| **데이터 구조** | 규칙적, 정형화된 데이터 입력 (테이블, 로우, 칼럼) | **자유로운 데이터 입력** (컬렉션, 다큐먼트, 필드) |
| **관계** | 테이블 간 **JOIN 지원** | 컬렉션 간 **JOIN 미지원** (Mongoose의 `populate`로 보완) |
| **우선 가치** | **안정성 (Consistency)**, 일관성 | **확장성 (Scalability)**, 가용성 |


-----

## 8-4. MongoDB 기본 관리 명령어 🗃️

MongoDB 셸에서 사용하는 데이터베이스 및 컬렉션 관리 명령어입니다.

| 작업 | 명령어 | 설명 |
| :--- | :--- | :--- |
| **셸 접속** | `mongo admin -u [이름] -p [비밀번호]` | 관리자 권한으로 MongoDB 셸에 접속합니다. |
| **DB 선택/생성** | `> use [데이터베이스명]` | 해당 DB로 전환합니다. 다큐먼트 입력 시 DB가 **자동 생성**됩니다. |
| **현재 DB 확인** | `> db` | 현재 사용 중인 데이터베이스 이름을 출력합니다. |
| **DB 목록 확인** | `> show dbs` | 생성된 DB 목록을 보여줍니다. |
| **컬렉션 생성** | `> db.createCollection('컬렉션명')` | 컬렉션을 명시적으로 생성합니다. (다큐먼트 입력 시 자동 생성도 가능) |
| **컬렉션 목록** | `> show collections` | 현재 DB의 컬렉션 목록을 보여줍니다. |

-----

## 8-5. CRUD 작업하기 (MongoDB Shell) 🛠️

MongoDB는 자바스크립트 객체 문법을 사용하여 쿼리를 수행합니다.

### 8.5.1 Create (생성)

자유로운 데이터 입력이 가능하며, 컬렉션에 필드를 정의할 필요가 없습니다.

```javascript
> db.컬렉션명.save(다큐먼트) 

// 예시
> db.users.save({ name: "nero", age: 32,  married: true, comment: "안녕하세요.", createdAt: new Date() });
```

### 8.5.2 Read (조회)

`find(필터, 프로젝션)` 메서드를 사용합니다.

| 기능 | 명령어 예시 | 설명 |
| :--- | :--- | :--- |
| **모두 조회** | `> db.컬렉션명.find({});` | 컬렉션 내 모든 다큐먼트를 조회합니다. |
| **특정 필드** | `> db.users.find({}, {_id:0, name:1, married: 1});` | `_id`를 제외하고 `name`, `married` 필드만 조회합니다 (1: 포함, 0: 제외). |
| **조건 조회** | `> db.users.find({ age: {$gt: 30}, married: true}, ...);` | `$gt` (초과)와 같은 **특수 연산자**를 사용합니다. |
| **OR 조건** | `> db.users.find({ $or: [{ age: {$gt: 30}}, {married: false}]}, ...);` | `$or` 연산자를 사용해 여러 조건 중 하나만 만족하는 다큐먼트를 조회합니다. |
| **정렬/제한** | `> db.users.find({...}).sort({age: -1}).limit(1).skip(1)` | `sort()`(-1: 내림차순, 1: 오름차순), `limit()`, `skip()` 메서드를 연결하여 사용합니다. |

### 8.5.3 Update (수정)

`update()` 메서드와 **`$set`** 연산자를 사용해 필드를 수정합니다.

```javascript
// name이 'nero'인 다큐먼트를 찾아 comment 필드 수정
> db.users.update({ name: 'nero' }, { $set: { comment: '안녕하세요. 수정한 필드입니다.' } });
// 결과: "nModified": 1 (수정된 다큐먼트 수)
```

### 8.5.4 Delete (삭제)

`remove()` 메서드를 사용합니다.

```javascript
// name이 'nero'인 다큐먼트 삭제
> db.users.remove({ name: 'nero' })
// 결과: 'nRemoved': 1 (삭제된 다큐먼트 수)
```

-----

## 8-6. 몽구스 (Mongoose) 사용하기 🐍

**Mongoose**는 MongoDB를 위한 **ODM (Object-Document Mapping)** 라이브러리입니다. (MySQL의 Sequelize와 같은 역할). Mongoose는 스키마리스 환경의 혼란(뱀)을 통제하고 구조화하는 역할(몽구스)을 합니다.

### 💡 Mongoose의 주요 기능 및 장점

1.  **스키마 기반 데이터 필터링**: Mongoose는 **스키마**를 도입하여 노드 서버 단에서 데이터를 **필터링**하고 유효성을 검사하여 **데이터 무결성**을 보완합니다.
2.  **관계 보완 (`populate`)**: MySQL의 **JOIN** 기능을 `populate` 메서드로 어느 정도 보완하여 관계 데이터를 쉽게 가져올 수 있습니다.
3.  **편의성**: ES2015 **프로미스 문법**과 가독성이 높은 쿼리 빌더를 지원합니다.
4.  **매핑**: 다큐먼트를 자바스크립트 객체와 매핑합니다. (MongoDB는 **릴레이션**이 아닌 **다큐먼트**를 사용하므로 ODM입니다.)

### 8.6.2 스키마 정의하기

`mongoose.Schema` 생성자를 이용해 필드의 타입, 필수 여부 등을 정의합니다.

```javascript
// schemas/user.js
const mongoose = require('mongoose');

const { Schema } = mongoose;
const userSchema = new Schema({
	name: { type: String, required: true, unique: true },
    age: { type: Number, required: true },
    married: { type: Boolean, required: true },
    comment: String,
    createdAt: { type: Date, default: Date.now },
});
   
// 모델 생성: 첫 인수가 'User'이면 컬렉션 이름은 자동으로 'users'가 됩니다.
module.exports = mongoose.model('User', userSchema);

// 컬렉션 이름 변경: 세 번째 인수로 지정 가능 (예: 'user_table')
// mongoose.model('User', userSchema, 'user_table');
```

### 8.6.3 쿼리 수행 및 관계 조회

Express 라우터에서 Mongoose 모델을 사용한 비동기 쿼리 예시입니다.

1.  **데이터 생성 (Create)**

      * Mongoose의 **`모델.create`** 메서드를 사용합니다. 이 과정에서 **스키마에 부합하지 않는 데이터는 자동으로 에러**가 발생합니다.

    <!-- end list -->

    ```javascript
    const user = await User.create({ name: req.body.name, age: req.body.age, married: req.body.married });
    ```

2.  **관계 데이터 조회 (`populate`)**

      * 특정 다큐먼트의 \*\*`ObjectId`\*\*를 기반으로 참조된(`ref`) 컬렉션의 **전체 다큐먼트**를 찾아와 해당 필드에 **치환**해 줍니다.

    <!-- end list -->

    ```javascript
    // commenter 필드에 저장된 ObjectId를 기반으로 User 다큐먼트 전체를 합쳐서 조회
    const comments = await Comment.find({ commenter: req.params.id })
      .populate('commenter');
    ```