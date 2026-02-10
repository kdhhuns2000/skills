---
name: spring-dev-flow
description: Java Spring Boot 프로젝트 개발 플로우를 가이드합니다. Entity, Repository, Service, Controller, DTO 설계 및 구현을 도와줍니다. 사용자가 "Spring 코드 작성", "엔티티 만들어줘", "서비스 계층 구현", "컨트롤러 작성", "Spring 테스트", "코드 리뷰", "JPA 엔티티", "REST API 구현", "QueryDSL", "DTO 변환"을 요청할 때 사용합니다.
metadata:
  author: Claude AI
  version: 2.0.0
  category: development
  tags: [java, spring-boot, jpa, querydsl, rest-api, testing]
---

# Spring Development Flow

Spring Boot 개발 가이드입니다.

## 핵심 원칙

1. **불변성 우선**: Protected 생성자 + Private Builder 패턴
2. **계층 분리**: Entity → Repository (+ Custom) → Service → Controller
3. **타입 안전성**: Record DTO + ErrorCode Enum
4. **통일된 응답**: `ApiResponse<T>` 래퍼
5. **N+1 방지**: QueryDSL fetchJoin 활용

---

## Step 1: Entity 설계

### 필수 어노테이션 패턴

```java
@Entity
@Table(name = "테이블명", schema = "스키마명")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder(access = AccessLevel.PRIVATE)
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class EntityName extends BaseTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "entity_idx")
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(name = "is_deleted")
    private Boolean isDeleted = false;

    // 연관관계
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_idx")
    private ParentEntity parent;

    @OneToMany(mappedBy = "entity", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<ChildEntity> children = new HashSet<>();

    // 정적 팩토리 메서드 (생성)
    public static EntityName createEntity(CreateRequestDto request, ParentEntity parent) {
        return EntityName.builder()
                .name(request.name())
                .parent(parent)
                .isDeleted(false)
                .build();
    }

    // 비즈니스 메서드 (상태 변경)
    public void updateName(String name) {
        this.name = name;
    }

    public void delete() {
        this.isDeleted = true;
    }
}
```

### BaseTimeEntity (공통 상속)

```java
@MappedSuperclass
@DynamicUpdate
@EntityListeners(AuditingEntityListener.class)
@Getter
public abstract class BaseTimeEntity {

    @CreatedDate
    @Column(name = "created", updatable = false)
    private LocalDateTime created;

    @LastModifiedDate
    @Column(name = "updated")
    private LocalDateTime updated;
}
```

### 체크리스트
- [ ] `@NoArgsConstructor(access = AccessLevel.PROTECTED)` - new 생성 방지
- [ ] `@Builder(access = AccessLevel.PRIVATE)` - 정적 팩토리로만 생성
- [ ] `@AllArgsConstructor(access = AccessLevel.PRIVATE)` - Builder 내부용
- [ ] `BaseTimeEntity` 상속 (created, updated 자동 관리)
- [ ] `@Setter` 사용 금지 → 비즈니스 메서드 사용
- [ ] 연관관계: `FetchType.LAZY` 기본, 필요시 `EAGER`
- [ ] `cascade = CascadeType.ALL, orphanRemoval = true` 라이프사이클 공유

### Enum 필드

```java
@Enumerated(EnumType.STRING)
@Column(name = "status", nullable = false)
private Status status;
```

---

## Step 2: Repository 계층

### 기본 Repository + Custom Repository 패턴

