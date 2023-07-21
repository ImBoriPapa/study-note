# 사용자 정의 예외 클래스와 Enum을 활용한 에러응답 

### 1. 문제 정의
- 클라이언트가 API 요청 시 에러가 발생할 때, 더 상세한 정보를 포함한 응답이 필요하다는 요구가 있었습니다. 

### 2. 사실 수집
- 에러 응답시 적절한 HttpStatus 사용하지 않고 있다.
- 에러 응답 발생시 어떤 이유에서 에러가 발생했는지 클라이언트가 알기 어렵다.

1. 예외를 중앙에서 처리할 전역 예외 처리기 구현
  ```java
  @RestControllerAdvice
  @Slf4j
  public class GlobalExceptionAdvisor {

    @ExceptionHandler(IllegalArgumentException.class)
    public final ResponseEntity<ApiResponse.Error> handleIllegalArgumentException(HttpServletRequest request, IllegalArgumentException e) {
        log.info("handle illegalArgumentException Requested Method:{}, Path: {}",request.getMethod(), request.getRequestURI());

        return ResponseEntity.badRequest().build();
    }
  }
  ```

2. 테스트 코드 작성
- /api/v2/posts으로 PATCH 요청시 IllegalArgumentException이 발생
- 응답은 HttpStatusCode 404

  ```java
  class GlobalExceptionAdvisorTest {

    private final GlobalExceptionAdvisor globalExceptionAdvisor;

    public GlobalExceptionAdvisorTest() {
        this.globalExceptionAdvisor = new GlobalExceptionAdvisor();
    }

    @Test
    @DisplayName("IllegalArgumentException 핸들링 테스트")
    void IllegalArgumentExceptionHandlingTest() throws Exception {
        //given
        HttpServletRequest request = Mockito.mock(HttpServletRequest.class);
        Mockito.when(request.getRequestURI()).thenReturn("/api/v2/posts");
        Mockito.when(request.getMethod()).thenReturn("PATCH");

        IllegalArgumentException illegalArgumentException = new IllegalArgumentException("수정 권한 없음");
        //when
        ResponseEntity<GlobalExceptionAdvisor.Error> errorResponseEntity = globalExceptionAdvisor.handleIllegalArgumentException(request, illegalArgumentException);
        GlobalExceptionAdvisor.Error error = errorResponseEntity.getBody();
        //then
        assertThat(errorResponseEntity.getStatusCode().is4xxClientError()).isTrue();
        assertThat(error.getMessage().equals(illegalArgumentException.getMessage())).isTrue();
    }
  ```

### 3. 원인추론
1. 위 테스트와 같이 게시물을 수정하는 PATCH 요청을 보냈다고 가정
2. 예외가 발생하고 원인은 작성자가 본인이 아니라서 게시글 수정권한이 없음 -> 권한없음을 알려주는 HttpStatus forbidden이 필요합니다.
  
### 4. 조사방법 결정
- 응답에 적절한 HttpStatus의 사용
- 사용자 정의 예외의 구현 및 활용
- Enum 클래스로 사용자 정의 응답값을 정의하여 일관성 있는 정보 유지

### 5. 조사 방법 구현

1. API 응답의 상태, 메시지,HttpStatus를 Enum을 이용해서 정의
   Enum을 활용하여 가독성이 좋고 일관성이 있게 응답 데이터를 관리해 보았습니다.
```java
@Getter
public enum ApiResponseStatus {

    SUCCESS(1000, "요청에 성공했습니다.", HttpStatus.OK),
    LOGOUT_SUCCESS(1001, "로그아웃 요청이 성공했습니다.", HttpStatus.OK),
    SERVER_ERROR(1002, "서버에 문제가 발생했습니다.", HttpStatus.INTERNAL_SERVER_ERROR),
    VALIDATION_ERROR(1003, "요청 값 검증에 실패했습니다.", HttpStatus.BAD_REQUEST),
    PERMISSION_DENIED(1004, "접근 권한이 없습니다.", HttpStatus.FORBIDDEN),
    BANNED_WORD(1005, "사용 금지된 단어를 사용하였습니다.", HttpStatus.BAD_REQUEST),
    INVALID_EMAIL_FORM(1006, "잘못된 이메일 형식입니다.", HttpStatus.BAD_REQUEST),
    FOUND_ACTIVE_MEMBERS_WITH_DUPLICATE_EMAILS(1007, "중복된 이메일로 활동중인 회원을 발견했습니다.", HttpStatus.BAD_REQUEST),
    ...
```

2. 사용자 정의 예외를 사용함으로써 예외 코드의 통합
- Spring의 RuntimeException 발생시 롤백 정책을 사용할 수 있게 RuntimeException과 상속을 활용하여 사용자 정의 예외를 구현 필요한 정보를 가진 예외를 생성 및 활용해 보았습니다.
```java
public class CustomRuntimeException extends RuntimeException {

    private HttpStatus httpStatus;

    private String status;

    private int errorCode;

    private String errorMessage;

    private CustomRuntimeException() {
    }

    private CustomRuntimeException(HttpStatus httpStatus, String status, int errorCode, String errorMessage) {
        super(status);
        this.httpStatus = httpStatus;
        this.status = status;
        this.errorCode = errorCode;
        this.errorMessage = errorMessage;
    }

    public static CustomRuntimeException createWithApiResponseStatus(ApiResponseStatus status) {
        return new CustomRuntimeException(status.getHttpStatus(), status.name(), status.getCode(), status.getMessage());
    }

    public static CustomRuntimeException createWithBindingResults(ApiResponseStatus status, BindingResult bindingResult) {
        return new CustomRuntimeException(status.getHttpStatus(), status.getMessage(), status.getCode(), extractErrorMessageFromBindingResults(bindingResult));
    }

    private static String extractErrorMessageFromBindingResults(BindingResult bindingResult) {
        return bindingResult.getFieldErrors()
                .stream().map(MessageSourceResolvable::getDefaultMessage)
                .collect(Collectors.toList())
                .toString();
    }

    public HttpStatus gethttpStatus() {
        return httpStatus;
    }

    public String getStatus() {
        return status;
    }

    public int getErrorCode() {
        return errorCode;
    }

    public String getErrorMessage() {
        return errorMessage;
    }


}
```

