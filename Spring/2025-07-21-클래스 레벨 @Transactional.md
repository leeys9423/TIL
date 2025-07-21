# 클래스 레벨 @Transactional, 정말 편할까? - 트랜잭션 경계 설정에 대한 고민

이 글을 쓰게 된 이유는, 흔히 Service Layer에서 클래스 레벨에 `@Transactional` 애노테이션을 선언하여 사용하곤 합니다.

## 예시

```java
@Transactional  // 클래스 레벨에 선언
@Service
public class UserService {

    public User getUser(Long id) {
        // 단순 조회인데도 쓰기 트랜잭션이 생성됨
        return userRepository.findById(id).orElse(null);
    }

    @Transactional(readOnly = true)  // 조회 메서드마다 이걸 붙여야 함
    public List<User> searchUsers(String keyword) {
        return userRepository.findByNameContaining(keyword);
    }

    public void createUser(User user) {
        // 이 메서드를 위해 클래스 레벨에 @Transactional을 선언
        userRepository.save(user);
    }
}
```

클래스 레벨에 선언하게 될 시에, 간단한 조회 메서드에도 `@Transactional`이 적용되게 됩니다.

## @Transactional 동작 원리

### 1. AOP 프록시 생성

Spring은 `@Transactional`이 붙은 클래스를 **프록시 객체**로 감쌉니다.

```java
// 원본 서비스
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
    }
}

// Spring이 실제로 생성하는 프록시 (개념적 표현)
public class UserService$$SpringProxy extends UserService {
    private UserService target;
    private PlatformTransactionManager transactionManager;

    @Override
    public void createUser(User user) {
        // 1. 트랜잭션 시작
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            // 2. 실제 메서드 실행
            target.createUser(user);

            // 3. 커밋
            transactionManager.commit(status);
        } catch (Exception e) {
            // 4. 롤백
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

### 2. 메서드 실행 시 상세 과정

```java
@Transactional
public void createUser(User user) {
    userRepository.save(user);  // 이 한 줄을 위해 아래 과정이 모두 실행됨
}
```

실제 내부에서 일어나는 일

```
1. 프록시 메서드 호출 시작
   ↓
2. TransactionInterceptor.invoke() 실행
   ↓
3. 트랜잭션 속성 확인 (@Transactional 설정값들)
   ↓
4. 기존 트랜잭션 존재 여부 확인 (Propagation 처리)
   ↓
5. 새로운 트랜잭션 시작
   - Connection Pool에서 Connection 획득
   - autoCommit = false 설정
   - 트랜잭션 컨텍스트에 Connection 바인딩
   ↓
6. 실제 메서드 실행 (target.createUser(user))
   ↓
7. 메서드 정상 종료 시:
   - connection.commit() 실행
   - Connection을 Pool로 반환
   - 트랜잭션 컨텍스트 정리
   ↓
8. 예외 발생 시:
   - 롤백 대상 예외인지 확인 (기본: RuntimeException, Error)
   - connection.rollback() 실행
   - Connection을 Pool로 반환
   - 예외 재던짐
```

### 3. 단순 조회 메서드에서 불필요한 과정

```
1. 쓰기 트랜잭션 시작 준비
2. Connection Pool에서 Connection 획득
3. autoCommit = false 설정 (불필요!)
4. 트랜잭션 매니저에서 커밋 준비 작업
5. 실제 SELECT 쿼리 실행
6. commit() 호출 (실제로는 아무것도 커밋할 게 없음)
7. Connection 반환
```

### 4. readOnly = true일 때의 차이

```
1. Connection.setReadOnly(true) 설정
2. 일부 DB에서 읽기 전용 최적화 적용
3. Hibernate에서 flush 모드를 MANUAL로 설정 (더티 체킹 비활성화)
4. 실제 커밋 과정 생략 (DB에 따라)
```

### 5. 트랜잭션 없이 실행될 때

```
1. Repository 호출
2. JPA에서 Connection Pool에서 Connection 획득
3. autoCommit = true 상태로 SELECT 실행
4. 즉시 Connection 반환
```

## 내용 정리

- 프록시 방식: 모든 @Transactional 메서드는 프록시를 통해 실행
- 무거운 준비 과정: 단순 조회도 트랜잭션 시작/종료 과정을 모두 거침
- Connection 점유 시간: 트랜잭션이 있으면 메서드 실행 동안 계속 Connection 점유
- 불필요한 오버헤드: 조회만 하는데도 커밋/롤백 준비 작업 수행

## 마무리

귀찮지만 클래스 레벨에서 `@Transactional` 애노테이션을 적용하지 말고, 코드가 길어질 수도 있겠지만 메서드 레벨에 알맞은 트랜잭션 설정을 하여 사용하는 것이 성능적으로나 에러 예방적으로나 좋다고 생각합니다!
