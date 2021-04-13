# 스토리 전체 리스트 조회(페이징)
> 스토리 페이지는 등록 서비스는 제공하지 않고, 조회 서비스만 제공한다.<br>
> 그래서 에코 플레이스 전체 리스트 조회와 비슷하게 로직을 구성하였다.

## StoryResponseDto.java
```java
@Getter
@NoArgsConstructor
public class StoryResponseDto {
    private String title;
    private String contents;
    private String imgUrl;

    public StoryResponseDto(Story story){
        this.title = story.getTitle();
        this.contents = story.getContents();
        this.imgUrl = story.getImgUrl();
    }
}
```

## StoryRepository.java
```java
public interface StoryRepository extends JpaRepository<Story, Long> {
}

```

## StoryService.java
```java
public class StoryService {
    private final StoryRepository storyRepository;

    public List<StoryResponseDto> findAllStory(Pageable pageable){
        return storyRepository.findAll(pageable).stream().map(StoryResponseDto::new).collect(Collectors.toList());
    }
}

```

## StoryController.java
```java
@RestController
@RequiredArgsConstructor
public class StoryController {

    private final StoryService storyService;

    @GetMapping("/api/story")
    public List<StoryResponseDto> storyList(@PageableDefault(sort = { "id" }, direction = Sort.Direction.DESC, size = 6) Pageable pageable){
        return storyService.findAllStory(pageable);
    }
}
```