3. 응답 객체를 생성
-  응답의 성공과 에러에 따라 공통의 응답 객체를 정의하여 API의 일관성을 유지해 보았습니다.
-  또한 사용자 정의 응답 상태와 코드등 좀 더 부가적인 정보를 포함시켜 클라이언트에게 좀 더 자세한 정보를 전달해 보았습니다.
```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class ApiResponse {

    public static <T> ApiResponse.Success<T> success(T result) {
        return new Success<>(ApiResponseStatus.SUCCESS, result);
    }

    public static <T> ApiResponse.Success<T> success(ApiResponseStatus status, T result) {
        return new Success<>(status, result);
    }

    public static ApiResponse.Error error(Exception exception, String requestedPath) {
        final ApiResponseStatus error = ApiResponseStatus.SERVER_ERROR;

        final String status = error.name();
        final int errorCode = error.getCode();
        final String message = exception.getMessage() != null ? exception.getMessage() : ApiResponseStatus.SERVER_ERROR.getMessage();

        return new Error(status, errorCode, message, requestedPath);
    }

    public static ApiResponse.Error error(CustomRuntimeException customRuntimeException, String requestedPath) {

        final String status = customRuntimeException.getStatus();
        final int errorCode = customRuntimeException.getErrorCode();
        final String message = customRuntimeException.getErrorMessage();

        return new Error(status, errorCode, message, requestedPath);
    }

    @NoArgsConstructor(access = AccessLevel.PRIVATE)
    @Getter
    public static class Success<T> {
        private String status;
        private Integer code;
        private String message;
        private T result;

        private Success(ApiResponseStatus status, T result) {
            this.status = status.name();
            this.code = status.getCode();
            this.message = status.getMessage();
            this.result = result;
        }
    }

    @NoArgsConstructor(access = AccessLevel.PRIVATE)
    @Getter
    public static class Error {
        private String status;
        private Integer code;
        private String message;
        private String requestPath;

        private Error(String status, Integer code, String message, String requestPath) {
            this.status = status;
            this.code = code;
            this.message = message;
            this.requestPath = requestPath;
        }
    }


}
```

4. 전역 예외 처리기
- CustomRuntimeException의 정보를 바탕으로 클라이언트에게 필요한 정보를 담아 응답합니다.
- handleException(): 개발자가 처리하지 못한 예외를 처리하고 응답
- handleCustomRuntimeException(): 개발자가 예상하고 의도한 예외를 처리하고 응답

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionAdvisor {

    @ExceptionHandler(Exception.class)
    public final ResponseEntity<ApiResponse.Error> handleException(HttpServletRequest request, Exception e) {
        log.info("handle exception");

        return ResponseEntity
                .internalServerError()
                .body(ApiResponse.error(e, request.getRequestURI()));
    }

    @ExceptionHandler(CustomRuntimeException.class)
    public final ResponseEntity<ApiResponse.Error> handleCustomRuntimeException(HttpServletRequest request, CustomRuntimeException e) {
        log.info("handle CustomRuntimeException status: {}, code: {}, message: {}", e.getStatus(), e.getErrorCode(), e.getMessage());

        return ResponseEntity
                .status(e.gethttpStatus())
                .body(ApiResponse.error(e, request.getRequestURI()));
    }

}
```

### 6. 결과 관찰
-  사용자 정의 예외에 PERMISSION_DENIED를 사용하면 응답에 권한 없음 HttpStatus.FORBIDDEN을 사용하는 것을 테스트
```java
class GlobalExceptionAdvisorTest {

    private final GlobalExceptionAdvisor globalExceptionAdvisor;

    public GlobalExceptionAdvisorTest() {
        this.globalExceptionAdvisor = new GlobalExceptionAdvisor();
    }

    @Test
    @DisplayName("사용자 정의 예외 핸들링 테스트: 접근 제한 예외 발생")
    void customRuntimeExceptionTest() throws Exception {
        //given
        CustomRuntimeException customRuntimeException = CustomRuntimeException.createWithApiResponseStatus(ApiResponseStatus.PERMISSION_DENIED);

        HttpServletRequest request = Mockito.mock(HttpServletRequest.class);
        Mockito.when(request.getRequestURI()).thenReturn("/api/v2/posts");
        Mockito.when(request.getMethod()).thenReturn("PATCH");
        //when
        ResponseEntity<ApiResponse.Error> responseEntity = globalExceptionAdvisor.handleCustomRuntimeException(request, customRuntimeException);
        //then
        assertEquals(HttpStatus.FORBIDDEN, responseEntity.getStatusCode());

    }

}
```

### 7. 깃허브 코드
[exception](https://github.com/ImBoriPapa/bad-request/tree/main/src/main/java/com/study/badrequest/exception)
[APIResponse](https://github.com/ImBoriPapa/bad-request/blob/main/src/main/java/com/study/badrequest/commons/response/ApiResponse.java)
[APIResponseStatus](https://github.com/ImBoriPapa/bad-request/blob/main/src/main/java/com/study/badrequest/commons/response/ApiResponseStatus.java)

참고: 
> CleanCode - Robert C. Martin : 7장 오류처리

> Effective Java 3/E Bloch, Joshua: 10장 예외
