# C++98 프로그램과 타입 모델

## 목표

C++98 컴파일러가 실제로 허용하는 문법과 표준 라이브러리 범위 안에서 여러 파일로 구성된 프로그램을 작성합니다.

Modern C++ 코드를 단순히 옛 문법으로 치환하는 것이 목적은 아닙니다. C++11 이후에 추가된 언어 기능과 라이브러리 타입을 사용할 수 없다는 점을 먼저 확인하고, 그 기능이 담당하던 역할을 C++98에서 어떻게 표현할지 다시 결정해야 합니다.

예를 들어 `std::unique_ptr`을 raw pointer로 단순 치환하면 소유권 정보가 사라집니다. 따라서 누가 메모리를 해제하는지 클래스 설계와 함수 규칙으로 별도로 정해야 합니다.

## 표준 모드를 먼저 고정합니다

```sh
c++ -std=c++98 -Wall -Wextra -Werror -pedantic main.cpp
```

컴파일러 기본 모드에 맡기면 실수로 C++11 이후 기능을 사용할 수 있습니다. 로컬 빌드, Makefile, 테스트, CI에서 모두 같은 `-std=c++98` 기준을 사용합니다.

C++98에는 다음 기능이 없습니다.

* `auto` 타입 추론
* range-based for
* lambda
* `nullptr`
* scoped enum (`enum class`)
* rvalue reference와 이동 생성자
* `std::unique_ptr`
* `std::optional`
* `std::variant`
* `override`
* `final`
* `noexcept`

이 기능들을 단순히 제거하는 것이 아니라 원래 담당하던 역할을 다시 표현해야 합니다.

예를 들면 다음과 같습니다.

* lambda → 함수 객체 또는 일반 함수
* `nullptr` → null pointer constant인 `0`
* `unique_ptr` → raw pointer와 명시적인 단일 소유 규칙
* `optional<T>` → 별도의 성공 여부 값과 `T`, 또는 전용 결과 타입
* 이동 → 복사하거나, 복사를 금지하고 객체 수명을 다르게 설계

## 번역 단위와 헤더

각 `.cpp` 파일은 독립적인 **번역 단위(translation unit)** 로 컴파일됩니다.

예를 들어 다음과 같이 빌드한다면,

```sh
c++ -std=c++98 -c Counter.cpp
c++ -std=c++98 -c main.cpp
c++ Counter.o main.o -o app
```

`Counter.cpp`와 `main.cpp`는 각각 별도로 컴파일되고 마지막에 linker가 두 object file을 연결합니다.

일반적으로 헤더에는 다음 내용을 둡니다.

* 클래스 선언
* 함수 선언
* 필요한 타입 선언
* template 정의

일반적인 비-template 함수 정의는 `.cpp` 파일에 둡니다.

```cpp
#ifndef STORE_HPP
#define STORE_HPP

#include <string>

class Store {
public:
    bool get(const std::string &key, std::string &value) const;
};

#endif
```

C++98에서 `#pragma once`를 지원하는 컴파일러는 많지만 표준 기능은 아닙니다. 컴파일러 독립성을 우선하면 include guard를 사용합니다.

또한 헤더는 자신이 사용하는 타입에 필요한 표준 헤더를 직접 포함해야 합니다.

예를 들어 `std::string`을 사용한다면 해당 헤더 자체가 `<string>`을 포함해야 합니다.

다른 파일이 우연히 먼저 `<string>`을 포함해 주는 상황에 의존하면 include 순서를 바꿨을 때 컴파일이 깨질 수 있습니다.

## 선언과 정의

```cpp
// Counter.hpp

class Counter {
public:
    Counter();

    void increment();
    int value() const;

private:
    int value_;
};
```

```cpp
// Counter.cpp

#include "Counter.hpp"

Counter::Counter()
    : value_(0) {
}

void Counter::increment() {
    ++value_;
}

int Counter::value() const {
    return value_;
}
```

선언과 정의의 함수 타입은 정확히 일치해야 합니다.

다음 두 함수는 서로 같은 함수가 아닙니다.

```cpp
int value();
int value() const;
```

뒤의 `const`는 반환값에 붙는 것이 아니라 **멤버 함수 자체의 타입 일부**입니다.

따라서 헤더에는

```cpp
int value() const;
```

라고 선언하고 `.cpp`에는

```cpp
int Counter::value() {
    return value_;
}
```

라고 정의하면 선언된 함수의 정의가 존재하지 않습니다. 이런 문제는 컴파일 단계가 아니라 link 단계에서 발견되기도 합니다.

