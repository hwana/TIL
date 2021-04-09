# 엔티티 생성하기

- 설계한 [DB명세서](https://www.notion.so/DB-47c57df87f564ccfb291cdc99f5ad9a2)를 기반으로 엔티티를 생성한다.
- 테이블 명은 복수형으로 통일했지만 편의상 엔티티명은 단수형을 사용했다.
- 모든 연관관계에 지연로딩(fetch = FetchType.LAZY)을 사용했다.
- User와 Comment, Like / Place와 Comment 를 양방향 관계로 설정했다.

## Comment.java
```java
@Getter
@NoArgsConstructor
@Entity
@Table(name = "comments")
public class Comment extends BaseTimeEntity{

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "COMMENT_ID")
    private Long id;

    @Column(nullable = false)
    private String contents;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "USER_ID")
    private User userComment;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "PLACE_ID")
    private Place placeComment;

    @Column(columnDefinition = "integer default 0")
    private int delYn;
}
```
## like.java
```java
@Getter
@NoArgsConstructor
@Entity
@Table(name = "likes")
public class Like {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "LIKE_ID")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "USER_ID")
    private User userLike;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "PLACE_ID")
    private Place placeLike;

}
```
## place.java
```java
@Getter
@NoArgsConstructor
@Entity
@Table(name = "places")
public class Place {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "PLACE_ID")
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String address;

    @Column(nullable = false)
    private String imgUrl;

    @Column(nullable = false)
    private String category;

    @Column(nullable = false)
    private String contents;

    @OneToMany(mappedBy = "placeComment")
    private List<Comment> commentList = new ArrayList<Comment>();

}

```
## Story.java
```java
@Getter
@NoArgsConstructor
@Entity
@Table(name = "stories")
public class Story extends CreateTimeEntity{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "STORY_ID")
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(nullable = false)
    private String contents;
}
```
## User.java
```java
@Getter
@NoArgsConstructor
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "USER_ID")
    private Long id;

    @Column(nullable = false)
    private String email;

    @Column(nullable = false)
    private String name;

    @Column
    private String imgUrl;

    @OneToMany(mappedBy = "userLike")
    private List<Like> likeList = new ArrayList<Like>();

    @OneToMany(mappedBy = "userComment")
    private List<Comment> commentList = new ArrayList<Comment>();
}
```

## BaseTimeEntity.java
```java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseTimeEntity {
    @CreatedDate
    private LocalDateTime createDateTime;

    @LastModifiedDate
    private LocalDateTime modifiedDateTime;
}
```
- 모든 Entity의 상위 클래스가 되어 Entity들의 createDateTime, modifiedDateTime을 자동으로 관리하는 역할
- `@EntityListeners(AuditingEntityListener.class)` : JPA Auditing 활성화 어노테이션
- 실제 엔티티로 사용되지는 않고 반복되는 구조를 상속받음으로써 해결

