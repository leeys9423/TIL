# Call by Value vs Call by Reference

## Call by Value란?

**메서드 호출 시 값의 복사본을 전달하는 방식**

- 원본이 아닌 복사된 값으로 작업
- 메서드 내부에서 파라미터를 수정해도 원본에 영향 없음
- 자바는 모든 경우에 call by value를 사용

## Call by Reference란?

**메서드 호출 시 변수 자체의 주소를 전달하는 방식**

- 원본 변수의 메모리 주소를 직접 전달
- 메서드 내부에서 원본 변수를 직접 수정 가능
- C++의 `&` 참조자를 통해 사용

## 자바 예시 - 객체 전달 (Call by Value)

```java
public class Main {
    public static void main(String[] args) {
        Person p = new Person();
        p.name = "철수";

        System.out.println("호출 전: " + p.name);  // 철수
        System.out.println("호출 전 참조값: " + p);  // Person@15db9742

        changeObject(p);

        System.out.println("호출 후: " + p.name);  // 철수 (안 바뀜!)
        System.out.println("호출 후 참조값: " + p);  // Person@15db9742 (똑같음)
    }

    public static void changeObject(Person p) {
        System.out.println("메서드 안 참조값: " + p);  // Person@15db9742

        p = new Person();  // 새 객체 생성
        p.name = "영희";

        System.out.println("새 객체 참조값: " + p);  // Person@6d06d69c (다름!)
        System.out.println("메서드 안 이름: " + p.name);  // 영희
    }
}

class Person {
    String name;
}
```

### 실행 결과

```
호출 전: 철수
호출 전 참조값: Person@15db9742
메서드 안 참조값: Person@15db9742
새 객체 참조값: Person@6d06d69c
메서드 안 이름: 영희
호출 후: 철수
호출 후 참조값: Person@15db9742
```

### 메모리 구조

```
[main 메서드]
p ----> 0x15db9742 { name: "철수" }
    |
    | 참조값 0x15db9742가 복사되어 전달
    v
[changeObject 메서드]
p(복사본) ----> 0x15db9742 { name: "철수" }
           |
           | p = new Person() 실행
           v
p(복사본) ----> 0x6d06d69c { name: "영희" }

메서드 종료 후:
[main 메서드]
p ----> 여전히 0x15db9742 { name: "철수" }
```

## C++ 예시 - Call by Reference

```cpp
#include <iostream>
using namespace std;

class Person {
public:
    string name;
};

void changeObject(Person &p) {  // & = call by reference
    cout << "메서드 안 주소: " << &p << endl;

    Person newPerson;
    newPerson.name = "영희";

    p = newPerson;  // 원본 자체가 바뀜!

    cout << "바꾼 후 주소: " << &p << endl;
    cout << "메서드 안 이름: " << p.name << endl;
}

int main() {
    Person p;
    p.name = "철수";

    cout << "호출 전: " << p.name << endl;
    cout << "호출 전 주소: " << &p << endl;

    changeObject(p);

    cout << "호출 후: " << p.name << endl;  // 영희로 바뀜!
    cout << "호출 후 주소: " << &p << endl;

    return 0;
}
```

### 실행 결과

```
호출 전: 철수
호출 전 주소: 0x7ffeefbff5a0
메서드 안 주소: 0x7ffeefbff5a0  (같은 주소!)
바꾼 후 주소: 0x7ffeefbff5a0  (여전히 같음!)
메서드 안 이름: 영희
호출 후: 영희  (원본이 바뀜!)
호출 후 주소: 0x7ffeefbff5a0
```

### 메모리 구조

```
[main 함수]
p (주소: 0x7ffeefbff5a0) { name: "철수" }
    |
    | 변수 p의 주소를 전달 (별명 생성)
    v
[changeObject 함수]
p는 원본의 별명(alias)
같은 주소 0x7ffeefbff5a0를 공유
    |
    | p = newPerson 실행
    v
원본 주소 0x7ffeefbff5a0의 내용이 바뀜
{ name: "영희" }

함수 종료 후:
[main 함수]
p (주소: 0x7ffeefbff5a0) { name: "영희" }  ← 원본이 바뀜!
```

## 핵심 차이점

| 구분                | Call by Value (자바)                 | Call by Reference (C++)        |
| ------------------- | ------------------------------------ | ------------------------------ |
| 전달 방식           | 값(또는 참조값)의 **복사본**         | 변수 **자체의 주소**           |
| 변수 교체 가능 여부 | ❌ `p = new Person()` → 원본 안 바뀜 | ✅ `p = newPerson` → 원본 바뀜 |
| 메모리 주소         | 복사본과 원본이 **다른 변수**        | **같은 변수**를 공유           |
| 독립성              | 메서드 내부와 외부가 독립적          | 메서드 내부에서 원본 직접 조작 |

## 결론

- **자바는 항상 call by value**

  - 기본 타입: 값 자체가 복사됨
  - 객체 타입: 참조값(주소값)이 복사됨
  - 메서드 안의 파라미터는 항상 **새로운 변수**

- **C++는 call by reference 지원**

  - `&`를 사용하면 원본 변수의 별명을 만듦
  - 메서드 안의 파라미터는 원본과 **같은 변수**

- **헷갈리는 이유**
  - 자바에서 객체 내용은 수정할 수 있어서 call by reference처럼 보임
  - 하지만 변수가 가리키는 대상(참조 자체)은 바꿀 수 없음
  - 이것이 call by value의 핵심 특징!

## 재귀 호출과의 연관성

재귀 호출에서도 각 호출마다 파라미터의 복사본이 새로 만들어지기 때문에,
각 재귀 단계가 서로의 값을 간섭하지 않고 독립적으로 작업할 수 있다.