## 기본 타입과 정수 변환

C++의 기본 정수 타입 크기를 무조건 특정 비트 수라고 가정하지 않습니다.

예를 들어 외부 문자열을 `int`로 읽고 싶다면 먼저 충분히 넓은 타입으로 변환한 뒤 실제 `int` 범위를 검사할 수 있습니다.

```cpp
#include <cerrno>
#include <climits>
#include <cstdlib>

char *end = 0;
errno = 0;

const long parsed = std::strtol(text, &end, 10);

if (errno != 0
    || end == text
    || *end != '\0'
    || parsed < INT_MIN
    || parsed > INT_MAX) {
    throw ParseError("invalid integer");
}

const int value = static_cast<int>(parsed);
```

각 검사는 서로 다른 오류를 확인합니다.

* `errno != 0`: 표현 가능한 `long` 범위를 벗어났는지 확인
* `end == text`: 숫자를 하나도 읽지 못했는지 확인
* `*end != '\0'`: 숫자 뒤에 처리하지 않은 문자가 남았는지 확인
* `INT_MIN`, `INT_MAX`: 결과가 실제 `int` 범위 안에 있는지 확인

`static_cast<int>` 자체가 범위를 검사해 주는 것은 아닙니다. **범위를 먼저 검증한 뒤 cast해야 합니다.**

## C++ cast

C 스타일 cast는 한 문법으로 여러 종류의 변환을 시도하기 때문에 어떤 변환을 의도했는지 코드만 보고 판단하기 어렵습니다.

C++에서는 목적에 맞는 cast를 사용합니다.

* `static_cast`: 관련 있는 타입 사이의 일반적인 명시적 변환
* `dynamic_cast`: 다형 클래스 계층에서 실행 시점의 실제 객체 타입 확인
* `const_cast`: `const` 또는 `volatile` 한정자 변경
* `reinterpret_cast`: 서로 직접적인 타입 관계가 없는 pointer, reference 또는 일부 정수/pointer 변환

예를 들어 수치 변환은 다음과 같이 작성할 수 있습니다.

```cpp
long value = 10;
int narrowed = static_cast<int>(value);
```

그러나 `static_cast`는 값이 `int` 범위에 들어오는지 확인하지 않습니다.

`dynamic_cast`는 아무 클래스에나 사용할 수 있는 것도 아닙니다. 실행 시점 타입 검사가 필요한 대상은 일반적으로 최소 하나의 virtual 함수를 가진 **다형(polymorphic) 클래스**여야 합니다.

`reinterpret_cast`는 메모리 내용을 안전하게 변환해 주는 기능이 아닙니다. 결과의 의미와 유효성은 변환 대상과 플랫폼에 크게 의존하므로 일반적인 타입 변환 수단으로 사용하지 않습니다.

## `const`

```cpp
const std::string &name() const;
```

여기에는 서로 다른 두 개의 `const`가 있습니다.

앞의 `const`:

```cpp
const std::string &
```

반환된 참조를 통해 문자열을 변경할 수 없다는 뜻입니다.

예를 들어 다음 코드는 허용되지 않습니다.

```cpp
object.name() += "x";
```

뒤의 `const`:

```cpp
name() const
```

이 멤버 함수가 일반 멤버를 변경하지 않는다는 뜻입니다.

따라서 `const` 객체에서도 호출할 수 있습니다.

```cpp
const User user;
user.name();
```

정확히 말하면 `const` 멤버 함수가 객체의 모든 상태를 절대 변경할 수 없다는 뜻은 아닙니다. `mutable` 멤버 등은 변경할 수 있습니다.

따라서 보통은 **외부에서 관찰되는 논리적인 값을 변경하지 않는 함수**라는 의미로 사용합니다.

C++98에는 `auto` 타입 추론이 없으므로 iterator 타입을 직접 작성해야 합니다.

```cpp
std::map<std::string, int>::const_iterator found =
    values.find(key);
```

반복해서 사용한다면 `typedef`로 줄이는 것도 일반적입니다.

```cpp
typedef std::map<std::string, int> ValueMap;

ValueMap::const_iterator found = values.find(key);
```

## enum과 이름 범위

C++98의 일반 `enum`은 C++11의 `enum class`처럼 독립적인 이름 범위와 강한 타입 검사를 제공하지 않습니다.

다음처럼 클래스나 구조체 내부에 enum을 넣으면 이름 충돌을 줄일 수 있습니다.

