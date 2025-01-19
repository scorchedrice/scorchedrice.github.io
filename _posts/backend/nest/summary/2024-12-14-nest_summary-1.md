---
title: "[Nest.js 정리] 로그인과정까지 흐름"
date: 2024-12-14 03:00:00 +0900
categories: [Backend, Nest정리]
tags: [nest.js, token, session, jwt, summary, auth]
render_with_liquid: false
---

무작정 강의를 따라치기보단 어느 정도까지 진행되었을 때, 혼자 해보는 것이 좋다고 판단하여 진행

# 해당 과정까지 개발 순서

1. 프로젝트의 시작
2. 어떤 테이블을 작성할지 설계
   1. 게시물 관련 내용을 담을 post table 작성
   2. user 정보를 담을 users table 작성
3. 이를 위해 어떤식으로 구현해야하는가
   1. service.ts, controller.ts를 각 모듈별로 설계
4. 설계를 바탕으로 모듈 생성
5. 각 모듈별 controller.ts, module.ts, service.ts 파일 수정

## 1. 프로젝트의 시작

```shell
nest new jw_sns
```
- 나의 경우엔 jw_sns라는 이름의 프로젝트를 시작한다.
- 또한 필요한 것들을 다운로드한다.
  - typeORM, @nestjs/typeorm
- DB를 연결한다.
  - postgres사용시 yarn add pg
  - docker-compose.yaml 작성

```yaml
services:
  postgres:
    image: postgres:15
    restart: always
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: jw_sns-postgres
```

## 2. 어떤 테이블을 만들어야하는가

우선 어떤 기능을 구현해야하는지 하나하나 따져보자.

1. 게시글 작성 관련
   1. 게시글 - (작성자, 좋아요 수, 댓글 수, 제목, 내용)
      1. 게시글과 작성자간 OneToMany (게시글 입장에선 ManyToOne)
2. 유저 관련
   1. 해당 유저의 이메일, 닉네임, 패스워드, 작성한 게시글, 좋아요 누른 것
      1. 작성한 게시글은 OneToMany관계

자, 이제 module을 생성하고 entity를 만들어보자.

```shell
nest g resource
// 이후 적합한 module을 만들자 (users, posts)
```

### a. posts.entity.ts

- 고려해야하는 것들
  - author의 경우 usersModel과 ManyToOne 관계이다.

```ts
@Entity()
export class PostsModel {
  @PrimaryGeneratedColumn()
  id: number;

  @ManyToOne(() => UsersModel, (user) => user.posts)
  authorId: number;
  // 주의! ManyToOne으로 엮이는 경우 TypeORM은 자동으로 authorId 칼럼을 생성
  // 따라서 author : string과 같은 경우 오류발생!
  // 관계형 DB의 특징이기에 준수해야함.

  @Column()
  title: string;

  @Column()
  content: string;

  @Column()
  likeCount: number;

  @Column()
  commentCount: number;
}
```

이후 엔티티를 해당 모듈에서 사용하기 위해 module.ts를 수정하여 TypeORM이 해당 엔티티를 인식할 수 있도록 하자.

***module에 imports 해야 service (핵심 로직 구현)에서 활용 가능하다.***

```ts
@Module({
  imports: [TypeOrmModule.forFeature([PostsModel])],
  controllers: [PostsController],
  providers: [PostsService],
})
export class PostsModule {}
```

### b. users.entity.ts

- 고려해야하는 것
  - nickname, email은 고유값이여야한다. (옵션)
  - nickname의 길이를 제한한다 (20자로 하자 일단)
  - 작성한 게시글 정보를 postsModel과 OneToMany 관계를 형성해야한다.

```ts
@Entity()
export class UsersModel {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({
    unique: true,
  })
  email: string;

  @Column({
    unique: true,
    length: 20,
  })
  nickname: string;

  @Column()
  password: string;

  @OneToMany(() => PostsModel, (post) => post.authorId)
  posts: PostsModel[];

  @Column({
    type: 'enum',
    enum: Object.values(Roles),
    default: Roles.USER,
  })
  role: Roles;
}

```