```java
// 1. 기본 Repository 인터페이스
@Repository
public interface EntityRepository extends JpaRepository<EntityName, Long>, EntityRepositoryCustom {
}

// 2. Custom 인터페이스 정의
public interface EntityRepositoryCustom {
    Optional<EntityName> findByNameAndParentId(String name, Long parentId);
    Page<EntityInfo> findAllWithFilter(SearchParam param, Pageable pageable);
}

// 3. Custom 구현체 (QueryDSL)
@Repository
@RequiredArgsConstructor
public class EntityRepositoryCustomImpl implements EntityRepositoryCustom {

    private final JPAQueryFactory queryFactory;
    // 다중 데이터소스의 경우: @Qualifier("clientJpaQueryFactory")

    @Override
    public Optional<EntityName> findByNameAndParentId(String name, Long parentId) {
        EntityName result = queryFactory
                .selectFrom(entityName)
                .join(entityName.parent, parent).fetchJoin()  // N+1 방지
                .where(
                    entityName.name.eq(name),
                    entityName.parent.id.eq(parentId),
                    entityName.isDeleted.eq(false)
                )
                .fetchOne();

        return Optional.ofNullable(result);
    }

    @Override
    public Page<EntityInfo> findAllWithFilter(SearchParam param, Pageable pageable) {
        List<EntityInfo> content = queryFactory
                .select(new QEntityInfo(
                    entityName.id,
                    entityName.name,
                    entityName.created
                ))
                .from(entityName)
                .where(
                    containsKeyword(param.keyword()),
                    eqStatus(param.status())
                )
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .orderBy(entityName.id.desc())
                .fetch();

        Long total = queryFactory
                .select(entityName.count())
                .from(entityName)
                .where(
                    containsKeyword(param.keyword()),
                    eqStatus(param.status())
                )
                .fetchOne();

        return new PageImpl<>(content, pageable, total != null ? total : 0L);
    }

    // 동적 조건 메서드
    private BooleanExpression containsKeyword(String keyword) {
        return StringUtils.hasText(keyword) ? entityName.name.contains(keyword) : null;
    }

    private BooleanExpression eqStatus(Status status) {
        return status != null ? entityName.status.eq(status) : null;
    }
}
```

### 체크리스트
- [ ] `JpaRepository<Entity, ID>` + `CustomRepository` 상속
- [ ] QueryDSL `fetchJoin()` 사용 (N+1 방지)
- [ ] 동적 조건은 `BooleanExpression` 메서드로 분리
- [ ] `null` 반환시 조건 무시됨 (동적 쿼리)
- [ ] 페이징: `offset()`, `limit()`, 별도 count 쿼리

---

## Step 3: Service 계층

### 기본 Service 패턴

```java
@Service
@Transactional(readOnly = true)  // 클래스 레벨: 읽기 전용
@RequiredArgsConstructor
public class EntityService {

    private final EntityRepository entityRepository;
    private final ParentRepository parentRepository;

    // 단건 조회
    public EntityDetailResponseDto findById(Long id) {
        EntityName entity = entityRepository.findById(id)
                .orElseThrow(() -> new BusinessException(ErrorCode.ENTITY_NOT_FOUND));

        return EntityDetailResponseDto.of(entity);
    }

    // 목록 조회 (페이징)
    public Page<EntityListResponseDto> findAll(SearchParam param, Pageable pageable) {
        return entityRepository.findAllWithFilter(param, pageable)
                .map(EntityListResponseDto::of);
    }

    // 생성
    @Transactional  // 메서드 레벨: 쓰기 활성화
    public Long create(Long parentId, EntityCreateRequestDto request) {
        ParentEntity parent = parentRepository.findById(parentId)
                .orElseThrow(() -> new BusinessException(ErrorCode.PARENT_NOT_FOUND));

        validateDuplicate(request.name(), parentId);

        EntityName entity = EntityName.createEntity(request, parent);
        EntityName saved = entityRepository.save(entity);

        return saved.getId();
    }

    // 수정
    @Transactional
    public void update(Long id, EntityUpdateRequestDto request) {
        EntityName entity = entityRepository.findById(id)
                .orElseThrow(() -> new BusinessException(ErrorCode.ENTITY_NOT_FOUND));

        entity.updateName(request.name());
        // Dirty Checking으로 자동 저장 (@DynamicUpdate로 수정 필드만 UPDATE)
    }

    // 삭제 (소프트 삭제)
    @Transactional
    public void delete(Long id) {
        EntityName entity = entityRepository.findById(id)
                .orElseThrow(() -> new BusinessException(ErrorCode.ENTITY_NOT_FOUND));

        entity.delete();  // isDeleted = true
    }

    // 검증 메서드
    private void validateDuplicate(String name, Long parentId) {
        entityRepository.findByNameAndParentId(name, parentId)
                .ifPresent(e -> {
                    throw new BusinessException(ErrorCode.DUPLICATE_ENTITY);
                });
    }
}
```

