---
name: spring-code-review
description: Spring Boot 프로젝트 코드 리뷰 스킬. spring-dev-flow 패턴을 기준으로 코드 품질, 아키텍처, 보안, 성능을 검증합니다. 사용자가 "코드 리뷰", "리뷰해줘", "검토해줘", "코드 분석", "품질 점검"을 요청할 때 사용합니다.
metadata:
  author: Claude AI
  version: 2.0.0
  category: review
  tags: [java, spring-boot, code-review, quality]
---

# Spring Code Review

> `/spring-dev-flow` 패턴을 기준으로 코드를 검증합니다.

---

## 리뷰 프로세스

```
1. 프로젝트 구조 분석 → 아키텍처 확인
2. 레이어별 리뷰 → Entity → Repository → Service → Controller → DTO
3. 횡단 관심사 → 보안, 예외 처리, 트랜잭션
4. 성능 점검 → N+1, 캐싱, 쿼리 최적화
5. 결과 리포트 → 우선순위별 정리
```

---

## 1. Entity 체크리스트

### 필수 패턴
```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // ✅ 필수
@Builder(access = AccessLevel.PRIVATE)               // ✅ 필수
@AllArgsConstructor(access = AccessLevel.PRIVATE)   // ✅ 필수
public class Entity extends BaseTimeEntity {        // ✅ BaseTimeEntity 상속
    // ...
}
```

### 체크 항목
| 항목 | 검증 기준 | 위반 시 |
|------|----------|---------|
| 생성자 접근 제한 | `@NoArgsConstructor(access = PROTECTED)` | ❌ 외부에서 new 가능 |
| Builder 접근 제한 | `@Builder(access = PRIVATE)` | ❌ 정적 팩토리 우회 가능 |
| Setter 사용 | `@Setter` 금지 | ❌ 불변성 위반 |
| 정적 팩토리 | `createXxx()` 메서드 존재 | ❌ 생성 로직 분산 |
| 비즈니스 메서드 | 상태 변경은 메서드로 | ❌ 외부에서 직접 수정 |
| BaseTimeEntity | 상속 여부 | ❌ created/updated 누락 |
| FetchType | `LAZY` 기본 | ⚠️ N+1 위험 |

### ❌ Bad
```java
@Entity
@Getter
@Setter  // ❌ Setter 사용
@NoArgsConstructor  // ❌ 접근 제한 없음
public class User {
    @ManyToOne  // ❌ 기본 EAGER
    private Team team;
}
```

### ✅ Good
```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder(access = AccessLevel.PRIVATE)
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class User extends BaseTimeEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;

    public static User createUser(String name, Team team) {
        return User.builder()
                .name(name)
                .team(team)
                .build();
    }

    public void updateName(String name) {
        this.name = name;
    }
}
```

---

## 2. Repository 체크리스트

### 필수 패턴
```java
// 기본 + Custom 분리
public interface UserRepository
    extends JpaRepository<User, Long>, UserRepositoryCustom {
}

// Custom 인터페이스
public interface UserRepositoryCustom {
    Page<UserInfo> findAllWithFilter(SearchParam param, Pageable pageable);
}

// QueryDSL 구현체
@Repository
@RequiredArgsConstructor
public class UserRepositoryCustomImpl implements UserRepositoryCustom {
    private final JPAQueryFactory queryFactory;
    // ...
}
```

### 체크 항목
| 항목 | 검증 기준 | 위반 시 |
|------|----------|---------|
| Custom Repository | 복잡 쿼리 분리 | ❌ Repository 비대화 |
| fetchJoin | 연관 엔티티 조회 시 | ❌ N+1 문제 |
| BooleanExpression | 동적 조건 메서드 분리 | ❌ 쿼리 중복 |
| 페이징 | offset/limit + 별도 count | ⚠️ 성능 이슈 |

### ❌ Bad (N+1 문제)
```java
// Repository
List<Order> findByUserId(Long userId);

// Service - N+1 발생!
List<Order> orders = orderRepository.findByUserId(userId);
orders.forEach(order -> {
    order.getItems().size();  // N개의 추가 쿼리
});
```