users또한 정상작동하도록 module.ts 파일을 수정하자.

## 3. 어떤식으로 구현할까

이것만은 정상 작동해야한다!

1. 게시글 작성
   1. 회원가입을 통해 로그인한 유저가 정상적으로 게시글을 작성할 수 있다.
2. 게시글 확인
   1. 작성된 게시글이 무엇이 있는지 유저는 확인할 수 있어야한다.

### posts

다음과 같은 기능이 구현되어야

1. 게시글의 작성
   1. 작성자, 제목, 내용을 입력하면 자동으로 게시글 생성
      1. 이 때 좋아요 / 댓글 수는 default로 0 부여
2. 게시글 수정
   1. id에 해당하는 게시글을 찾고 존재한다면 다음 과정 진행
   2. 제목, 내용을 입력해도 안해도 동작할 수 있도록 (선택적 업데이트)
3. 게시글 삭제
   1. id에 해당하는 게시글이 존재한다면 이를 삭제
4. 게시글 확인 (전체, 하나 상세)
   1. 하나의 포스트를 찾는 경우 id로 찾는다. 없으면 에러

#### posts.service.ts

작성한 테이블에 접근해야하므로 module.ts 수정을 통해 의존성 주입이 가능하도록 한다.

```ts
@Module({
  imports: [TypeOrmModule.forFeature([UsersModel])],
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

작성한 service.ts코드는 다음과 같다.

```ts
@Injectable()
export class PostsService {
  constructor(
    @InjectRepository(PostsModel)
    private postsRepository: Repository<PostsModel>,
  ) {}

  // 1. 모든 게시글 확인하기
  async getAllPosts() {
    return this.postsRepository.find();
  }

  // 2. 특정 게시글 확인하기 (id로)
  async getPost(id: number) {
    const post = await this.postsRepository.findOne({
      where: {
        id,
      },
    });
    if (!post) {
      throw new NotFoundException('해당 게시글을 찾을 수 없습니다.');
    }
    return post;
  }

  // 3. 게시글 작성하기
  // authorId: number, title: string, content: string
  async createPost(
    postData: Pick<PostsModel, 'authorId' | 'title' | 'content'>,
  ) {
    const post = this.postsRepository.create({
      ...postData,
      likeCount: 0,
      commentCount: 0,
    });
    const newPost = await this.postsRepository.save(post);
    return newPost;
  }

  // 4. 게시글 수정하기
  async updatePost(
    id: number,
    updateData: Partial<Pick<PostsModel, 'title' | 'content'>>,
  ) {
    const post = await this.postsRepository.findOne({
      where: {
        id,
      },
    });
    if (!post) {
      throw new NotFoundException('해당 게시글을 찾을 수 없습니다.');
    }

    if (updateData.title) {
      post.title = updateData.title;
    }

    if (updateData.content) {
      post.content = updateData.content;
    }

    const newPost = await this.postsRepository.save(post);
    return newPost;
  }

  // 5. 게시글 삭제하기
  async deletePost(id: number) {
    const post = await this.postsRepository.findOne({
      where: {
        id,
      },
    });
    if (!post) {
      throw new NotFoundException('해당 게시글을 찾을 수 없습니다.');
    }
    await this.postsRepository.delete(id);
    return `${id}번 게시글을 삭제했습니다.`;
  }
}
```

#### posts.controller.ts

service.ts에서 핵심 로직을 작성했으니, 요청이 왔을 때 안내하는 역할인 controller를 수정해야한다.

```ts
@Controller('posts')
export class PostsController {
  constructor(private readonly postsService: PostsService) {}

  // 1. 모든 게시글을 보기위한 요청
  @Get()
  getPosts() {
    return this.postsService.getAllPosts();
  }

  // 2. 특정 게시글 (id)을 확인하기 위한 요청
  @Get(':id')
  getPost(@Param('id', ParseIntPipe) id: number) {
    return this.postsService.getPost(id);
  }

