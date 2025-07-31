# @DynamicUpdate

## 정의

`@DynamicUpdate`는 JPA/Hibernate에서 제공하는 애노테이션으로, 엔티티 업데이트 시 **변경된 컬럼만 UPDATE쿼리**에 포함시키는 기능입니다.

기본적으로 JPA/Hibernate는 모든 컬럼을 포함한 고정된 UPDATE쿼리를 사용하지만, 이 애노테이션을 사용하면 실제로 변경된 필드만을 대상으로 동적 쿼리를 생성합니다.

## 기본 JPA/Hibernate 동작 방식

### 일반적인 UPDATE 쿼리 (기본값)

```java
@Entity
public class User {
    private String name;
    private String email;
    private String phone;
    private Date lastLoginDate;
    // ... 기타 필드들
}

// 이메일만 변경하는 경우
User user = userRepository.findById(1L);
user.setEmail("new@email.com");
userRepository.save(user);
```

**생성되는 SQL**:

```sql
UPDATE users SET
    name = ?,
    email = ?,
    phone = ?,
    last_login_date = ?,
    -- ... 모든 컬럼
WHERE id = ?
```

### 왜 이렇게 동작할까?

JPA/Hibernate가 모든 컬럼을 포함한 고정 쿼리를 사용하는 이유는 **PreparedStatement의 캐싱 효과 때문**입니다.

**PreparedStatement 캐싱의 이점**:

- SQL 파싱 비용 절약
- 실행 계획 재사용으로 최적화 비용 절약
- 동일한 쿼리 구조로 캐시 적중률 향상

```sql
-- 항상 같은 구조의 쿼리 = 캐시 재사용 가능
UPDATE users SET name=?, email=?, phone=?, last_login_date=? WHERE id=?
```

## @DynamicUpdate 적용

### 사용법

```java
@Entity
@DynamicUpdate
public class User {
    private String name;
    private String email;
    private String phone;
    private Date lastLoginDate;
    // ... 기타 필드들
}
```

**동일한 코드로 이메일만 변경 시**:

```sql
-- 변경된 컬럼만 포함
UPDATE users SET email = ? WHERE id = ?
```

## 트레이드 오프: 언제 사용해야 할까?

### ✅ @DynamicUpdate를 사용하는 경우

**테이블 특성**:

- 컬럼 수가 많은 테이블 (20개 이상)
- 큰 사이즈 컬럼 존재 (TEXT, BLOB, JSON)
- 인덱스가 많이 걸린 테이블

**업데이트 패턴**:

- 전체 필드 중 일부만 자주 변경
- 부분 업데이트가 빈번한 API
- 개별 레코드 수정이 배치 작업보다 많음

**성능 요구사항**:

- 네트워크 대역폭이 제한적
- 데이터베이스 부하가 높음
- 동시성이 중요한 시스템

### ❌ @DynamicUpdate를 피하는 경우

**테이블 특성**:

- 컬럼 수가 적은 단순한 테이블 (10개 미만)
- 모든 컬럼이 작은 사이즈
- 읽기 전용에 가까운 테이블

**업데이트 패턴**:

- 대부분의 필드를 함께 업데이트
- 배치 작업이 주된 업데이트 방식
- 같은 패턴의 업데이트가 매우 빈번

**시스템 특성**:

- PreparedStatement 캐싱 효과가 중요한 고성능 시스템
- 메모리가 제한적인 환경

## 결론

`@DynamicUpdate`는 PreparedStatement 캐싱 효율성 vs 불필요한 업데이트 방지 사이의 트레이드 오프를 개발자가 선택할 수 있게 해주는 도구다.

현대 ORM에서 기본적으로 PreparedStatement를 사용하는 이유가 캐싱을 통한 성능 최적화임을 이해하고, 자신의 애플리케이션 특성에 맞는 선택을 하는 것이 중요하다.