```cpp
struct Status {
    enum Value {
        Queued,
        Running,
        Done
    };
};
```

사용할 때는 다음과 같이 표현할 수 있습니다.

```cpp
Status::Value status = Status::Queued;
```

C++98 enum 값은 정수로 암묵 변환될 수 있습니다.

```cpp
int raw = Status::Running;
```

반대로 명시적인 cast를 사용하면 enum에 실제로 정의하지 않은 정수 값도 만들 수 있습니다.

```cpp
Status::Value status = static_cast<Status::Value>(999);
```

따라서 파일, 네트워크, 사용자 입력처럼 외부에서 들어온 정수 값을 enum으로 사용할 때는 먼저 허용되는 값인지 검증해야 합니다.

## `0`과 null pointer

C++98에는 `nullptr`가 없습니다.

pointer가 어떤 객체도 가리키지 않는다는 값을 표현할 때 일반적으로 `0`을 사용합니다.

```cpp
Handler *handler = 0;
```

여기서 `0`은 pointer 타입이 아닙니다.

**정수 리터럴 `0`이 null pointer constant로 특별 취급되어 pointer로 변환되는 것**입니다.

이 차이는 overload에서 중요합니다.

```cpp
void process(int);
void process(Handler *);
```

다음 호출은 pointer overload가 아니라 `int` overload를 선택합니다.

```cpp
process(0);
```

또한 C++98의 `NULL`도 구현에 따라 보통 `0` 또는 비슷한 정수 상수로 정의되므로 `nullptr`처럼 pointer 전용 타입이라고 생각해서는 안 됩니다.

## 이름과 scope

긴 함수에서 같은 변수 이름을 계속 재사용하면 어느 객체가 살아 있는지 확인하기 어려워집니다.

필요한 범위만 별도 block으로 제한할 수 있습니다.

```cpp
{
    std::map<Key, Value *>::iterator found = data.find(key);

    if (found != data.end())
        return found->second;
}
```

block을 벗어나면 `found`는 더 이상 존재하지 않습니다.

특히 C++에서는 scope 종료가 객체의 소멸 시점과 연결되므로 변수 범위를 좁히는 것은 단순한 이름 정리 이상의 의미가 있습니다.

## build 문제를 구분합니다

프로그램을 만들 때 발생하는 문제를 단계별로 나누면 원인을 찾기 쉽습니다.

### Compile 오류

각 번역 단위를 object file로 만들지 못한 경우입니다.

예:

* C++98에서 지원하지 않는 문법
* 타입 불일치
* 필요한 선언 누락
* 잘못된 함수 호출
* 잘못된 template 사용

### Link 오류

각 `.cpp`는 컴파일됐지만 최종 실행 파일을 만들지 못한 경우입니다.

예:

* 필요한 `.cpp` 파일을 link하지 않음
* 선언만 있고 정의가 없음
* 선언과 정의의 signature가 다름
* template 정의를 사용할 수 있는 위치에 두지 않음

### 실행 중 오류

실행 파일은 만들어졌지만 프로그램 동작 중 문제가 발생한 경우입니다.

예:

* 이미 해제된 메모리 접근
* dangling pointer
* 잘못된 입력 처리
* system call 실패를 무시함
* file descriptor 누수
* 잘못된 배열 범위 접근

## 자주 놓치는 문제

* 프로그램 build는 C++98인데 테스트만 컴파일러 기본 모드로 실행합니다.
* 헤더에 `using namespace std;`를 둡니다.
* `long` 결과를 범위 확인 없이 `int`로 변환합니다.
* `NULL`을 `nullptr`와 같은 pointer 전용 타입으로 생각합니다.
* 멤버 함수 선언과 정의에서 뒤의 `const`가 다릅니다.
* template 정의를 `.cpp`에 두고 다른 번역 단위에서 사용하려 합니다.
* 다른 헤더의 간접 include에 의존합니다.
* cast가 값의 유효성이나 객체 수명을 검사해 준다고 생각합니다.

## 완료 기준

* 모든 build와 test에서 C++98 표준 모드를 사용합니다.
* 번역 단위, 선언, 정의, link의 관계를 설명할 수 있습니다.
* C++11 이후 기능을 단순 제거하지 않고 C++98에서 필요한 동작을 다시 설계할 수 있습니다.
* 정수 변환 전에 문자열 전체와 대상 타입 범위를 검사합니다.
* `const`, enum, null pointer의 C++98 특성을 설명하고 코드에서 처리합니다.
* compile 오류와 link 오류를 구분할 수 있습니다.