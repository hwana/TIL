# 댓글 등록, 수정, 삭제, 조회

> 게시글에 댓글이 달릴 수 있으며, 댓글이 10개 이상일때는 페이징이 된다. 댓글을 수정할 때는 제목만 수정이 되며 삭제가 가능하다.

## 댓글 삭제
### CommentController.java
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class CommentController {
    private final CommentService commentService;

    @DeleteMapping("/comment/{commentId}")
    public void deleteComment(@PathVariable(value = "commentId") Long id){
        commentService.deleteComment(id);
    }
}
````
### CommentService.java
```java
@Service
@RequiredArgsConstructor
@Transactional
public class CommentService {

    private final CommentRepository commentRepository;
    
    public void deleteComment(Long id){
        commentRepository.deleteById(id);
    }
}
```
## 댓글 수정
### Comment.java
```java
@Getter
@NoArgsConstructor
@Entity
@Table(name = "comments")
public class Comment extends BaseTimeEntity{
    .
    .
    .

    public void changeContents(String contents){
        this.contents = contents;
    }
}
```
- JPA에서 제공하는 변경감지 기능을 사용하기 위해서 changeContents 메소드를 생성하였다.

### CommentUpdateRequestDto.java
```java
@Getter
@NoArgsConstructor
public class CommentUpdateRequestDto {
    private String contents;
}
```
### CommentController.java
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class CommentController {

    private final CommentService commentService;

    @PatchMapping("/comment/{commentId}")
    public void updateComment(@PathVariable(value = "commentId") Long id, @RequestBody CommentUpdateRequestDto commentUpdateRequestDto){
        commentService.updateComment(id, commentUpdateRequestDto.getContents());
    }
}
```
### CommentService.java
```java
@Service
@RequiredArgsConstructor
@Transactional
public class CommentService {

    private final CommentRepository commentRepository;

    public void updateComment(Long id, String contents){
        Comment comment = commentRepository.getOne(id);
        comment.changeContents(contents);
    }
}
```
- getOne(id) 메소드를 사용하여 1차 캐시에 저장되어 있던 객체를 조회하여, 해당 객체를 변경감지 기능으로 수정을 했다.
