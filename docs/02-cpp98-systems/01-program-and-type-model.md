# C++98 프로그램과 타입 모델

## 목표

C++98 compiler가 실제로 허용하는 문법과 표준 라이브러리 범위 안에서 여러 파일 프로그램을 구성합니다. Modern C++ 코드를 문법만 바꿔 복사하지 않고, 사용할 수 없는 기능을 확인한 뒤 타입과 소유 방식을 다시 정합니다.

## 표준 모드를 먼저 고정합니다

```sh
c++ -std=c++98 -Wall -Wextra -Werror -pedantic main.cpp
```

compiler 기본 모드에 맡기면 실수로 C++11 이후 기능을 사용할 수 있습니다. 기준 build 명령과 CI에서 같은 `-std=c++98`을 사용합니다.

C++98에는 다음 기능이 없습니다.

- `auto` 타입 추론
- range-for
- lambda
- `nullptr`
- scoped enum
- 이동 생성자와 rvalue reference
- `unique_ptr`, `optional`, `variant`
- `override`, `final`, `noexcept`

필요한 동작을 별도 클래스, 함수 객체, 상태 값과 명시적인 복사 규칙으로 표현합니다.

## 번역 단위와 헤더

각 `.cpp`는 별도로 컴파일됩니다. 헤더에는 선언과 template 정의를 두고 일반 함수 정의는 `.cpp`에 둡니다.

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

C++98에서는 `#pragma once`를 지원하는 compiler가 많지만 표준 기능은 아닙니다. 이식성을 우선하면 include guard를 사용합니다.

헤더는 필요한 표준 헤더를 직접 포함합니다. 다른 헤더가 우연히 포함한 선언에 기대지 않습니다.

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

Counter::Counter() : value_(0) {}
void Counter::increment() { ++value_; }
int Counter::value() const { return value_; }
```

선언과 정의의 signature가 정확히 같아야 합니다. `const` 누락도 다른 함수가 됩니다.

## 기본 타입과 변환

정수 크기를 가정하지 않습니다. 입력을 `long`으로 읽은 뒤 `INT_MIN`과 `INT_MAX`를 확인하고 `int`로 변환하는 식으로 범위를 검사합니다.

```cpp
char *end = 0;
errno = 0;
const long parsed = std::strtol(text, &end, 10);
if (errno != 0 || end == text || *end != '\0'
    || parsed < INT_MIN || parsed > INT_MAX) {
    throw ParseError("invalid integer");
}
```

C 스타일 cast는 여러 변환을 한 문법으로 수행해 의도를 숨깁니다. 다음 C++ cast를 사용합니다.

- `static_cast`: 수치 변환과 명시적인 일반 변환
- `dynamic_cast`: 다형 기반 타입에서 실제 파생 타입 확인
- `const_cast`: const 속성 변경
- `reinterpret_cast`: 비트 수준 또는 관련 없는 pointer 변환

cast가 범위나 수명을 검사해 주지는 않습니다.

## `const`

```cpp
const std::string &name() const;
```

첫 `const`는 반환한 문자열을 이 참조로 변경하지 못한다는 뜻이고, 뒤의 `const`는 호출 중 객체의 논리적 값을 바꾸지 않는 멤버 함수라는 뜻입니다.

C++98에서는 `const_iterator`를 명시적으로 사용합니다.

```cpp
std::map<std::string, int>::const_iterator found = values.find(key);
```

## enum과 이름 범위

C++98 enum 값은 바깥 범위로 노출되고 정수로 암묵 변환될 수 있습니다.

```cpp
struct Status {
    enum Value {
        Queued,
        Running,
        Done
    };
};
```

위처럼 타입 내부에 넣어 이름 충돌을 줄일 수 있습니다. 그래도 잘못된 정수 값을 cast해 넣을 수 있으므로 외부 입력은 별도 함수에서 검증합니다.

## `0`과 null pointer

C++98에는 `nullptr`가 없습니다. pointer의 빈 값은 `0`을 사용합니다.

```cpp
Handler *handler = 0;
```

정수 overload와 pointer overload가 함께 있는 API에서는 `0`이 모호할 수 있습니다. overload 설계를 단순하게 유지합니다.

## 이름과 scope

긴 함수에서 같은 이름을 재사용하지 않습니다. iterator 범위가 끝나면 다음 변수를 선언하도록 scope를 좁힙니다.

```cpp
{
    std::map<Key, Value>::iterator found = data.find(key);
    if (found != data.end())
        return found->second;
}
```

## build 문제를 구분합니다

- compile 오류: C++98에서 지원하지 않는 문법, 타입 불일치, 누락된 선언
- link 오류: `.cpp` 누락, signature 불일치, template 정의 위치 오류
- 실행 오류: 수명, 입력, 시스템 호출 처리 문제

## 자주 놓치는 문제

- build는 C++98인데 test만 compiler 기본 모드로 실행합니다.
- 헤더에 `using namespace std;`를 둡니다.
- `long` 입력을 범위 확인 없이 `int`로 cast합니다.
- `NULL`이 항상 pointer 전용 타입이라고 생각합니다.
- 멤버 함수 선언과 정의에서 `const`가 다릅니다.
- template 정의를 `.cpp`에 두고 다른 번역 단위에서 사용합니다.

## 완료 기준

- C++98 build flag와 warning을 고정합니다.
- 헤더와 `.cpp`의 선언·정의 관계를 설명합니다.
- Modern C++ 기능을 C++98에서 사용할 수 있는 방식으로 다시 설계합니다.
- 정수 변환 전에 입력 전체와 범위를 검사합니다.
- `const`, enum, null pointer의 C++98 특성을 코드에서 처리합니다.
