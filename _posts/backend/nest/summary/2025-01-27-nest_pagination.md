---
title: "[Nest.js 정리] Pagination개념과 Cursor Based Pagination 구현"
date: 2025-01-27 01:00:00 +0900
categories: [Backend, Nest정리]
tags: [nest.js, summary, Pagination]
render_with_liquid: false
mermaid: true
---

# Pagination
- 쿼리에 해당되는 데이터를 한번에 다 불러오지 않고 부분적으로 쪼개서 불러오는 것을 말한다.
  - 한번에 다 가져오는 것, 결국 비용이다! 메모리, 서버 사용 비용 등의 이유로 pagination을 사용.

## Pagination의 종류
1. 페이지를 기준으로 데이터를 잘라서 요청하는 Pagination
  - ***장점*** : 서버의 입장에서 관련 알고리즘이 간단함.
  - ***단점*** : 중간에 데이터가 삽입/삭제된 경우 다음 페이지로 이동했을 때 누락(혹은 중복)되는 데이터가 발생할 수 있다.
2. 스크롤을 내리면(스크롤이 하단에 도달한 경우) 다음 데이터를 가져오는 Pagination
  - 가장 최근에 가져온 데이터를 기준으로 다음 데이터를 가져온다.
  - ***장점*** : 최근 데이터의 기준값을 기반으로 쿼리가 작성되기에 누락/중복 이슈가 적다.
  - ***단점*** : 페이지 기준 Pagination보다 알고리즘이 복잡하다.

## Pagination 구현하기 위한 Global setting
1. ValidationPipe 설정
   - transform : true를 Validation 옵션으로 추가하여 정상적인 정렬이 가능하도록 해야한다.
     - 이 경우 default 값으로 설정된 인스턴스가 값이 들어오면 수정될 수 있도록 하는 것임.

  ```ts
  new ValidationPipe({
    transform: true,
    transformOptions: {
      // class validator을 기준으로 임의로 타입 변형하는 것을 허용한다.
      enableImplicitConversion: true,
    },
  }),
  ```

## Cursor Based Pagination 구현
- 서버는 데이터와 다음 페이징에 관련된 정보를 담은 정보를 보내줘야한다.
  
  ```json
  {
    "data" : [
      // data들
    ],
    "next" : 'next관련 정보'
  }
  ```
  
  이런식으로 보내야한다!

### paginate DTO

만약 post 들을 가져오는 경우, 오름차순/내림차순 정렬, 몇개를 가져올 것인가 등 정보를 담은 DTO를 만들어야한다.

```ts
export class PaginatePostDto {
  // 이전의 마지막 id, 해당 id부터 값을 가져옴
  // @Type => 타입을 변환하는 것 (class transformer)
  // 근데, 그냥 main.ts 에서 transformer option 추가해서 이를 자동화할 수 있다.
  @IsNumber()
  @IsOptional()
  where__id_more_than?: number;

  @IsNumber()
  @IsOptional()
  where__id_less_than?: number;

  // 정렬
  @IsIn(['ASC', 'DESC'])
  @IsOptional()
  order__createdAt: 'ASC' | 'DESC' = 'ASC';

  // 몇개를 응답으로 받을 것이냐.
  @IsNumber()
  @IsOptional()
  take: number = 20;
}
```


#### 네이밍 규칙

위 DTO에서 보면 이상한점이 있다. 왜 `order__createdAt` 이런식으로 _를 하나쓸꺼면 하나쓰지 왜 다를까?
- 요청할 때 값을 보면 알 수 있다. post 들을 받아오는 과정에서 오름차순/내림차순 중 어떤 방식으로 가져올 것이냐 요청할 땐 다음처럼 요청한다.
  ```ts
  // ...
  order: {
    createdAt: dto.order__createdAt,
  }
  // ...
  ```
  - order이란 객체 안에 createdAt이 존재한다. 이러한 규칙을 보기 좋게 하기 위해 저런식으로 네이밍하는 경우가 많다.

### 로직 구현

처음 보면 복잡할 수 있다. 아래 코드에 한줄한줄 주석을 작성했으니 보며 이해해보자.

```ts
async paginatePosts(dto: PaginatePostDto) {
  // where은 최소값이냐, 최대값이냐에 따라 다른 값을 가져야한다.
  // 따라서 빈 객체를 만들고, 상황에 맞는 값을 넣는다 생각하자.
  const where: FindOptionsWhere<PostsModel> = {};

  if (dto.where__id_less_than) {
    where.id = LessThan(dto.where__id_less_than);
  } else if (dto.where__id_more_than) {
    where.id = MoreThan(dto.where__id_more_than);
  }
  
  const posts = await this.postsRepository.find({
    where,
    order: {
      createdAt: dto.order__createdAt,
    },
    take: dto.take,
  });
  
  // data를 가져오는 것 외에도 다음 게시물은 무엇인지, 다음 요청 url은 무엇인지 응답해야한다.
  // 가져온 post 수가 내가 가져오기로한 수와 갯수가 다르다면, 다음 페이지는 없다는 의미임을 활용해 다음처럼 작성한다.
  const lastItem =
    posts.length > 0 && posts.length === dto.take
      ? posts[posts.length - 1]
      : null;
  
  // lastItem이 존재하면 (=== 다음페이지가 있다면) Url을 만들어야한다.
  const nextUrl = lastItem && new URL(`${PROTOCOL}://${HOST}/posts`);
  if (nextUrl) {
    // 우선 dto object를 순회하면서 요청 url을 만들어나간다.
    // 오름차순/내림차순, 가져올 개수는 변경할 필요가 없으니 일단 다 넣는다.
    // 단, where의 경우엔 오름/내림 배열에따라 다른 값을 넣어야하니 for문 이후에 처리한다.
    for (const key of Object.keys(dto)) {
      if (dto[key]) {
        if (key !== 'where__id_more_than' && key !== 'where__id_less_than') {
          nextUrl.searchParams.append(key, dto[key]);
        }
      }
    }

    let key = null;
    if (dto.order__createdAt === 'ASC') {
      key = 'where__id_more_than';
    } else {
      key = 'where__id_less_than';
    }

    nextUrl.searchParams.append(key, lastItem.id.toString());
  }
  
  return {
    data: posts,
    counts: posts.length,
    cursor: {
      after: lastItem?.id ?? null,
    },
    next: nextUrl?.toString() ?? null,
  };
}
```

다음엔 일반적인 Pagination과 Pagination을 일반화하는 과정에 대해 학습하고 업로드할 예정이다.
