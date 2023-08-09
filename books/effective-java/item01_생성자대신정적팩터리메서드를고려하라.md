## 정적 팩터리 메서드 패턴(static factory method)

- 클래스의 인스턴스를 얻는 전통적인 수단은 public 생성자를 사용하는 것이다.
- 클래스는 인스턴스를 반환하는 단순한 정적 메서드를 제공할 수 있다라는 것을 프로그래머는 꼭 알아두고 있어야 한다.

## 장점
### 1.이름을 가질 수 있다.

전통적인 public 생성자
- 회원 가입시 Email을 사용해서 User 객체를 생성하는 생성자와 OAuth2를 이용해서 객체를 생성하는 생성자는 주석이 없다면 구분하기 어렵다.

```java
public class User {
    private Long id;
    private String oauthId;
    private String email;
    private String username;
    private String password;
    private RegistrationType registrationType;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private LocalDateTime deletedAt;
    
    // Create User By Email
    public User(String email,String username, String password) {
        this.email = email;
        this.username = username;
        this.password = password;
        this.registrationType = BY_EMAIL;
        this.createdAt = LocalDateTime.now();
    }
    
    // Create User By OAuth2
    public User(String email,String oauthId, String username, RegistrationType registrationType) {
        this.email = email;
        this.oauthId = oauthId;
        this.username = username;
        this.registrationType = registrationType;
        this.createdAt = LocalDateTime.now();
    }
}
```
- 정적 팩터리 메서드를 사용해서 조금 더 개발자가 알아보기 쉬운 이름으로 인스턴스를 생성할 수 있다.
```java
    public static User createByEmail(String email, String username, String password) {
        return new User(email, username, password);
    }

    public static User createByOauth2(String email, String oauthId, String username, RegistrationType registrationType) {
        return new User(email, oauthId, username, registrationType);
    }
```

### 2. 호출될 때마다 인스턴스를 새로 생성하지는 않아도 된다.
### 3. 반환 타입의 하위 타입 객체를 반환할 수 있는 능력이 있다.
### 4. 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.
### 5. 정적 팩터리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.
  