  // 3. 게시글을 작성하기 위한 요청
  // Body값으로 작성자, 제목, 내용을 받으면 이를 기반으로 게시글을 작성한다.
  @Post()
  postPosts(
    @Body('authorId') authorId: number,
    @Body('title') title: string,
    @Body('content') content: string,
  ) {
    return this.postsService.createPost({ authorId, title, content });
  }

  // 4. 게시글 수정
  @Put(':id')
  updatePosts(
    @Param('id', ParseIntPipe) id: number,
    @Body('title') title?: string,
    @Body('content') content?: string,
  ) {
    return this.postsService.updatePost(id, { title, content });
  }

  // 5. 게시글 삭제
  @Delete(':id')
  deletePost(@Param('id', ParseIntPipe) id: number) {
    return this.postsService.deletePost(id);
  }
}
```

중간중간 parseIntPipe를 활용해서 url에서 오는 string을 number로 바꿔 요청을 진행했다.
또한 Pick, Partial을 활용하여 타입 안정성을 고려했다.

### users

다음과 같은 기능이 구현되어야

1. users 반환
   1. 굳이 필요없는 기능이지만, 개발의 편의 목적.
2. 유저 등록
   1. 추후 auth - 회원가입 기능에서 이를 활용함.
   2. 유저를 등록하는 과정으로, 닉네임 중복 및 email 중복여부 확인 후 등록
3. email을 활용하여 유저 정보를 가져오기

#### users.service.ts

```ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(UsersModel)
    private readonly usersRepository: Repository<UsersModel>,
  ) {}

  async getAllUsers() {
    return this.usersRepository.find();
  }

  async addUser(
    userData: Pick<UsersModel, 'email' | 'nickname' | 'password'>,
    role?: Roles,
  ) {
    const existingEmail = await this.usersRepository.exists({
      where: {
        email: userData.email,
      },
    });
    if (existingEmail) {
      throw new BadRequestException('중복된 email입니다.');
    }

    const existingNickname = await this.usersRepository.findOne({
      where: {
        nickname: userData.nickname,
      },
    });
    if (existingNickname) {
      throw new BadRequestException('중복된 닉네임입니다.');
    }

    const userObject = this.usersRepository.create({
      email: userData.email,
      nickname: userData.nickname,
      password: userData.password,
      role: role,
    });

    const newUser = await this.usersRepository.save(userObject);
    return newUser;
  }

  async getUserByEmail(email: string) {
    return this.usersRepository.findOne({
      where: {
        email,
      },
    });
  }
}
```

#### users.controller.ts

```ts
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  getAllUsers() {
    return this.usersService.getAllUsers();
  }

  @Post()
  addUser(
    @Body('email') email: string,
    @Body('nickname') nickname: string,
    @Body('password') password: string,
    @Body('role') role?: Roles,
  ) {
    return this.usersService.addUser({ email, nickname, password }, role);
  }
}
```

### auth

회원가입 및 로그인 로직은 크게 다음과 같다.

1. email, nickname, password를 받고 회원가입을 진행한다.
   1. 단 email, nickname의 unique여부를 판별하고
   2. password는 암호화(hash)를 진행한다. (bcrypt 설치 필요)
   3. 만약 회원가입이 완료되었다면 Token을 반환한다. (@nestjs/jwt 설치 필요)
2. 로그인의 경우 email과 password를 확인한다.
   1. password확인의 경우 hash값들을 비교한다.
   2. DB에 접근해서 존재하는 유저인지 확인해야한다. (검증)
   3. 검증을 통과하면 Token을 반환한다.

이를 구현하기 위해 auth module을 만들고 구현해보자.
table은 따로 필요없으니 service, controller을 구현하자.

#### 설치

```shell
yarn add @nestjs/jwt bcrypt
```

#### module 주입

설치한 jwt는 추가적인 주입이 필요하다. register을 활용하여 jwt을 주입하자.
또한 기존에 작성한 UsersModule을 활용할 예정이므로 이 또한 주입한다.

```ts
@Module({
  imports: [JwtModule.register({}), UsersModule],
  controllers: [AuthController],
  providers: [AuthService],
})
export class AuthModule {}
```

#### auth.service.ts

로그인 과정은 다음의 과정으로 진행한다.

1. loginWithEmail 함수 실행 (로그인 시작 함수)
2. authenticateWithEmailAndPassword 함수 실행 (검증)
   1. 사용자 존재 여부 확인
   2. 비밀번호 일치여부 확인
   3. 유효한 경우 사용자 정보 반환
3. loginUser 함수 호출 (검증 통과한 경우 실행)
   1. signToken 호출하여 refresh와 access 토큰 생성

그리고 회원가입은 다음의 과정으로 진행한다.

1. registerWithEmail 함수 실행 (회원가입 실행 함수)
2. 비밀번호를 암호화하여 usersService의 함수 활용하여 DB 연결
3. 회원가입 이후 로그인이 되도록 하려면 이후 로그인 과정 진행 (바로 loginUser 함수 호출)

정의해야하는 로직은 다음과 같다.

1. Header에 담아서 보낸 토큰을 확인하는 로직
2. Basic 인증과정
3. 토큰 검증과정
4. Refresh 토큰으로 새로운 Access 토큰을 발급받는 과정
5. JWT 토큰을 생성하는 로직
6. 로그인
7. 회원가입

##### 토큰 관련 로직

토큰 관련된 로직은 다음과 같은 로직이 필요하다.
1. Header에서 토큰 빼기
2. 토큰의 유효성 확인하기 (검증)
3. 토큰 등록
4. access 토큰의 재발급 과정
5. Basic 요청이 온 경우 이를 바탕으로 해당 유저 검증하기
   1. email, password로 분리하고
   2. DB에서 유저 정보가 있는지 판단

```ts
// 1. 토큰 추출 - Header에 담긴 값을 바탕으로 토큰을 확인
extractTokenFromHeader(header: string, isBearer: boolean) {
  const splitToken = header.split(' ');
  const prefix = isBearer ? 'Bearer' : 'Basic';
  if (splitToken.length !== 2 || splitToken[0] !== prefix) {
    throw new UnauthorizedException('Header에 담긴 토큰의 유형이 잘못되었습니다.')
  }
  const token = splitToken[1];
  return token;
}

