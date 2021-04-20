# 좋아요 등록, 좋아요 취소
> 1. 사용자는 한 장소에 하나의 좋아요를 누를 수 있다.
> 2. 사용자는 눌렀던 좋아요를 취소할 수 있다.

## LikeSaveRequestDto.java
```java
@Getter
@NoArgsConstructor
public class LikeSaveRequestDto {
    private String userId;
    private Long placeId;
}
```
## LikeRepository.java
```java
public interface LikeRepository extends JpaRepository<Like, Long> {
}
```
## LikeRepositoryCustom.java
```java
@Repository
@RequiredArgsConstructor
public class LikeRepositoryCustom {

    private final JPAQueryFactory jpaQueryFactory;
    QPlace qPlace = QPlace.place;
    QLike qLike = QLike.like;

    public Long findLikeByUserIdAndPlaceId(String userId, Long placeId){
        return jpaQueryFactory.select(qLike.id)
                .from(qLike).where(qLike.placeLike.id.eq(placeId).and(qLike.userLike.id.like(userId)))
                .fetchOne();
    }
}
```
## LikeService.java
```java
@Service
@RequiredArgsConstructor
@Transactional
public class LikeService {

    private final LikeRepository likeRepository;
    private final PlaceRepository placeRepository;
    private final UserRepository userRepository;
    private final LikeRepositoryCustom likeRepositoryCustom;

    public void like(LikeSaveRequestDto likeSaveRequestDto){
        Long likeId = likeRepositoryCustom.findLikeByUserIdAndPlaceId(likeSaveRequestDto.getUserId(), likeSaveRequestDto.getPlaceId());

        if(likeId == null){
            Place place = placeRepository.getOne(likeSaveRequestDto.getPlaceId());
            User user = userRepository.getOne(likeSaveRequestDto.getUserId());
            likeRepository.save(Like.builder().placeLike(place).userLike(user).build());
        }else{
            likeRepository.deleteById(likeId);
        }
    }
}
```
- likeId가 조회되지 않을 땐 데이터를 insert 하고 likeId가 조회된다면 데이터를 delete 한다.
## LikeController.java
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class LikeController {

    private final LikeService likeService;

    @PostMapping("/like")
    public void like(@RequestBody LikeSaveRequestDto likeSaveRequestDto){
        likeService.like(likeSaveRequestDto);
    }
}
```
