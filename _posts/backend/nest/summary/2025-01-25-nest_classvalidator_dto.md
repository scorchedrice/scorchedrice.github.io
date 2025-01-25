---
title: "[Nest.js 정리] Class Validator, DTO, Class Transformer"
date: 2025-01-25 23:00:00 +0900
categories: [Backend, Nest정리]
tags: [nest.js, summary, DTO, classValidator, ClassTransformer]
render_with_liquid: false
mermaid: true
---

# Class Validator

[class-validator 정보 모음](https://github.com/typestack/class-validator/blob/develop/README.md)

- 게시글을 작성할 때, title, content를 받아야하는데, 오타로 인해 에러가 발생할 수 있음. 이러한 경우 효과적으로 관리하기 위해 활용하는 것이 class validator
  - 이 때, title과 content를 class로 묶어서 관리하는데, 이를 DTO라고 한다.
  - DTO는 Data Transfer Object의 약자이다.
  ```ts
  // posts.service.ts의 일부
  async createPost(authorId: number, title: string, content: string) {
    // create => 저장할 객체를 생성한다.
    // save => 객체를 저장한다. (create 매서드에서 생성한 객체로)
    // 이를 조합해서 진행하자!
    const post = this.postsRepository.create({
      author: {
        id: authorId,
      },
      title,
      content,
      likeCount: 0,
      commentCount: 0,
    });
    const newPost = await this.postsRepository.save(post);
    return newPost;
  } 
  ```
  만약 이와같은 예시가 있다면, 다음처럼 DTO를 설정하고 간단하게 변경할 수 있다.

  ```ts
  // posts/dto/create-post.dto.ts
  export class CreatePostDto {
    title : string;
    content : string;
  }
  ```
  ```ts
  async createPost(authorId: number, postDto: CreatePostDto) {
    // create => 저장할 객체를 생성한다.
    // save => 객체를 저장한다. (create 매서드에서 생성한 객체로)
    // 이를 조합해서 진행하자!
    const post = this.postsRepository.create({
      author: {
        id: authorId,
      },
      ...postDto,
      likeCount: 0,
      commentCount: 0,
    });
    const newPost = await this.postsRepository.save(post);
    return newPost;
  }
  ```
  이에 맞게 controller도 바꿔줘야 한다.
  ```ts
  postPosts(
    // @Request() req : any,
    @User('id') userId: number,
    @Body() body: CreatePostDto,
    // @Body('title') title: string,
    // @Body('content') content: string,
  ) {
    const authorId = userId;
    return this.postsService.createPost(authorId, body);
  }
  ```
## 설치
```shell
yarn add class-validator class-transformer
```

설치한 이후 모든 모듈에서 사용가능하도록 `main.ts`를 수정해야한다.

```ts
// main.ts 에 다음과 같은 내용을 삽입한다.
// 해당 코드는 Global한 환경(app 전반적 환경)에서 validation Pipe를 사용할 수 있도록 한다는 코드이다.

app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
  }),
);
```

##  사용 예시
위에서 정의한 Dto에 사용하는 예시로 사용방법을 확인해보자.

```ts
import { IsString } from "class-validator";

export class CreatePostDto {
    @IsString()  
    title : string;
    
    @IsString()
    content : string;
  }
```

이런식으로 string만 가능하도록 validator을 사용할 수 있다.

## 에러 메세지 변경 방법
- 아래 validator에 옵션을 넣어 메세지를 변환할 수 있다.
```ts
import { IsString } from "class-validator";

export class CreatePostDto {
    @IsString({
      message: "제목은 스트링이여야함!"
    })
    title : string;

    @IsString({
      message: "내용은 스트링이여야한다고!"
    })
    content : string;
}
```

### 메세지의 일반화
- 만약 글자제한을 두는 `@Length` validation을 사용하는 경우, 계속 몇글자부터 몇글자 내로 작성해라.. 이런 메세지를 하나하나 다 적어야할까?
  - 이 경우 message를 함수로 받고, args를 전달한 후 이를 활용할 수 있다.
  ```ts
  @Length(1, 20, {
    message(args: ValidationArguments) {
      // value : 검증되고 있는 값
      // constraints : 입력된 제한 사항들
      // targetName : 검증하고 있는 클래스 명
      // object : 검증하고 있는 객체
      // property : 검증되고 있는 객체의 프로퍼티 이름
      if (args.constraints.length === 2) return `${args.constraints[0]} - ${args.constraints[1]} 사이의 글자를 입력해주세요.`
      else return `${args.constraints[0]} 이상을 입력해주세요.`
    }
  })
  ```
  아니면 다음처럼 함수를 정의하고 사용할 수 있다.
  ```ts
  export function lengthValidation(args: ValidationArguments) {
    if (args.constraints.length === 2) return `${args.constraints[0]} - ${args.constraints[1]} 사이의 글자를 입력해주세요.`
    else return `${args.constraints[0]} 이상을 입력해주세요.`
  }
  ```
  ```ts
  @Length(1,20,{
    message: lengthValidation
  })
  ```
  
## 효율적인 DTO 활용
- class로 사용하기에 상속 등 class의 다양한 기능을 활용한 방법으로 이를 관리할 수 있다.
  - entity에서 PickType을 사용해서 이를 가져올 수 있다.

### Entity의 변경
생각해보니까, title, content 모두 PostsModel에 있다. 그렇다면 다음처럼 Entity를 수정하고, 이를 가져올 수 있지 않을까?
```ts
@Entity()
export class PostsModel extends BaseModel {
  @ManyToOne(() => UsersModel, (user) => user.posts, {
    nullable: false,
  })
  author: UsersModel;

  @Column()
  @IsString({
    message: 'title은 string을 받아야합니다.'
  })
  title: string;

  @Column()
  @IsString({
    message: 'content는 string을 받아야합니다.'
  })
  content: string;

  @Column()
  likeCount: number;

  @Column()
  commentCount: number;
}
```
위 처럼 `@IsString()`을 해당 엔티티에 넣었다.

그 이후 DTO를 다음처럼 바꿀 수 있다.

그냥 Pick은 extend에 사용할 수 없는데, PickType이라는 유틸리티를 활용하면 이를 extends할 수 있다.

> Pick, Omit, Partial => Type 반환
> PickType, OmitType, PartialType => 값을 반환

```ts
export class CreatePostDto extends PickType(PostsModel, ['title', 'content']) {}
```

만약 update와 같이 필수적으로 입력하지 않아도 되는 경우엔 `@IsOptional`,
길이 제한이 필요한 경우엔 `@Length` 등 다양한 validator을 활용하면 된다.

# Class Transformer

만약 유저정보를 가져오는데, 비밀번호와 같은 민감정보를 FE측에 넘겨줄 필요가 있을까? 이런식으로 숨기고 싶은 정보가 있거나 할 때 `@Exclude`와 같은 것들을 사용할 수 있다.

## 사용 과정

우선 사용하기 위해선 `class-transformer`가 설치되어 있어야하고, `controller.ts`에 `@UseInterceptors(ClassSerializerInterceptor)`가 적용되어 있어야한다.
해당 내용은 추후에 다룰 예정이다. 아무튼!

```ts
@Get()
@UseInterceptors(ClassSerializerInterceptor)
getAllUsers() {
  return this.usersService.getAllUsers();
}
```

이처럼 controller을 설정해주고

```ts
@Column()
@IsString()
@Length(3,8)
@Exclude({
  toPlainOnly: true,
})
password: string;
```
이런식으로 적용한다면! 비밀번호가 나오지 않음을 알 수 있다~!

### Exclude Annotation Option
크게 두가지가 있다. `toClassOnly`, `toPlainOnly`

`toClassOnly`는 class instance (dto) 로 변환될때만 (요청)

`toPlainOnly`는 plain object (json, 응답) 로 변환할때만 (응답) 

옵션을 따로 지정하지 않으면, 위 두가지 상황 모두 적용되므로, 비밀번호의 경우엔 받는 것은 허용해야하니 위와같은 옵션을 설정해야한다.

## Interceptor app 전체에 적용하기
근데, 위처럼 하나하나 하는 것 좋으나, `@UseInterceptors(ClassSerializerInterceptor)` 이걸 까먹고 그런다면 보안적 문제가 발생할 수 있다.

`app.module.ts`에서 이를 모든 app에서 적용하도록 할 수 있다.

providers부분을 수정하면되는데, 다음처럼 수정하면 적용된다.

```ts
providers: [
  AppService,
  {
    provide: APP_INTERCEPTOR,
    useClass: ClassSerializerInterceptor,
  },
],
```