### ✅ Good (fetchJoin)
```java
@Override
public List<Order> findByUserIdWithItems(Long userId) {
    return queryFactory
            .selectFrom(order)
            .join(order.items, item).fetchJoin()  // ✅ fetchJoin
            .where(order.user.id.eq(userId))
            .fetch();
}
```

---

## 3. Service 체크리스트

### 필수 패턴
```java
@Service
@Transactional(readOnly = true)  // ✅ 클래스 레벨: 읽기 전용
@RequiredArgsConstructor
public class UserService {

    @Transactional  // ✅ 쓰기 메서드만 추가
    public Long create(CreateRequest request) {
        // ...
    }
}
```

### 체크 항목
| 항목 | 검증 기준 | 위반 시 |
|------|----------|---------|
| 클래스 트랜잭션 | `@Transactional(readOnly = true)` | ⚠️ 성능 손실 |
| 쓰기 트랜잭션 | 메서드에 `@Transactional` | ❌ 트랜잭션 없이 쓰기 |
| 예외 처리 | `BusinessException` + `ErrorCode` | ❌ 일관성 없는 예외 |
| 상태 변경 | Entity 비즈니스 메서드 사용 | ❌ Service에서 직접 수정 |
| Dirty Checking | 명시적 save 불필요 | ⚠️ 불필요한 save 호출 |

### ❌ Bad
```java
@Service
public class UserService {  // ❌ 트랜잭션 없음

    public void update(Long id, String name) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Not found"));  // ❌ 일반 예외
        user.setName(name);  // ❌ Setter 사용
        userRepository.save(user);  // ❌ 불필요한 save
    }
}
```

### ✅ Good
```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class UserService {

    @Transactional
    public void update(Long id, UpdateRequest request) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));

        user.updateName(request.name());  // ✅ 비즈니스 메서드
        // Dirty Checking으로 자동 저장
    }
}
```

---

## 4. DTO 체크리스트

### 필수 패턴

**Request DTO** - 순수 record
```java
public record CreateUserRequest(
        @NotBlank(message = "이름은 필수입니다")
        String name,

        @Email(message = "이메일 형식이 올바르지 않습니다")
        String email
) {
}
```

**Response DTO** - Builder + of() 팩토리
```java
@Builder(access = AccessLevel.PRIVATE)
public record UserResponse(
        Long id,
        String name,

        @JsonInclude(JsonInclude.Include.NON_NULL)
        String email
) {
    public static UserResponse of(User user) {
        return UserResponse.builder()
                .id(user.getId())
                .name(user.getName())
                .email(user.getEmail())
                .build();
    }
}
```

### 체크 항목
| 항목 | 검증 기준 | 위반 시 |
|------|----------|---------|
| Request | `record` + Bean Validation | ❌ 불변성/검증 누락 |
| Response | `@Builder(access = PRIVATE)` + `of()` | ❌ 생성 방식 불일치 |
| JsonInclude | `NON_NULL` / `NON_EMPTY` 적용 | ⚠️ 불필요한 null 전송 |
| Entity 노출 | Response에 Entity 직접 사용 금지 | ❌ 보안 위험 |

---

## 5. Controller 체크리스트

### 필수 패턴
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    @PostMapping
    public ApiResponse<Long> create(
            @Valid @RequestBody CreateUserRequest request  // ✅ @Valid
    ) {
        return ApiResponse.ok(userService.create(request));  // ✅ ApiResponse 래핑
    }
}
```

### 체크 항목
| 항목 | 검증 기준 | 위반 시 |
|------|----------|---------|
| 응답 래핑 | `ApiResponse<T>` | ❌ 응답 형식 불일치 |
| 요청 검증 | `@Valid @RequestBody` | ❌ 검증 누락 |
| 페이징 | `@PageableDefault` 사용 | ⚠️ 기본값 불명확 |
| 비즈니스 로직 | Controller에 로직 없음 | ❌ 레이어 침범 |

### ❌ Bad
```java
@PostMapping
public User create(@RequestBody CreateUserRequest request) {  // ❌ Entity 직접 반환, @Valid 누락
    User user = new User();  // ❌ Controller에서 Entity 생성
    user.setName(request.name());
    return userRepository.save(user);  // ❌ Repository 직접 호출
}
```

### ✅ Good
```java
@PostMapping
public ApiResponse<Long> create(
        @Valid @RequestBody CreateUserRequest request
) {
    Long id = userService.create(request);
    return ApiResponse.ok(id);
}
```

---

## 6. 예외 처리 체크리스트

### 필수 패턴
```java
// ErrorCode Enum
@Getter
@RequiredArgsConstructor
public enum ErrorCode {
    USER_NOT_FOUND("USER_001", HttpStatus.NOT_FOUND, "사용자를 찾을 수 없습니다."),
    DUPLICATE_EMAIL("USER_002", HttpStatus.CONFLICT, "이미 존재하는 이메일입니다.");

