# 에코 플레이스 등록, 조회하기(페이징 기능 추가)

## DTO를 사용한 이유
- Entity 클래스는 DB와 맞닿아 있는 핵심 클래스로서 단순 데이터를 주고 받기 위해서 사용하면 안된다.
- 실제로 한 엔티티를 위한 API가 다양하게 만들어지는데, 한 엔티티가 각각의 API를 위한 모든 요구사항을 담기는 어렵다.
- 엔티티를 변경하게 되면 API 스펙이 변하게 된다.<br>
‼ **결론 : 엔티티 대신 API 스펙에 맞는 별도의 DTO를 노출해야 한다.**

### PlaceSaveRequestDto.java
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

    public Place toEntity(){
        return Place.builder()
                .name(name)
                .address(address)
                .imgUrl(imgUrl)
                .category(category)
                .contents(contents)
                .allMenuVegan(allMenuVegan)
                .build();
    }
}
```
- 에코 플레이스 저장을 위한 DTO
- toEntity 메소드를 작성하여 DTO에서 Entity로 변환이 가능하게 했다.

### PlaceResponseDto.java
```java
@Getter
@NoArgsConstructor
public class PlaceResponseDto {
    private String name;
    private String address;
    private String imgUrl;
    private String category;
    private String contents;
    private int allMenuVegan;

    public PlaceResponseDto(Place place) {
        this.name = place.getName();
        this.address = place.getAddress();
        this.imgUrl = place.getImgUrl();
        this.category = place.getCategory();
        this.contents = place.getContents();
        this.allMenuVegan = place.getAllMenuVegan();
    }
}
```
- 에코 플레이스 조회 응답을 위한 DTO
- Place Entity를 매개변수로 받은 생성자로 각 필드를 초기화 시켜주었다.(Entity ➡ DTO 변환)

### PlaceRepository.java
```java
public interface PlaceRepository extends JpaRepository<Place, Long> {
    public Page<Place> findAll(Pageable pageable);
}
```
- Spring Data JPA에서 제공하는 JpaRepository를 구현하였다.

> #### PlaceRepository에서 `Page<Place> findAll()` 메소드를 사용할 수 있는 이유<br>
> JpaRepository는 PagingAndSortingRepository를 상속받기 때문에 기본적인 페이징 기능을 구현할 수 있다.
> ![image](https://user-images.githubusercontent.com/37647995/114500586-48ae2600-9c63-11eb-9b5e-9e921c7e8225.png)

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

     public List<PlaceResponseDto> findAllPlace(Pageable pageable){
        return placeRepository.findAll(pageable).stream().map(PlaceResponseDto::new).collect(Collectors.toList());
    }
}
```
- @RequiredArgsConstructor 어노테이션을 사용하여 초기화 되지 않은 final 필드의 의존성을 주입하였다.
- 현재는 로컬 서버에 이미지가 업로드 되도록 구현하였다.
- Repository에서 반환된 Place(entity)를 스트림을 활용해 PlaceResponseDto로 변환해서 리스트로 반환하였다.

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
    public List<PlaceResponseDto> placeList(@PageableDefault(sort = { "name" }, direction = Sort.Direction.ASC, size = 12) Pageable pageable){
        return placeService.findAllPlace(pageable);
    }
}
```
- 페이징 기본 정렬은 이름순으로, 한 페이지당 불러올 게시물 수는 12개로 설정하였다.
> #### `@PageableDefault` 어노테이션 제공<br>
> ![image](https://user-images.githubusercontent.com/37647995/114501014-1cdf7000-9c64-11eb-9c3b-b2d742f88a33.png)

### REST API TEST
#### 에코 플레이스 등록하기
![image](https://user-images.githubusercontent.com/37647995/114399899-62585a80-9bdc-11eb-89da-f020fe8f3751.png)

#### 에코 플레이스 전체 조회
![image](https://user-images.githubusercontent.com/37647995/114400060-8c118180-9bdc-11eb-8437-32d4e5bb1900.png)