### 체크리스트
- [ ] 클래스 레벨 `@Transactional(readOnly = true)`
- [ ] 쓰기 메서드만 `@Transactional` 추가
- [ ] `@RequiredArgsConstructor` 생성자 주입
- [ ] 조회 실패시 `BusinessException` + `ErrorCode`
- [ ] Entity의 비즈니스 메서드로 상태 변경
- [ ] Dirty Checking 활용 (명시적 save 불필요)

---

## Step 4: DTO 설계

### Request DTO (순수 record)

```java
public record EntityCreateRequestDto(
        @NotBlank(message = "이름은 필수입니다")
        @Size(max = 100, message = "이름은 100자 이하여야 합니다")
        String name,

        @NotNull(message = "타입은 필수입니다")
        EntityType type,

        String description
) {
}
```

### Response DTO (Builder + 정적 팩토리)

```java
@Builder(access = AccessLevel.PRIVATE)
public record EntityDetailResponseDto(
        Long id,
        String name,
        EntityType type,
        String parentName,
        LocalDateTime created,

        @JsonInclude(JsonInclude.Include.NON_NULL)
        String description,

        @JsonInclude(JsonInclude.Include.NON_EMPTY)
        List<ChildInfo> children
) {
    public static EntityDetailResponseDto of(EntityName entity) {
        return EntityDetailResponseDto.builder()
                .id(entity.getId())
                .name(entity.getName())
                .type(entity.getType())
                .parentName(entity.getParent() != null ? entity.getParent().getName() : null)
                .created(entity.getCreated())
                .description(entity.getDescription())
                .children(entity.getChildren().stream()
                        .map(ChildInfo::of)
                        .toList())
                .build();
    }
}
```

### Info DTO (내부 전달용)

```java
public record EntityInfo(
        Long id,
        String name,
        LocalDateTime created
) {
    // QueryDSL Projection용 생성자
}
```

### 체크리스트
- [ ] Request DTO: 순수 `record` + Bean Validation
- [ ] Response DTO: `@Builder(access = AccessLevel.PRIVATE)` + `of()` 메서드
- [ ] `@JsonInclude(NON_NULL)` / `@JsonInclude(NON_EMPTY)` 선택적 필드
- [ ] Info DTO: QueryDSL Projection 또는 내부 전달용

---

## Step 5: Controller 계층

### 기본 Controller 패턴

```java
@RestController
@RequestMapping("/api/v1/entities")
@RequiredArgsConstructor
public class EntityController {

    private final EntityService entityService;

    // 단건 조회
    @GetMapping("/{id}")
    public ApiResponse<EntityDetailResponseDto> findById(@PathVariable Long id) {
        return ApiResponse.ok(entityService.findById(id));
    }

    // 목록 조회 (페이징)
    @GetMapping
    public ApiResponse<Page<EntityListResponseDto>> findAll(
            @ModelAttribute SearchParam param,
            @PageableDefault(size = 20, sort = "id", direction = Sort.Direction.DESC) Pageable pageable
    ) {
        return ApiResponse.ok(entityService.findAll(param, pageable));
    }

    // 생성
    @PostMapping
    public ApiResponse<Long> create(
            @PathVariable Long parentId,
            @Valid @RequestBody EntityCreateRequestDto request
    ) {
        Long id = entityService.create(bigCampaignId, request);
        return ApiResponse.ok(id);
    }

    // 수정
    @PutMapping("/{id}")
    public ApiResponse<Void> update(
            @PathVariable Long id,
            @Valid @RequestBody EntityUpdateRequestDto request
    ) {
        entityService.update(id, request);
        return ApiResponse.ok(null);
    }

    // 삭제
    @DeleteMapping("/{id}")
    public ApiResponse<Void> delete(@PathVariable Long id) {
        entityService.delete(id);
        return ApiResponse.ok(null);
    }
}
```

### ApiResponse 래퍼

```java
public record ApiResponse<T>(
        boolean success,
        T data
) {
    public static <T> ApiResponse<T> ok(@Nullable final T data) {
        return new ApiResponse<>(true, data);
    }

    public static ApiResponse<ExceptionResponse> error(final ExceptionResponse exceptionResponse) {
        return new ApiResponse<>(false, exceptionResponse);
    }
}
```

