# Builder 패턴과 정적 Static Factory Method 패턴의 활용한 Member Entity 생성

개인 프로젝트에서 회원 엔티티를 관리하기 위한 정보 약 15개의 필드로 구성된 엔티티를 만들고 회원가입은 이메일과 OAuth2 등록을 구분하는 코드의 필요.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_id")
    private Long id;

    @Column(name = "authentication_code", unique = true, nullable = false)
    private String authenticationCode;

    @Column(name = "oauth_id", nullable = true)
    private String oauthId;

    @Column(name = "email", nullable = false)
    private String email;

    @OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "member_profile_id")
    private MemberProfile memberProfile;

    @Enumerated(EnumType.STRING)
    @Column(name = "registration_type")
    private RegistrationType registrationType;

    @Column(name = "password")
    private String password;

    @Column(name = "contact")
    private String contact;

    @Column(name = "authority", nullable = false)
    @Enumerated(EnumType.STRING)
    private Authority authority;

    @Column(name = "ip_address")
    private String ipAddress;
    @Column(name = "account_status")

    @Enumerated(EnumType.STRING)
    private AccountStatus accountStatus;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    @Column(name = "date_index")
    private Long dateIndex;
```

## 1. 문제
1. 회원 엔티티의 생성시 필요한 필드가 약 5개 이상으로 많은 필드를 초기화를 해야 함으로써 가독성이 떨어집니다.
2. 회원 엔티티의 생성시 이메일 가입과 Oauth2 가입의 구분하기 위해 생성자의 오버로딩으로는 생성되는 객체의 구분이 어렵습니다.
3. 생성자는 IDE의 도움이 없이는 같은 타입의 매개변수의 구분이 어려워 엉뚱한 값을 매개변수에 할당할 가능성이 있습니다.

## 2. 사실수집: 
아래 테스트 코드를 작성함으로써 문제를 확인하였습니다.
```java
    @Test
    @DisplayName("이메일로 가입 회원 엔티티 생성")
    void createWithEmailTestByConstructor() throws Exception {
        //given
        final String authenticationCode = "somethingCode";
        final String email = "email@email.com";
        final RegistrationType registrationType = RegistrationType.BAD_REQUEST;
        final String password = "thisIsPassword";
        final String contact = "01012341234";
        final Authority authority = Authority.MEMBER;
        final String ipAddress = "thisIsIpAddress";
        final AccountStatus accountStatus = AccountStatus.ACTIVE;
        final Long dateIndex = 20230704L;
        final LocalDateTime createdAt = LocalDateTime.now();
        final LocalDateTime updatedAt = LocalDateTime.now();
        final LocalDateTime deletedAt = LocalDateTime.now();
        //when
        Member createdWithEmail = new Member(authenticationCode, email, registrationType, password, contact, authority, ipAddress, accountStatus, dateIndex, createdAt, updatedAt, deletedAt);

        //then
        assertThat(createdWithEmail).isNotNull();

    }

    @Test
    @DisplayName("OAuth2 가입 회원 엔티티 생성")
    void createWithOAuth2TestByConstructor() throws Exception {
        //given
        final String authenticationCode = "somethingCode";
        final String oauthId = "421421";
        final String email = "email@google.com";
        final RegistrationType registrationType = RegistrationType.GOOGLE;
        final Authority authority = Authority.MEMBER;
        final AccountStatus accountStatus = AccountStatus.ACTIVE;
        final Long dateIndex = 20230704L;
        final LocalDateTime createdAt = LocalDateTime.now();
        final LocalDateTime updatedAt = LocalDateTime.now();
        final LocalDateTime deletedAt = LocalDateTime.now();
        //when
        Member createdWithOauth2 = new Member(authenticationCode, oauthId, email, registrationType, authority, accountStatus, dateIndex, createdAt, updatedAt, deletedAt);

        //then
        assertThat(createdWithOauth2).isNotNull();

    }