    private final String code;
    private final HttpStatus httpStatus;
    private final String message;
}

// BusinessException
throw new BusinessException(ErrorCode.USER_NOT_FOUND);
```

### 체크 항목
| 항목 | 검증 기준 | 위반 시 |
|------|----------|---------|
| 예외 클래스 | `BusinessException` 사용 | ❌ 일관성 없음 |
| ErrorCode | Enum으로 관리 | ❌ 하드코딩된 메시지 |
| GlobalExceptionHandler | `@RestControllerAdvice` | ❌ 예외 처리 분산 |
| 민감정보 | 에러 응답에 노출 금지 | ❌ 보안 위험 |

---

## 7. 보안 체크리스트

### 체크 항목
| 항목 | 검증 기준 | 위반 시 |
|------|----------|---------|
| SQL Injection | 파라미터 바인딩 사용 | ❌ 심각한 보안 취약점 |
| XSS | 입력값 이스케이프 | ❌ 스크립트 삽입 위험 |
| 비밀번호 | `PasswordEncoder` 사용 | ❌ 평문 저장 |
| 민감정보 | 하드코딩 금지 | ❌ 정보 노출 |
| CORS | 명시적 설정 | ⚠️ 의도치 않은 접근 |

### ❌ Bad (SQL Injection)
```java
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)
User findByName(String name);  // ❌ 위험!
```

### ✅ Good
```java
@Query("SELECT u FROM User u WHERE u.name = :name")
User findByName(@Param("name") String name);  // ✅ 파라미터 바인딩
```

---

## 8. 성능 체크리스트

### 체크 항목
| 항목 | 검증 기준 | 위반 시 |
|------|----------|---------|
| N+1 | fetchJoin 또는 @EntityGraph | ❌ 쿼리 폭발 |
| Lazy Loading | `FetchType.LAZY` 기본 | ⚠️ 불필요한 로딩 |
| 페이징 | offset/limit 사용 | ⚠️ 전체 로딩 |
| 캐싱 | `@Cacheable` 적절히 사용 | ⚠️ 반복 쿼리 |
| @DynamicUpdate | 수정 필드만 UPDATE | ⚠️ 불필요한 UPDATE |

---

## 리뷰 결과 출력 형식

### 요약
```
📊 코드 리뷰 결과
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ 양호: X건  ⚠️ 개선 권장: Y건  ❌ 필수 수정: Z건
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 상세 결과
```
❌ [필수] Entity - Setter 사용
   위치: User.java:15
   문제: @Setter 어노테이션 사용
   영향: 불변성 위반, 상태 추적 어려움
   해결: 비즈니스 메서드로 변경

   // Before
   user.setName(newName);

   // After
   user.updateName(newName);
```

### 우선순위
1. **❌ 필수 수정** - 보안, 데이터 무결성 관련
2. **⚠️ 개선 권장** - 성능, 유지보수성 관련
3. **💡 제안** - 베스트 프랙티스 권장

---

## 리뷰 실행 방법

1. **전체 리뷰**: 프로젝트 전체 분석
2. **파일 리뷰**: 특정 파일만 분석
3. **PR 리뷰**: 변경된 파일만 분석

```
사용자: "이 코드 리뷰해줘"
→ 해당 파일/변경사항에 대해 위 체크리스트 기준으로 검증
```
