# EventListener를 활용한 Service layer의 의존성과 관심사의 분리

## 1. 문제정의
- 질문 게시글을 생성,수정,삭제하는 서비스 계층에 너무 많은 기능과 의존성이 있어 테스트 작성이 복잡하다.
- 질문 게시글의 생성 로직외에 다른 비즈니스 코드 또한 포함하고 있다.
- 기능을 추가하거나 수정해야할때 기존의 로직을 파악하고 수정해야할 확률이 높다.

## 2. 사실수집
-  아래 서비스레이어의 코드는 질문 영속 계층외에도 여러 다른 의존성을 포함하고 있습니다.
-  질문 엔티티 생성 로직외에 해시태그, 질문태그, 이미지 기능에 비즈니스 코드 또한 포함되어있습니다.

```java
@Service
@Slf4j
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class QuestionServiceImpl implements QuestionService {
    private final MemberRepository memberRepository;
    private final QuestionRepository questionRepository;
    private final QuestionTagRepository questionTagRepository;
    private final HashTagRepository hashTagRepository;
    private final QuestionImageRepository questionImageRepository;
    

    @Override
    @Transactional
    public QuestionResponse.Create createQuestionProcessing(Long memberId, QuestionRequest.Create form) {
        log.info("Create Question Processing memberId: {}, title: {}", memberId, form.getTitle());
        //1. 회원 정보 조회
        Member member = findMemberById(memberId);
        //2. 질문 엔티티 생성
        Question question = questionRepository.save(createQuestionEntity(form.getTitle(), form.getContents(), member));
        //3. 해시태그 엔티티 생성
        List<HashTag> hashTags = new ArrayList<>();

        for (String tag : form.getTags()) {
            hashTags.add(HashTag.createHashTag(tag));
        }

        List<HashTag> hashTagList = hashTagRepository.saveAll(hashTags);

        //4. 질문태그 생성
        List<QuestionTag> questionTags = new ArrayList<>();
        for (HashTag hashTag : hashTagList) {
            questionTags.add(QuestionTag.createQuestionTag(question, hashTag));
        }

        List<QuestionTag> questionTagList = questionTagRepository.saveAll(questionTags);
        
        //5. 임시 저장된 이미지 저장 상태로 변경
        if (form.getImageIds() != null) {
            questionImageRepository.findAllById(form.getImageIds()).forEach(image -> image.changeTemporaryToSaved(question));
        }
        
        return new QuestionResponse.Create(question.getId(), question.getAskedAt());
    }
```

- 테스트 코드를 작성할 때 관련된 모든 로직을 설정하고 테스트해야해서 테스트 작성이 어렵습니다.
```java
@ExtendWith(MockitoExtension.class)
class CreateQuestionProcessingTest {

    @InjectMocks
    private QuestionServiceImpl questionService;
    @Mock
    private MemberRepository memberRepository;
    @Mock
    private QuestionRepository questionRepository;
    @Mock
    private QuestionTagRepository questionTagRepository;
    @Mock
    private HashTagRepository hashTagRepository;
    @Mock
    private QuestionImageRepository questionImageRepository;

    @Test
    @DisplayName("질문글 생성 테스트")
    void createQuestionTest() throws Exception {
        //given
        final Long memberId = 123L;
        final String title = "제목입니다.";
        final String contents = "내용입니다.";
        final List<String> tags = List.of("tag1", "tag2", "tag3");
        final List<Long> imageIds = List.of(1L, 2L, 3L);
        QuestionMetrics questionMetrics = QuestionMetrics.createQuestionMetrics();
        Member member = Member.createWithEmail("email@email.com", "password1234", "01012341234");
        member.assignMemberProfile(MemberProfile.createMemberProfile("nickname", ProfileImage.createDefaultImage("image")));
        Question question = Question.createQuestion(title, contents, member, questionMetrics);
        final List<HashTag> hashTags = tags.stream().map(HashTag::createHashTag).collect(Collectors.toList());
        final List<QuestionTag> questionTags = hashTags.stream().map(hashTag -> QuestionTag.createQuestionTag(question, hashTag)).collect(Collectors.toList());
        QuestionRequest.Create form = new QuestionRequest.Create(title, contents, tags, imageIds);
        //when
        given(memberRepository.findById(any())).willReturn(Optional.of(member));
        given(questionRepository.save(any())).willReturn(question);
        given(hashTagRepository.saveAll(any())).willReturn(hashTags);
        given(questionTagRepository.saveAll(any())).willReturn(questionTags);
        given(questionImageRepository.findAllById(imageIds)).willReturn(new ArrayList<>());
        questionService.createQuestionProcessing(memberId, form);
        //then
        verify(memberRepository).findById(memberId);
        verify(questionRepository).save(question);
        verify(hashTagRepository).saveAll(hashTags);
        verify(questionTagRepository).saveAll(questionTags);
        verify(questionImageRepository).findAllById(imageIds);

    }
}
```