```

## 3. 원인추론: 
1. 생성자를 이용해서 5개 이상의 필드를 매개변수로 할당할때 가독성 및 엉뚱한 값을 할당할 수가 있다.
2. 생성자의 오버라이딩으로 생성하는 객체는 어떤 회원 객체를 반환하는지 구분이 힘들다.

## 4. 조사방법 결정:
- 생성자가 아닌 빌더패턴으로 가독성 항샹
- 정적 팩토리 메서드 패턴을 사용해서 의미있는 메서드의 이름을 사용

## 5. 조사방법 구현:
1. 생성자를 Builder 패턴을 사용해서 어떤 변수에 어떤 값이 할당되는 것인지 개발자가 확인하기 쉽게 가독성을 보완했습니다.
2. Static Factory Method 패턴을 사용해 의미있는 이름으로 개발자가 어떤 회원 엔티티가 생성되는지 구분하기 쉽게 보완했습니다.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_id")
    private Long id;

    @Column(name = "authentication_code", unique = true, nullable = false)
    private String authenticationCode;

    @Column(name = "oauth_id", nullable = true)
    private String oauthId;

    @Column(name = "email", nullable = false)
    private String email;

    @OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "member_profile_id")
    private MemberProfile memberProfile;

    @Enumerated(EnumType.STRING)
    @Column(name = "registration_type")
    private RegistrationType registrationType;

    @Column(name = "password")
    private String password;

    @Column(name = "contact")
    private String contact;

    @Column(name = "authority", nullable = false)
    @Enumerated(EnumType.STRING)
    private Authority authority;

    @Column(name = "ip_address")
    private String ipAddress;
    @Column(name = "account_status")

    @Enumerated(EnumType.STRING)
    private AccountStatus accountStatus;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    @Column(name = "date_index")
    private Long dateIndex;

    @Builder(access = AccessLevel.PROTECTED)
    protected Member(String oauthId, RegistrationType registrationType, String email, String password, String contact, Authority authority, String ipAddress, AccountStatus accountStatus, LocalDateTime createdAt, LocalDateTime updatedAt, LocalDateTime deletedAt) {
        this.oauthId = oauthId;
        this.registrationType = registrationType;
        this.email = email;
        this.password = password;
        this.contact = contact;
        this.authority = authority;
        this.ipAddress = ipAddress;
        this.accountStatus = accountStatus;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
        this.deletedAt = deletedAt;
    }

    public static Member createWithEmail(String email, String password, String contact) {

        Member member = Member.builder()
                .email(email)
                .oauthId(null)
                .password(password)
                .registrationType(RegistrationType.BAD_REQUEST)
                .contact(contact)
                .authority(Authority.MEMBER)
                .accountStatus(AccountStatus.ACTIVE)
                .createdAt(LocalDateTime.now())
                .updatedAt(null)
                .deletedAt(null)
                .build();

        member.generateDateTimeIndex();
        member.generateAuthenticationCode();

        return member;
    }

    public static Member createWithOauth2(String email, String oauthId, RegistrationType registrationType) {
        
        Member member = Member.builder()
                .email(email)
                .oauthId(oauthId)
                .password(null)
                .registrationType(registrationType)
                .contact(null)
                .authority(Authority.MEMBER)
                .accountStatus(AccountStatus.ACTIVE)
                .createdAt(LocalDateTime.now())
                .updatedAt(null)
                .deletedAt(null)
                .build();

        member.generateDateTimeIndex();
        member.generateAuthenticationCode();

        return member;
    }

```

## 6. 결과 관찰:
문제를 보완해서 테스트 코드를 작성하였습니다.

    @Test
    @DisplayName("이메일로 회원 생성 테스트: -> createWithEmail()")
    void createWithEmailTest() throws Exception {
        //given
        final String email = "email@email.com";
        final String password = "password1234!@";
        final String contact = "01012341234";

        Member member = Member.createWithEmail(email, password, contact);
        //when
        Member save = memberRepository.save(member);
        Member found = memberRepository.findById(save.getId()).get();
        //then
        assertThat(save).isNotNull();
        assertThat(found.getId().equals(save.getId())).isTrue();
        assertThat(found.getAuthenticationCode().equals(save.getAuthenticationCode())).isTrue();
        assertThat(found.getOauthId()).isNull();
        assertThat(found.getEmail().equals(email)).isTrue();
        assertThat(found.getRegistrationType() == RegistrationType.BAD_REQUEST).isTrue();
        assertThat(found.getPassword()).isNotNull();
        assertThat(found.getContact().equals(contact)).isTrue();
        assertThat(found.getAuthority() == Authority.MEMBER).isTrue();
        assertThat(found.getIpAddress()).isNull();
        assertThat(found.getAccountStatus() == AccountStatus.ACTIVE).isTrue();
        assertThat(found.getCreatedAt().isEqual(save.getCreatedAt())).isTrue();
        assertThat(found.getUpdatedAt().isEqual(save.getUpdatedAt())).isTrue();
        assertThat(found.getDeletedAt().isEqual(save.getDeletedAt())).isTrue();
        assertThat(found.getDateIndex().equals(save.getDateIndex())).isTrue();
    }

    @Test
    @DisplayName("Oauth2 회원 생성 테스트: -> createWithOauth2()")
    void createWithOauth2Test() throws Exception {
        //given
        final String email = "email@email.com";
        final String oauthId = "12345";
        final RegistrationType registrationType = RegistrationType.GOOGLE;

        Member member = Member.createWithOauth2(email, oauthId, registrationType);
        //when
        Member save = memberRepository.save(member);
        Member found = memberRepository.findById(save.getId()).get();
        //then
        assertThat(save).isNotNull();
        assertThat(found.getId().equals(save.getId())).isTrue();
        assertThat(found.getAuthenticationCode().equals(save.getAuthenticationCode())).isTrue();
        assertThat(found.getOauthId().equals(oauthId)).isTrue();
        assertThat(found.getEmail().equals(email)).isTrue();
        assertThat(found.getRegistrationType() == registrationType).isTrue();
        assertThat(found.getPassword()).isNull();
        assertThat(found.getContact()).isNull();
        assertThat(found.getAuthority() == Authority.MEMBER).isTrue();
        assertThat(found.getIpAddress()).isNull();
        assertThat(found.getAccountStatus() == AccountStatus.ACTIVE).isTrue();
        assertThat(found.getCreatedAt().isEqual(save.getCreatedAt())).isTrue();
        assertThat(found.getUpdatedAt().isEqual(save.getUpdatedAt())).isTrue();
        assertThat(found.getDeletedAt().isEqual(save.getDeletedAt())).isTrue();
        assertThat(found.getDateIndex().equals(save.getDateIndex())).isTrue();
    }

7. 깃허브 위치
- https://github.com/ImBoriPapa/bad-request/blob/main/src/main/java/com/study/badrequest/domain/member/Member.java

참고:
> CleanCode - Robert C. Martin : 2장 의미있는 이름

> Effective Java 3/E Bloch, Joshua: 2장 객체 생성과 파괴 

> https://velog.io/@lgsgst5613/Trouble-Shooting-%ED%8A%B8%EB%9F%AC%EB%B8%94-%EC%8A%88%ED%8C%85


