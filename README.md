# 게시판 REST API

Spring Boot + JPA로 만든 글/댓글 REST API 프로젝트입니다.
기능을 한 단계씩 구현하면서 단계마다 커밋했습니다.

## 기술 스택

- Java 25
- Spring Boot 4.1.2 (Web MVC, Data JPA, Validation)
- H2 Database (MySQL 모드)
- Lombok

## 실행 환경 (프로필)

| 프로필 | DB | 설명 |
|---|---|---|
| `dev` (기본) | `jdbc:h2:./db_dev` | 파일 DB, 재시작해도 데이터 유지 |
| `test` | `jdbc:h2:mem:db_test` | 메모리 DB |

- H2 콘솔: `http://localhost:8080/h2-console`
- SQL 로그(`show-sql`, 바인딩 파라미터)와 트랜잭션 로그가 출력됩니다.

## 패키지 구조

```
com.back
├── BackApplication.java              # @EnableJpaAuditing
├── domain
│   └── post
│       ├── post
│       │   ├── controller/ApiV1PostController
│       │   ├── dto/PostDto
│       │   ├── entity/Post
│       │   ├── repository/PostRepository
│       │   └── service/PostService
│       └── postComment
│           ├── controller/ApiV1PostCommentController
│           ├── dto/PostCommentDto
│           └── entity/PostComment
└── global
    ├── initData/BaseInitData         # 샘플 데이터 자동 생성
    ├── jpa/entity/BaseEntity         # id, createDate, modifyDate 공통 필드
    └── rsData/RsData                 # 공통 응답 객체
```

## 주요 구현 내용

### 1. 엔티티

- **BaseEntity**: 모든 엔티티의 부모 클래스(`@MappedSuperclass`)
  - `id`(IDENTITY), `createDate`(`@CreatedDate`), `modifyDate`(`@LastModifiedDate`)
  - JPA Auditing으로 생성일/수정일 자동 기록
- **Post**: `title`, `content`, `comments`
  - 댓글과 `@OneToMany` 양방향 관계 (`cascade = PERSIST, REMOVE`, `orphanRemoval = true`)
  - 댓글 추가/조회/삭제를 `Post`가 담당 (`addComment`, `findCommentById`, `deleteComment`)
- **PostComment**: `post`(`@ManyToOne(fetch = LAZY)`), `content`

### 2. 샘플 데이터 자동 생성

`BaseInitData`가 앱 시작 시 `ApplicationRunner`로 샘플 데이터를 넣습니다.

- 글 3개, 댓글 5개 (글 1에 3개, 글 2에 2개)
- 데이터가 이미 있으면 건너뜀 (`postService.count() > 0`)
- `@Lazy` self 주입으로 프록시를 거쳐 호출해서 `@Transactional`이 적용되도록 함

### 3. DTO

엔티티를 그대로 반환하지 않고 DTO(`record`)로 변환해서 응답합니다.

- **PostDto**: `id`, `createDate`, `modifyDate`, `title`, `content`
- **PostCommentDto**: `id`, `createDate`, `modifyDate`, `content`

엔티티를 직접 반환할 때 생기는 순환 참조 문제(글 ↔ 댓글)를 DTO로 해결했습니다.

### 4. 공통 응답 객체 `RsData<T>`

```java
public record RsData<T>(String resultCode, String msg, T data) {
    public RsData(String resultCode, String msg) {
        this(resultCode, msg, null);
    }
}
```

- `Map` → 댓글 전용 `RsData` → `Object` 타입 → 제네릭 `RsData<T>` 순서로 개선
- 반환할 데이터가 없으면 `RsData<Void>`를 사용

### 5. 트랜잭션

- 조회 API: `@Transactional(readOnly = true)`
- 삭제 API: `@Transactional`
- `hikari.auto-commit: false`

## API 명세

### 글

| Method | URL | 설명 | 응답 |
|---|---|---|---|
| GET | `/api/v1/posts` | 글 목록 조회 | `List<PostDto>` |
| GET | `/api/v1/posts/{id}` | 글 단건 조회 | `PostDto` |
| GET | `/api/v1/posts/{id}/delete` | 글 삭제 | `RsData<Void>` |

### 댓글

| Method | URL | 설명 | 응답 |
|---|---|---|---|
| GET | `/api/v1/posts/{postId}/comments` | 댓글 목록 조회 | `List<PostCommentDto>` |
| GET | `/api/v1/posts/{postId}/comments/{id}` | 댓글 단건 조회 | `PostCommentDto` |
| GET | `/api/v1/posts/{postId}/comments/{id}/delete` | 댓글 삭제 | `RsData<Void>` |

### 응답 예시

```json
// GET /api/v1/posts/1
{
  "id": 1,
  "createDate": "2026-11-16T14:55:37.123456",
  "modifyDate": "2026-11-16T14:55:37.123456",
  "title": "제목 1",
  "content": "내용 1"
}
```

```json
// GET /api/v1/posts/1/comments/1/delete
{
  "resultCode": "200-1",
  "msg": "1번 댓글이 삭제되었습니다.",
  "data": null
}
```

## 커밋 히스토리

| 커밋 | 내용 |
|---|---|
| 000 | 프로젝트 생성 |
| 001 | JPA/H2 설정 및 Post, PostComment 엔티티 추가 |
| 002 | 글 목록 조회 API 추가 및 JPA 설정 |
| 003 | 글 단건 조회 API 추가 |
| 004 | 글 조회 API 응답을 PostDto로 변경 |
| 005 | PostDto를 record로 변경 |
| 006 | 댓글 목록/단건 조회 API 추가 |
| 007 | 댓글 삭제 API 추가 및 조회 API에 readOnly 트랜잭션 적용 |
| 008 | 댓글 삭제 API 응답을 resultCode, msg 형태로 변경 |
| 009 | 공통 응답 객체 RsData 추가 및 댓글 삭제 API에 적용 |
| 010 | RsData에 data 필드 추가 및 삭제된 댓글 정보 반환 |
| 011 | 글 삭제 API 추가 |
| 012 | RsData data 타입을 Object로 변경 및 ForPostRsData 제거 |
| 013 | RsData를 제네릭으로 변경 |
| 014 | 삭제 API 응답에서 data 제거 및 RsData 생성자 추가 |

## 앞으로 개선할 점

- 테스트에 `@ActiveProfiles("test")` 적용 (지금은 테스트도 dev DB를 사용)
- 삭제 API를 `GET` 대신 `@DeleteMapping`으로 변경
- 없는 id 요청 시 500 대신 404 응답 (`Optional.get()` 예외 처리)
- 글/댓글 작성, 수정 API 추가