### 체크리스트
- [ ] 모든 응답 `ApiResponse<T>` 래핑
- [ ] `@Valid @RequestBody` 요청 검증
- [ ] `@PageableDefault` 기본 페이징 설정
- [ ] 필요시 커스텀 어노테이션 활용

---

## Step 6: 예외 처리

### BusinessException

```java
@Getter
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super();
        this.errorCode = errorCode;
    }

    @Override
    public synchronized Throwable fillInStackTrace() {
        return this;  // 성능 최적화: 스택트레이스 생략
    }
}
```

### ErrorCode Enum

```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {
    // 인증
    INVALID_TOKEN("AUTH_001", HttpStatus.UNAUTHORIZED, "권한이 없습니다."),

    // 리소스 Not Found
    ENTITY_NOT_FOUND("ENTITY_001", HttpStatus.NOT_FOUND, "엔티티를 찾을 수 없습니다."),
    PARENT_NOT_FOUND("ENTITY_002", HttpStatus.NOT_FOUND, "상위 항목을 찾을 수 없습니다."),

    // 중복/충돌
    DUPLICATE_ENTITY("ENTITY_003", HttpStatus.CONFLICT, "이미 존재하는 항목입니다."),

    // Rate Limit
    RATE_LIMIT_EXCEEDED("RATE_001", HttpStatus.TOO_MANY_REQUESTS, "요청 횟수를 초과했습니다."),

    // 파일
    FILE_SIZE_EXCEEDED("FILE_001", HttpStatus.BAD_REQUEST, "파일 크기가 제한을 초과했습니다."),
    UNSUPPORTED_FILE_TYPE("FILE_002", HttpStatus.BAD_REQUEST, "지원하지 않는 파일 형식입니다."),

    // 검증
    INVALID_INPUT("VALID_001", HttpStatus.BAD_REQUEST, "입력값이 올바르지 않습니다."),

    // 서버 오류
    INTERNAL_SERVER_ERROR("SERVER_001", HttpStatus.INTERNAL_SERVER_ERROR, "서버 내부 오류입니다.");

    private final String code;
    private final HttpStatus httpStatus;
    private final String message;
}
```