## 3. 원인추론
- 질문,태그,이미지 기능의 코드가 강하게 결합되어 있습니다.

## 4. 조사방법 결정
Spring의 EventListener을 활용해서 의존성과 관심사의 분리시키고 여러 기능을 각각의 서비스 레이로 분리 시켜 결합도를 낮추고 본연에 기능과 역활에 충실하도록 변경하도록 결정하였습니다.

## 5. 조사방법 구현
- 질문 서비스에서 꼭 필요한 영속 계층의 의존성만 남겨놓고 분리하였습니다.
- 기능별 비즈니스 로직을 분리하고 본연에 역활과 책임만 가지도록 변경하였습니다.
- 이벤트 리스너를 활용하여 필요한 기능들을 호출하였습니다.
- 더 작은 단위의 테스트 코드 작성하여 조금더 의미있는 테스트를 작성하였습니다.

```java
@Service
@Slf4j
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class QuestionServiceImpl implements QuestionService {
    private final MemberRepository memberRepository;
    private final QuestionRepository questionRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Override
    @Transactional
    public QuestionResponse.Create createQuestionProcessing(Long memberId, QuestionRequest.Create form) {
        log.info("Create Question Processing memberId: {}, title: {}", memberId, form.getTitle());

        checkNumberOfTags(form.getTags());

        final List<Long> imageIds = form.getImageIds() == null ? Collections.emptyList() : form.getImageIds();
        
        Member member = findMemberById(memberId);

        Question question = questionRepository.save(createQuestionEntity(form.getTitle(), form.getContents(), member));

        eventPublisher.publishEvent(new QuestionEventDto.CreateEvent(member.getId(), question.getId(), form.getTags(), imageIds));

        return new QuestionResponse.Create(question.getId(), question.getAskedAt());
    }

    private void checkNumberOfTags(List<String> tags) {
        if (tags == null || tags.isEmpty() || tags.size() > 5) {
            throw CustomRuntimeException.createWithApiResponseStatus(AT_LEAST_ONE_TAG_MUST_BE_USED_AND_AT_MOST_FIVE_TAGS_MUST_BE_USED);
        }
    }

    private Question createQuestionEntity(String title, String contents, Member member) {
        return Question.createQuestion(title, contents, member, QuestionMetrics.createQuestionMetrics());
    }

    private Member findMemberById(Long memberId) {
        return memberRepository
                .findById(memberId)
                .orElseThrow(() -> CustomRuntimeException.createWithApiResponseStatus(NOTFOUND_MEMBER));
    }
```   

이벤트 리스너를 사용해 발행된 이벤트를 수신
```java
@Component
@Slf4j
@RequiredArgsConstructor
public class QuestionCreateEventListener {
    private final QuestionTagService questionTagService;
    private final QuestionImageService questionImageService;
    private final RecordService recordService;

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void handleCreateQuestionEvent(QuestionEventDto.CreateEvent dto) {
        log.info("Question Create Event");

        final Long memberId = dto.getMemberId();
        final Long questionId = dto.getQuestionId();

        questionTagService.createQuestionTag(questionId, dto.getTags());

        recordService.recordMemberInformation(createRecordDto(memberId, questionId));

        questionImageService.changeTemporaryToSaved(questionId, dto.getImages());
    }

    private MemberRecordRequest createRecordDto(Long memberId, Long questionId) {
        final String description = "memberId: " + memberId + ", " + "questionId: " + questionId;
        return new MemberRecordRequest(ActionStatus.CREATE_QUESTION, memberId, null, description, LocalDateTime.now());
    }
}

```
- 각 기능에 역활과 책임에 맞는 서비스레이어 작성
```java
@Service
@Slf4j
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class QuestionTagServiceImpl implements QuestionTagService {

    private final QuestionTagRepository questionTagRepository;
    private final HashTagRepository hashTagRepository;
    private final QuestionRepository questionRepository;

    @Transactional
    public void createQuestionTag(Long questionId, List<String> tags) {
        log.info("질문 태그 생성 시작 - QuestionId: {}, Requested Tag name: {}", questionId, tags.toArray());
        .... Business Code
    }
}

@Service
@Transactional(readOnly = true)
@Slf4j
@RequiredArgsConstructor
public class QuestionImageServiceImpl implements QuestionImageService {
    private final ImageUploader imageUploader;
    private final QuestionRepository questionRepository;
    private final QuestionImageRepository questionImageRepository;
    private final String FOLDER_NAME = "QUESTIONS";

    @Transactional
    public void changeTemporaryToSaved(Long questionId,List<Long> imageIds) {
        log.info("질문 게시판 임시 이미지 저장완료로 변경");
       .... Business Code
    }
```


