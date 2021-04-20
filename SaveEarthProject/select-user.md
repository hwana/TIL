# 마이페이지(내가 작성한 댓글 리스트, 내가 찜한 장소 리스트)

## 내가 작성한 댓글 리스트

### CommentRepositoryCustom.java
```java
@Repository
@RequiredArgsConstructor
public class CommentRepositoryCustom {
    private final JPAQueryFactory jpaQueryFactory;
    QComment qComment = QComment.comment;

    public List<String> findUserComment(String userId){
        return jpaQueryFactory.select(qComment.contents)
                .from(qComment)
                .where(qComment.userComment.id.eq(userId))
                .fetch();
    }
}
```
- 댓글 엔티티를 사용하여 댓글 테이블에서 userId로 사용자가 작성한 댓글리스트를 반환하였다.
### CommentService.java
```java
@Service
@RequiredArgsConstructor
@Transactional
public class CommentService {

    private final CommentRepositoryCustom commentRepositoryCustom;

    public List<String> selectUserComment(String userId){
        return commentRepositoryCustom.findUserComment(userId);
    }
}
```
### CommentController.java
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class CommentController {

    private final CommentService commentService;

    @GetMapping("/comment/user/{userId}")
    public List<String> userCommentList(@PathVariable(value = "userId") String userId){
        return commentService.selectUserComment(userId);
    }
}
```
## 내가 찜한 장소 리스트

### LikeReponseDto.java
```java
@Getter
@NoArgsConstructor
public class LikeResponseDto {

    private String name;
    private String imgUrl;
    private String category;

    public LikeResponseDto (Like like) {
        this.name = like.getPlaceLike().getName();
        this.imgUrl = like.getPlaceLike().getImgUrl();
        this.category = like.getPlaceLike().getCategory();
    }
}
```

### LikeRepositoryCustom.java
```java
@Repository
@RequiredArgsConstructor
public class LikeRepositoryCustom {

    private final JPAQueryFactory jpaQueryFactory;
    QPlace qPlace = QPlace.place;
    QLike qLike = QLike.like;

    public List<Like> findLikeList(String userId){
        return jpaQueryFactory.select(qLike)
                .from(qLike)
                .innerJoin(qLike.placeLike, qPlace)
                .fetchJoin()
                .where(qLike.userLike.id.like(userId))
                .fetch();
    }
}
```
- 장소에 대한 정보를 가져와야 하므로 place 엔티티와 페치조인을 했다.

### LikeService.java
```java
@Service
@RequiredArgsConstructor
@Transactional
public class LikeService {

    private final LikeRepositoryCustom likeRepositoryCustom;

    public List<LikeResponseDto> userLikeList(String userId){
        return likeRepositoryCustom.findLikeList(userId).stream().map(LikeResponseDto::new).collect(Collectors.toList());
    }
}
```
### LikeController.java
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class LikeController {

    private final LikeService likeService;

    @GetMapping("/like/{userId}")
    public List<LikeResponseDto> userLike(@PathVariable(value = "userId") String userId){
        return likeService.userLikeList(userId);
    }
}
```
