# 에코 플레이스 등록, 조회하기

### PlaceSaveDto.java
```java
@Getter
@NoArgsConstructor
public class PlaceSaveRequestDto {

    private String name;

    private String address;

    private String imgUrl;

    private String category;

    private String contents;

    private int allMenuVegan;

    @Builder
    public PlaceSaveDto(String name, String address, String imgUrl, String category, String contents, int allMenuVegan) {
        this.name = name;
        this.address = address;
        this.imgUrl = imgUrl;
        this.category = category;
        this.contents = contents;
        this.allMenuVegan = allMenuVegan;
    }

    public Place toEntity(){
        return Place.builder()
          .name(name).address(address).imgUrl(imgUrl).category(category).contents(contents).allMenuVegan(allMenuVegan).build();
    }

}
```
- Entity 클래스는 DB와 맞닿아 있는 핵심 클래스로서 데이터를 주고 받기 위해서 사용하면 안된다.
- toEntity 메소드를 작성하여 DTO에서 Entity로 변환이 가능하게 했다.

### PlaceRepository.java
```java
public interface PlaceRepository extends JpaRepository<Place, Long> {
    public List<Place> findAll();
}
```
- Spring Data JPA에서 제공하는 JpaRepository를 구현하였다.

### PlaceService.java
```java
@Service
@Transactional
@RequiredArgsConstructor
public class PlaceService {
    private final PlaceRepository placeRepository;

    public void savePlace(PlaceSaveRequestDto placeSaveRequestDto, MultipartFile file){
        try {
            String baseDir = "디렉토리 경로";
            String filePath = baseDir + "\\" + file.getOriginalFilename();
            file.transferTo(new File(filePath));

            placeSaveRequestDto.setImgUrl(filePath);
            placeRepository.save(placeSaveRequestDto.toEntity());
        }catch(Exception e){
            e.printStackTrace();
        }
    }

    public List<Place> findAllPlace(){
        return placeRepository.findAll();
    }
}
```
- @RequiredArgsConstructor 어노테이션을 사용하여 초기화 되지 않은 final 필드의 의존성을 주입하였다.
- 현재는 로컬 서버에 이미지가 업로드 되도록 구현하였다.

### PlaceController.java
```java
@RestController
@RequiredArgsConstructor
public class PlaceController {

    private final PlaceService placeService;

    @PostMapping("/api/place")
    public void save(PlaceSaveRequestDto placeSaveRequestDto, @RequestParam("img") MultipartFile file){
        placeService.savePlace(placeSaveRequestDto, file);
    }

    @GetMapping("/api/place")
    public List<Place> placeList(){
        return placeService.findAllPlace();
    }
}
```

### REST API TEST
#### 에코 플레이스 등록하기
![image](https://user-images.githubusercontent.com/37647995/114399899-62585a80-9bdc-11eb-89da-f020fe8f3751.png)

#### 에코 플레이스 전체 조회
![image](https://user-images.githubusercontent.com/37647995/114400060-8c118180-9bdc-11eb-8437-32d4e5bb1900.png)