### GlobalExceptionHandler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    protected ResponseEntity<ApiResponse<ExceptionResponse>> handleBusinessException(BusinessException e) {
        log.warn("Business error - code: {}", e.getErrorCode().getCode());
        return createExceptionResponse(e.getErrorCode());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    protected ResponseEntity<ApiResponse<ExceptionResponse>> handleValidation(MethodArgumentNotValidException e) {
        String detail = e.getBindingResult().getFieldErrors().stream()
                .map(error -> error.getField() + ": " + error.getDefaultMessage())
                .collect(Collectors.joining(", "));

        log.warn("Validation error: {}", detail);
        return createExceptionResponse(ErrorCode.METHOD_ARGUMENT_NOT_VALID, detail);
    }

    @ExceptionHandler(Exception.class)
    protected ResponseEntity<ApiResponse<ExceptionResponse>> handleException(Exception e) {
        log.error("Unexpected error", e);
        return createExceptionResponse(ErrorCode.INTERNAL_SERVER_ERROR);
    }

    private ResponseEntity<ApiResponse<ExceptionResponse>> createExceptionResponse(
            ErrorCode errorCode, String... details) {
        ExceptionResponse response = ExceptionResponse.of(errorCode, details);
        return ResponseEntity.status(errorCode.getHttpStatus())
                .body(ApiResponse.error(response));
    }
}
```

### ExceptionResponse

```java
@Builder(access = AccessLevel.PRIVATE)
public record ExceptionResponse(
        String code,
        String message,

        @JsonInclude(JsonInclude.Include.NON_EMPTY)
        String detail
) {
    public static ExceptionResponse of(ErrorCode errorCode, String... details) {
        return ExceptionResponse.builder()
                .code(errorCode.getCode())
                .message(errorCode.getMessage())
                .detail(details.length > 0 ? String.join(" ", details) : null)
                .build();
    }
}
```

---

## Step 7: 테스트 작성

### Controller 테스트 (WebMvcTest + RestDocs)

```java
@WebMvcTest(EntityController.class)
@AutoConfigureMockMvc
@AutoConfigureRestDocs
@ExtendWith(RestDocumentationExtension.class)
class EntityControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private EntityService entityService;

    @MockitoBean
    private AuthInterceptor authInterceptor;

    @Autowired
    private ObjectMapper objectMapper;

    @BeforeEach
    void setUp(WebApplicationContext context, RestDocumentationContextProvider restDoc) throws Exception {
        doReturn(true).when(authInterceptor).preHandle(any(), any(), any());

        this.mockMvc = MockMvcBuilders.webAppContextSetup(context)
                .addFilters(new CharacterEncodingFilter("UTF-8", true))
                .apply(documentationConfiguration(restDoc)
                        .operationPreprocessors()
                        .withRequestDefaults(prettyPrint())
                        .withResponseDefaults(prettyPrint()))
                .build();
    }

    @Test
    @DisplayName("GET /api/v1/entities/{id} - 성공")
    void findById_Success() throws Exception {
        // given
        EntityDetailResponseDto response = EntityDetailResponseDto.builder()
                .id(1L)
                .name("테스트")
                .build();
        given(entityService.findById(1L)).willReturn(response);

        // when & then
        mockMvc.perform(get("/api/v1/entities/{id}", 1L)
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data.id").value(1L))
                .andExpect(jsonPath("$.data.name").value("테스트"))
                .andDo(print());
    }

    @Test
    @DisplayName("POST /api/v1/entities - 생성 성공")
    void create_Success() throws Exception {
        // given
        EntityCreateRequestDto request = new EntityCreateRequestDto("새엔티티", EntityType.TYPE_A, null);
        given(entityService.create(anyLong(), any())).willReturn(1L);

        // when & then
        mockMvc.perform(post("/api/v1/entities")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data").value(1L));
    }
}
```

### TestEntityFactory (리플렉션 활용)

```java
public class TestEntityFactory {

    public static EntityName createEntity(Long id, String name) {
        try {
            Constructor<EntityName> constructor = EntityName.class.getDeclaredConstructor();
            constructor.setAccessible(true);
            EntityName entity = constructor.newInstance();

            setField(entity, "id", id);
            setField(entity, "name", name);
            setField(entity, "isDeleted", false);

            return entity;
        } catch (Exception e) {
            throw new RuntimeException("Failed to create test entity", e);
        }
    }

    private static void setField(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```

---

## 워크플로우 요약

```
1. Entity 설계
   └─ Protected 생성자 + Private Builder
   └─ 정적 팩토리 메서드 (createXxx)
   └─ 비즈니스 메서드 (update, delete)
        ↓
2. Repository 작성
   └─ JpaRepository + CustomRepository
   └─ QueryDSL로 복잡 쿼리 분리
   └─ fetchJoin으로 N+1 방지
        ↓
3. Service 구현
   └─ 클래스: @Transactional(readOnly = true)
   └─ 쓰기 메서드: @Transactional
   └─ BusinessException 예외 처리
        ↓
4. DTO 설계
   └─ Request: record + @Valid
   └─ Response: @Builder + of() 팩토리
        ↓
5. Controller 작성
   └─ ApiResponse<T> 래핑
   └─ 커스텀 어노테이션 활용
        ↓
6. 테스트 작성
   └─ @WebMvcTest + MockitoBean
   └─ RestDocs 문서화
```

---

## 코드 리뷰 체크리스트

### 필수 점검
- [ ] Entity: Protected 생성자 + Private Builder 패턴 적용
- [ ] Entity: @Setter 대신 비즈니스 메서드 사용
- [ ] Repository: N+1 문제 없음 (fetchJoin 확인)
- [ ] Service: 트랜잭션 범위 적절성
- [ ] Controller: ApiResponse 래핑
- [ ] 예외: BusinessException + ErrorCode 사용
- [ ] DTO: Request는 record, Response는 of() 팩토리

### 성능 점검
- [ ] @DynamicUpdate 적용 (수정 필드만 UPDATE)
- [ ] FetchType.LAZY 기본 사용
- [ ] 페이징 쿼리 최적화
- [ ] 불필요한 데이터 조회 없음