// 2. 토큰의 유효성 확인
verifyToken(token: string) {
  return this.jwtService.verify(token, {
    secret: JWT_SECRET,
  });
}

// 3. 토큰 등록 - refresh와 access 발급
signToken(user: Pick<UsersModel, 'email' | 'id'>, isRefreshToken: boolean) {
  const payload = {
    email : user.email,
    id : user.id,
    type : isRefreshToken ? 'refresh' : 'access',
  };
  return this.jwtService.sign(payload, {
    secret : JWT_SECRET,
    expiresIn : isRefreshToken ? 3600 : 300,
  })
}

// 4. 토큰의 재발급 과정
rotateToken(token: string, isRefreshToken: boolean) {
  const decoded = this.jwtService.verify(token, {
    secret: JWT_SECRET,
  });

  if (decoded.type !== 'refresh') {
    throw new UnauthorizedException(
      '토큰 재발급은 refresh token으로만 해야합니다.',
    );
  }
  return this.signToken(
    {
      ...decoded,
    },
    isRefreshToken,
  );
}

// 5. Basic 요청 => 로그인 유저 정보 확인 - 해당 값을 바탕으로 로그인 유효성을 판단한다.
decodeBasicToken(base64String: string) {
  const decoded = Buffer.from(base64String, 'base64').toString('utf8');
  const split = decoded.split(':');
  if (split.length !== 2) {
    throw new UnauthorizedException('Header에 담은 토큰 형식이 잘못되었습니다.')
  }
  const email = split[0];
  const password = split[1];
  return {
    email,
    password,
  }
}
```

##### 위의 토큰 관련 로직을 활용한 기능 구현

```ts
// 1. 로그인 과정
async loginWithEmail(user: Pick<UsersModel, 'email' | 'password'>) {
  const existingUser = await this.authenticateWithEmailAndPassword(user);
  return this.loginUser(existingUser);
}

