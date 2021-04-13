# 에코 플레이스, 스토리 상세보기
- 에코 플레이스, 스토리의 아이디 값으로 DB에서 상세 조회를 했다.

## PlaceDetailResponseDto.java
```java
@Getter
@NoArgsConstructor
public class PlaceDetailResponseDto {
    private String name;
    private String address;
    private String imgUrl;
    private String category;
    private String contents;
    private int allMenuVegan;

    public PlaceDetailResponseDto(Place place) {
        this.name = place.getName();
        this.address = place.getAddress();
        this.imgUrl = place.getImgUrl();
        this.category = place.getCategory();
        this.contents = place.getContents();
        this.allMenuVegan = place.getAllMenuVegan();
    }
}
```

## PlaceRepository.java
```java
public interface PlaceRepository extends JpaRepository<Place, Long> {
    public Optional<Place> findById(Long id);
}
```

## PlaceService.java
```java
@Service
@Transactional
@RequiredArgsConstructor
public class PlaceService {
    private final PlaceRepository placeRepository;

    public Optional<PlaceDetailResponseDto> findById(Long id){
        return placeRepository.findById(id).map(PlaceDetailResponseDto::new);
    }
}
```

## PlaceController.java
```java
@RestController
@RequiredArgsConstructor
public class PlaceController {

    private final PlaceService placeService;
    
    @GetMapping("/api/place/{placeId}")
    public Optional<PlaceDetailResponseDto> placeDetailList(@PathVariable(value = "placeId") Long id){
        return placeService.findById(id);
    }
}
```

## StoryRepository.java
```java
public interface StoryRepository extends JpaRepository<Story, Long> {
    Optional<Story> findById(Long id);
}
```

## StoryService.java
```java
@Service
@Transactional
@RequiredArgsConstructor
public class StoryService {
    private final StoryRepository storyRepository;

    public Optional<StoryResponseDto> findById(Long id){
        return storyRepository.findById(id).map(StoryResponseDto::new);
    }
}
```

## StoryController.java
```java
@RestController
@RequiredArgsConstructor
public class StoryController {

    private final StoryService storyService;

    @GetMapping("/api/story/{storyId}")
    public Optional<StoryResponseDto> storyDetailList(@PathVariable(value = "storyId") Long id){
        return storyService.findById(id);
    }
}
```
