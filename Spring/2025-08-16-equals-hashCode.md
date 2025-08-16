# JPA 환경에서 equals/hashCode 구현하기

## equals와 hashCode 기본 개념

### Object 클래스의 기본 동작

```java
// Object 클래스의 기본 equals
public boolean equals(Object obj) {
    return (this == obj);  // 참조값 비교
}
```

### equals 오버라이딩 5대 규칙

1. **반사적(Reflexive)**: `x.equals(x)`는 항상 true
2. **대칭적(Symmetric)**: `x.equals(y)`가 true면 `y.equals(x)`도 true
3. **이행적(Transitive)**: `x.equals(y)`와 `y.equals(z)`가 true면 `x.equals(z)`도 true
4. **일관적(Consistent)**: 객체가 변경되지 않았다면 몇 번을 호출해도 결과가 같아야 함
5. **null과의 비교**: `x.equals(null)`은 항상 false

### hashCode 규칙

- **equals-hashCode 계약**: `obj1.equals(obj2)`가 true면 `obj1.hashCode() == obj2.hashCode()`
- 역은 성립하지 않음 (해시 충돌 가능)

## JPA 환경에서의 특수한 문제들

### 1. 프록시 객체 문제

**문제 상황:**

```java
// 잘못된 구현
@Override
public boolean equals(Object obj) {
    if (getClass() != obj.getClass()) return false; // 문제!
    // ...
}

// 실제 실행 시
User realUser = entityManager.find(User.class, 1L);        // 실제 객체
User proxyUser = order.getUser();                         // 프록시 객체

System.out.println(realUser.getClass());
// 출력: class com.example.User

System.out.println(proxyUser.getClass());
// 출력: class com.example.User$HibernateProxy$Abc123

realUser.equals(proxyUser); // false! (같은 ID인데)
```

### 2. ID 기반 구현의 함정

**문제:**

```java
@Override
public boolean equals(Object obj) {
    User user = (User) obj;
    return Objects.equals(id, user.id); // ID가 null이면?
}

@Override
public int hashCode() {
    return Objects.hashCode(id); // ID가 null이면 0 반환
}
```

**실제 문제 상황:**

```java
Set<User> userSet = new HashSet<>();

User user1 = new User("john");  // id = null
User user2 = new User("jane");  // id = null
User user3 = new User("bob");   // id = null

userSet.add(user1); // hashCode = 0
userSet.add(user2); // hashCode = 0
userSet.add(user3); // hashCode = 0

System.out.println(userSet.size()); // 1 (모든 객체가 같다고 판단!)

// 영속화 후
entityManager.persist(user1); // id = 1, hashCode 변경!

userSet.contains(user1); // false! (HashSet에서 찾을 수 없음)
```

## 실제 비즈니스 문제 사례들

### 쇼핑몰 장바구니 중복 상품 문제

```java
// 잘못된 구현으로 인해
// 같은 상품이 장바구니에 중복으로 들어감
// iPhone 15 - 수량: 1개
// iPhone 15 - 수량: 2개  <- 중복!
```

### 주문 중복 처리 문제

```java
// 같은 주문번호로 여러 번 주문이 생성됨
// 결과: 고객 카드 2번 결제, 재고 2배 차감
```

### 금융 거래 중복 처리

```java
// processedAt 필드 변경으로 hashCode가 바뀌면서
// 같은 거래를 중복으로 처리
// 결과: 고객 계좌에서 돈이 중복 출금됨
```

## JPA Buddy 플러그인의 해결책

### 안전한 구현 패턴

```java
@Entity
public class User {
    @Id
    @GeneratedValue
    private Long id;

    @Column(unique = true)
    private String email; // Natural ID로 사용

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || Hibernate.getClass(this) != Hibernate.getClass(o))
            return false;

        User user = (User) o;
        return email != null && Objects.equals(email, user.getEmail());
    }

    @Override
    public int hashCode() {
        return Objects.hashCode(email);
    }
}
```

### 핵심 해결책들

1. **`Hibernate.getClass()` 사용** - 프록시 문제 해결
2. **Natural ID 기반 비교** - ID null 문제 해결
3. **null 체크 포함** - NPE 방지
4. **getter 사용** - 지연 로딩 고려

## 구현 전략별 비교

### 1. Natural ID 방식 (권장)

```java
// 비즈니스적으로 유일한 값 사용
return Objects.equals(email, user.email);
```

- ✅ 안정적이고 예측 가능
- ⚠️ Natural ID가 변경되면 문제 발생 가능

### 2. ID 기반 개선 방식

```java
@Override
public boolean equals(Object o) {
    // ... 프록시 체크 로직
    return id != null && Objects.equals(id, user.id);
}

@Override
public int hashCode() {
    return getClass().hashCode(); // 클래스 해시코드 사용
}
```

- ✅ ID 변경 없음 보장
- ⚠️ 새 엔티티 처리 주의 필요

### 3. UUID 사용 방식

```java
@Id
private String id = UUID.randomUUID().toString();
```

- ✅ 생성 즉시 유일한 ID 보장
- ⚠️ 성능상 오버헤드

## 주의사항과 모범 사례

### 1. 가변 필드를 Natural ID로 사용 금지

```java
// 위험한 예시
private String email; // 이메일이 변경될 수 있음

// 안전한 예시
private final String uuid = UUID.randomUUID().toString();
```

### 2. 컬렉션 사용 시 주의점

```java
Set<User> users = new HashSet<>();
User user = new User();
users.add(user);

// 객체 상태 변경 후
user.setEmail("new@email.com"); // hashCode 변경!
users.contains(user); // false! 찾을 수 없음
```

### 3. 상속 관계에서의 처리

```java
// instanceof vs getClass() 선택
if (obj instanceof Person) { ... }        // 하위 클래스도 허용
if (getClass() != obj.getClass()) { ... } // 정확히 같은 클래스만
```

## 실무 적용 가이드

### 1. JPA Buddy 플러그인 활용

- IntelliJ IDEA → Settings → Plugins → "JPA Buddy" 검색
- Alt+Insert → "equals() and hashCode()" 선택
- Business Key 옵션 권장

### 2. 팀 컨벤션 정하기

```java
// 예시 컨벤션
// 1. Natural ID 우선 사용
// 2. Natural ID 없으면 UUID 고려
// 3. ID 기반은 최후 수단
// 4. 항상 JPA Buddy로 생성
```

### 3. 테스트 코드 작성

```java
@Test
void equalsHashCodeContract() {
    User user1 = new User("test@email.com");
    User user2 = new User("test@email.com");

    // equals 계약 확인
    assertTrue(user1.equals(user2));
    assertTrue(user2.equals(user1));
    assertEquals(user1.hashCode(), user2.hashCode());

    // 컬렉션에서 정상 동작 확인
    Set<User> users = new HashSet<>();
    users.add(user1);
    assertTrue(users.contains(user2));
}
```

## 결론

JPA 환경에서의 equals/hashCode 구현은 단순해 보이지만 실제로는 매우 복잡하고 중요한 문제다. 잘못 구현하면 비즈니스 로직에서 심각한 오류가 발생할 수 있으며, 이는 고객 신뢰도 하락과 금전적 손실로 이어질 수 있다.

**핵심 원칙:**

1. JPA 환경에서는 일반적인 equals/hashCode 구현으로 부족하다
2. 프록시 객체, ID null 상태 등을 반드시 고려해야 한다
3. Natural ID를 활용한 구현이 가장 안전하다
4. JPA Buddy 같은 도구를 적극 활용하자
5. 충분한 테스트 코드로 검증하자