## 6. 결과 관찰
- 세분화된 테스트 작성
```java
@ExtendWith(MockitoExtension.class)
class CreateQuestionProcessingTest extends QuestionServiceTestBase {

    @Test
    @DisplayName("질문글 생성 실패 테스트1: tag가 최소 1개 이상이 아닐 경우")
    void createQuestionTest1() throws Exception {
        //given
        final Long memberId = 123L;
        final String title = "제목입니다.";
        final String contents = "내용입니다.";

        QuestionRequest.Create form = new QuestionRequest.Create(title, contents, null, new ArrayList<>());
        //when

        //then
        assertThatThrownBy(() -> questionService.createQuestionProcessing(memberId, form))
                .isInstanceOf(CustomRuntimeException.class);

    }

    @Test
    @DisplayName("질문글 생성 실패 테스트2: tag가 최대 5개 이상일 경우")
    void createQuestionTest2() throws Exception {
        //given
        final Long memberId = 123L;
        final String title = "제목입니다.";
        final String contents = "내용입니다.";
        final List<String> tags = List.of("tag1", "tag2", "tag3", "tag4", "tag5", "tag6");
        final List<Long> imageIds = null;

        QuestionRequest.Create form = new QuestionRequest.Create(title, contents, tags, imageIds);
        //when

        //then
        assertThatThrownBy(() -> questionService.createQuestionProcessing(memberId, form))
                .isInstanceOf(CustomRuntimeException.class);

    }

    @Test
    @DisplayName("질문글 생성 실패 테스트3: 회원 정보를 찾을 수 없을 경우")
    void createQuestionTest3() throws Exception {
        //given
        final Long memberId = 123L;
        final String title = "제목입니다.";
        final String contents = "내용입니다.";
        final List<String> tags = List.of("tag1", "tag2", "tag3");
        final List<Long> imageIds = null;

        QuestionRequest.Create form = new QuestionRequest.Create(title, contents, tags, imageIds);
        //when
        given(memberRepository.findById(any())).willReturn(Optional.empty());
        //then
        assertThatThrownBy(() -> questionService.createQuestionProcessing(memberId, form))
                .isInstanceOf(CustomRuntimeException.class);

    }

    @Test
    @DisplayName("질문글 생성 성공 테스트")
    void createQuestionTest() throws Exception {
        //given
        final Long memberId = 123L;
        final String title = "제목입니다.";
        final String contents = "내용입니다.";
        final List<String> tags = List.of("tag1", "tag2", "tag3");
        final List<Long> imageIds = new ArrayList<>();

        Member member = createSampleMember();
        QuestionRequest.Create form = new QuestionRequest.Create(title, contents, tags, imageIds);

        Question question = Question.createQuestion(title, contents, member, QuestionMetrics.createQuestionMetrics());

        //when
        given(memberRepository.findById(any())).willReturn(Optional.of(member));
        given(questionRepository.save(any())).willReturn(question);
        questionService.createQuestionProcessing(memberId, form);
        //then
        verify(memberRepository).findById(memberId);
        verify(questionRepository).save(question);
        verify(eventPublisher).publishEvent(any(QuestionEventDto.CreateEvent.class));

    }


    private Member createSampleMember() {
        final String email = "email@email.com";
        final String password = "password1234";
        final String contact = "01012341234";
        final String nickname = "nickname";
        final String defaultImage = "defaultImage";
        Member member = Member.createWithEmail(email, password, contact);
        member.assignMemberProfile(MemberProfile.createMemberProfile(nickname, ProfileImage.createDefaultImage(defaultImage)));
        return member;
    }
}
```

7. 깃허브 위치
[QuestionService Code](https://github.com/ImBoriPapa/bad-request/blob/main/src/main/java/com/study/badrequest/service/question/QuestionServiceImpl.java)
[EventListener Code](https://github.com/ImBoriPapa/bad-request/tree/main/src/main/java/com/study/badrequest/event/question)

참고
> CleanCode - Robert C. Martin : 10장 클래스

> https://mangkyu.tistory.com/292