// 1-1 존재하는 유저인지 판단.
async authenticateWithEmailAndPassword(
  user: Pick<UsersModel, 'email' | 'password'>,
) {
  // a. 사용자 정보 확인
  const existingUser = await this.usersService.getUserByEmail(user.email);
  if (!existingUser) {
    throw new UnauthorizedException('존재하지 않는 사용자입니다.');
  }

  // b. 비밀번호의 비교
  // 앞엔 일반 비밀번호 뒤엔 hash값
  const checkPass = await bcrypt.compare(
    user.password,
    existingUser.password,
  );
  if (!checkPass) {
    throw new UnauthorizedException('비밀번호가 틀렸습니다.');
  }

  return existingUser;
}

// 1-2 유효한 유저라면 토큰 발급 과정 진행
loginUser(user: Pick<UsersModel, 'email' | 'id'>) {
  return {
    accessToken: this.signToken(user, false),
    refreshToken: this.signToken(user, true),
  };
}
```

```ts
// 2. 회원가입
async registerWithEmail(
  user: Pick<UsersModel, 'email' | 'password' | 'nickname'>,
) {
  // bcrypt의 경우 해시화 하고 싶은 패스워드 , round를 변수로 적는다.
  // rounds는 hash에 소요되는 시간을 의미한다.
  const hash = await bcrypt.hash(user.password, HASH_ROUNDS);

  const newUser = await this.usersService.createUser({
    ...user,
    password: hash,
  });
  return this.loginUser(newUser);
}
```

#### auth.controller.ts

controller의 역할은 service로의 연결을 잘 해주는 길 안내 역할이므로 service에서 정의한 함수가 요구하는 것을 잘 전달만 해주면 된다.

우리가 안내해줘야 하는 목적지는 다음과 같다.

1. 이메일로 로그인하기 (service에서 loginWithEmail로 실행, 토큰, 이메일, 비밀번호 필요)
   1. Basic Token을 받고 이를 service로 전달. service는 이를 decode 하여 이메일과 비밀번호를 판별한다.
2. 회원가입하기 (registerWithEmail로 실행, 이메일, 비밀번호, 닉네임 필요)
3. 토큰 재발급 (access and refresh)

```ts
@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('token/access')
  postTokenAccess(
    @Headers('authorization') rawToken: string,) {
    const token = this.authService.extractTokenFromHeader(rawToken, true);
    /**
     * 반환 => accessToken : {token}
     */
    const newToken = this.authService.rotateToken(token, false)

    return {
      accessToken : newToken,
    }
  }

  @Post('token/refresh')
  postTokenRefresh(
    @Headers('authorization') rawToken: string,) {
    const token = this.authService.extractTokenFromHeader(rawToken, true);
    /**
     * 반환 => accessToken : {token}
     */
    const newToken = this.authService.rotateToken(token, true)

    return {
      refreshToken : newToken,
    }
  }

  @Post('login/email')
  postLoginEmail(
    @Headers('authorization') rawToken: string,
    @Body('email') email: string,
    @Body('password') password: string,
  ) {
    const token = this.authService.extractTokenFromHeader(rawToken, false);
    const credentials = this.authService.decodeBasicToken(token);

    return this.authService.loginWithEmail(credentials);
  }

  @Post('register/email')
  postRegisterEmail(
    @Body('email') email: string,
    @Body('password') password: string,
    @Body('nickname') nickname: string,
  ) {
    return this.authService.registerWithEmail({email, password, nickname})
  }
}

```

# 결론

Nest에서 가장 중요한 것은 다음과 같다고 생각한다.

1. 의존성을 잘 고려해야한다.
   1. 다른 모듈 (auth module에서 user module 사용하는 것)이나 jwt와 같은 것들을 사용하기 위해선 해당 모듈에 주입하는 것을 잊지 않아야한다.
      1. 해당 모듈을 imports하고 exports하는 것을 생각해야한다.
   2. entity 또한 모듈에서 사용하기 위해선 주입해야한다.
2. 함수의 시작지점을 짜고, 어떤 로직으로 실행되어야하는지 고려해야한다.
   1. login의 경우 검증 => login이 진행되어야함.
   2. 회원가입의 경우 usersService에 접근해서 실행하여야함.
