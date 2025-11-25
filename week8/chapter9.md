## 💾 9장. 익스프레스로 SNS 만들기



-----

### 1\. 프로젝트 초기 구조 및 환경 설정

| 항목 | 내용 | 예시 |
| :--- | :--- | :--- |
| **프로젝트 구조** | `nodebird` 폴더를 생성하고 `package.json`을 작성해야 함. | `{"scripts": {"start": "nodemon app"}}` |
| **템플릿 엔진** | \*\*Nunjucks(넌적스)\*\*를 사용해 뷰 렌더링을 처리함. | `app.set('view engine', 'html');` |
| **포트 설정** | 서버 포트를 **8001번**으로 설정하여 연결함. | `app.set('port', process.env.PORT \|\| 8001);` |
| **비밀 키 관리** | **`.env` 파일**을 사용하여 비밀 키, DB 정보 등을 안전하게 관리해야 함. | `COOKIE_SECRET=nodebirdsecretkey` |

-----

### 2\. 서버 구조 및 미들웨어

  * **컨트롤러 분리**: 라우터의 비즈니스 로직(DB 조회, 렌더링 등)을 **`controllers` 폴더**로 분리하여 코드를 관리함.
  * **미들웨어 역할**:
      * **404 처리**: 요청 경로를 찾지 못하는 상황을 처리하는 미들웨어를 마지막에 배치해야 함.
      * **에러 처리**: 404 미들웨어 뒤에 위치시켜 서버 에러를 처리함.

<!-- end list -->

```javascript
// app.js (미들웨어 및 라우터 설정)
app.use('/', pageRouter);
app.use((req, res, next) => { // 404 미들웨어
    const error =  new Error(`${req.method} ${req.url} 라우터가 없습니다.`);
    error.status = 404;
    next(error);
});
app.use((err, req, res, next) => { // 에러 핸들러
    res.locals.message = err.message;
    res.locals.error = process.env.NODE_ENV !== 'production' ? err : {};
    res.status(err.status || 500).render('error');
});
```

-----

### 3\. 데이터베이스 모델 및 관계 설정 (Sequelize)

MySQL과 Sequelize를 사용하여 데이터를 정의하고 관계를 설정해야 함.

#### 3.1. 모델 간의 관계 설정 예시

| 관계 유형 | 모델 | 관계 정의 (예시) | 설명 |
| :--- | :--- | :--- | :--- |
| **1:N** | `User` : `Post` | `User.hasMany(Post); Post.belongsTo(User);` | 사용자 한 명이 게시글을 여러 개 작성할 수 있음. |
| **N:M** (게시글-태그) | `Post` : `Hashtag` | `Post.belongsToMany(Hashtag, { through: 'PostHashtag' });` | 게시글과 해시태그는 다대다 관계임. |
| **N:M** (팔로우) | `User` : `User` | `User.belongsToMany(User, { foreignKey: 'followingId', as: 'Followers' });` | 같은 테이블 간의 다대다 관계는 **`as` 키**를 사용하여 외래 키를 구별해야 함. |

#### 3.2. Sequelize 동기화

```javascript
// models/index.js (DB 연결 및 동기화)
db.sequelize.sync({ force: false })
    .then(() => { console.log('데이터베이스 연결 성공'); })
    .catch((err) => { console.error(err); });
```

-----

### 4\. 인증 시스템 구현 (Passport)

사용자 인증 및 로그인 처리는 **Passport** 모듈을 사용하여 구현해야 함.

#### 4.1. 직렬화/역직렬화 (세션 처리)

```javascript
// passport/index.js
passport.serializeUser((user, done) => {
    done(null, user.id); // 사용자 객체에서 아이디만 추려 세션에 저장함.
});

passport.deserializeUser((id, done) => {
    User.findOne({ where: { id } })
        .then(user => done(null, user)) // 요청 시마다 아이디로 사용자 객체 정보를 가져옴.
        .catch(err => done(err));
});
```

#### 4.2. 로그인 제어 미들웨어

로그인 상태에 따른 접근을 제어하기 위해 미들웨어를 생성하여 사용함.

```javascript
// middlewares/index.js
exports.isLoggedIn = (req, res, next) => {
    if (req.isAuthenticated()) { // Passport가 제공하는 메서드
        next(); // 로그인 되어 있으면 다음 미들웨어로
    } else {
        res.redirect('/?error=로그인이 필요합니다.');
    }
};

exports.isNotLoggedIn = (req, res, next) => {
    if (!req.isAuthenticated()) {
        next(); // 로그인 되어 있지 않으면 다음 미들웨어로
    } else {
        const message = encodeURIComponent('로그인한 상태입니다.');
        res.redirect(`/?error=${message}`);
    }
};
```

#### 4.3. 카카오 로그인 전략

```javascript
// passport/kakaoStrategy.js
new KakaoStrategy({
    clientID: process.env.KAKAO_ID, // .env 파일에서 불러옴
    callbackURL: '/auth/kakao/callback',
}, async (accessToken, refreshToken, profile, done) => {
    // 사용자 조회 후 없으면 회원가입, 있으면 로그인 처리 로직
});
```

-----

### 5\. 파일 업로드 및 게시글 작성

  * **`Multer` 패키지**: 파일(사진) 업로드를 처리하기 위해 사용함.
  * **업로드 경로**: 업로드된 파일은 서버의 특정 폴더에 저장되고, 해당 파일 경로는 DB에 저장됨.

<!-- end list -->

```javascript
// post.js 라우터 (Multer 적용)
const upload = multer({
    storage: multer.diskStorage({
        destination(req, file, done) {
            done(null, 'uploads/'); // 파일이 저장될 폴더
        },
        filename(req, file, done) {
            const ext = path.extname(file.originalname);
            done(null, path.basename(file.originalname, ext) + Date.now() + ext);
        },
    }),
    limits: { fileSize: 5 * 1024 * 1024 }, // 5MB 제한
});
```