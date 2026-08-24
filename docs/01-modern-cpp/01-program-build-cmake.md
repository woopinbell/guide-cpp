전체적으로 방향은 좋습니다. 다만 **CMake의 의존성 전파**, **헤더의 정의**, **copy elision**, **moved-from 객체**, **`std::move`**, **`noexcept` 이동** 부분은 현재 설명만으로는 오해할 여지가 있습니다. 아래처럼 보완하는 편이 좋습니다.

---

# Modern C++ 프로그램·빌드·CMake

## 목표

C++ 소스 코드가 실행 파일이 되는 과정을 구분하고, 라이브러리·애플리케이션·테스트를 각각 CMake target으로 구성합니다.

CMake 명령을 많이 외우는 것보다 다음 질문에 답할 수 있어야 합니다.

* 어느 `.cpp` 파일이 어느 target에 포함됩니까?
* 공개 헤더를 사용하는 데 필요한 include 경로는 어느 target이 제공합니까?
* C++ 표준과 compiler warning은 어느 target에 적용됩니까?
* 한 target이 사용하는 라이브러리는 누구에게까지 전달되어야 합니까?
* compile 오류와 link 오류는 어느 단계에서 발생하며 무엇을 확인해야 합니까?

## 번역 단위와 link

일반적인 C++ 빌드는 크게 다음 단계를 거칩니다.

```text
source
  ↓
preprocessing
  ↓
translation unit
  ↓
compilation
  ↓
object file
  ↓
linking
  ↓
executable / library
```

compiler는 일반적으로 프로젝트의 모든 `.cpp`를 하나의 덩어리로 처리하지 않습니다.

각 `.cpp`는 `#include`로 포함된 헤더의 내용과 합쳐져 하나의 **번역 단위(translation unit)** 가 됩니다.

```text
main.cpp + 포함된 headers
        ↓
translation unit
        ↓
main.o

job.cpp + 포함된 headers
        ↓
translation unit
        ↓
job.o

main.o + job.o
        ↓
      linker
        ↓
    executable
```

따라서 서로 다른 `.cpp`는 기본적으로 독립적으로 compile됩니다.

예를 들어 `main.cpp`에 다음 선언만 있어도 compile은 가능할 수 있습니다.

```cpp
void run_job();

int main() {
    run_job();
}
```

compiler는 `run_job()`의 선언을 알고 있으므로 호출 코드를 만들 수 있습니다.

하지만 어느 object file에도 실제 정의가 없다면 link 단계에서 실패합니다.

```text
undefined reference to `run_job()`
```

문제를 단계별로 구분합니다.

* **preprocessing/compile 오류**

  * 헤더를 찾지 못함
  * 문법 오류
  * 이름을 찾지 못함
  * 타입 불일치
  * overload 해석 실패
  * template instantiation 실패

* **link 오류**

  * 선언은 있지만 정의가 없음
  * 필요한 library가 link되지 않음
  * 동일한 비-inline 정의가 여러 번 존재함
  * 함수 선언과 실제 정의의 signature가 다름

* **runtime 실패**

  * 잘못된 입력
  * 잘못된 상태
  * dangling pointer/reference
  * 범위를 벗어난 접근
  * 환경이나 외부 자원 문제

* **test 실패**

  * 프로그램 자체는 실행되었지만 관찰된 결과가 테스트가 요구하는 결과와 다름

따라서 `undefined reference`가 발생했다면 헤더의 문법을 먼저 볼 문제가 아닙니다.

우선 다음을 확인합니다.

1. 해당 함수를 정의한 `.cpp`가 target에 포함되어 있는가?
2. 정의가 다른 library target에 있다면 그 target을 link했는가?
3. 선언과 정의의 namespace와 signature가 같은가?
4. 필요한 외부 library를 실제 link 단계에 전달했는가?

## 헤더와 source file의 역할

공개 헤더에는 **사용자가 코드를 compile하기 위해 알아야 할 선언**을 둡니다.

예:

```cpp
// include/task_store.hpp

#pragma once

#include <cstddef>
#include <string>

class TaskStore {
public:
    void add(std::string name);
    std::size_t size() const noexcept;
};
```

구현은 일반적으로 `.cpp`에 둡니다.

```cpp
// src/task_store.cpp

#include "task_store.hpp"

void TaskStore::add(std::string name) {
    // ...
}
```

구현에만 필요한 helper도 외부에서 사용할 이유가 없다면 `.cpp`에 숨기는 편이 좋습니다.

```cpp
namespace {

bool is_valid_name(const std::string& name) {
    return !name.empty();
}

}
```

### 헤더에 정의를 두면 항상 잘못인가?

그렇지는 않습니다.

문제는 **헤더 자체가 아니라 ODR(One Definition Rule)을 위반하는 정의**입니다.

예를 들어 다음 함수를 헤더에 그대로 정의하고 여러 `.cpp`에서 포함하면 문제가 될 수 있습니다.

```cpp
int calculate() {
    return 42;
}
```

각 번역 단위에 동일한 외부 linkage 함수 정의가 생기기 때문입니다.

반면 다음과 같은 정의는 헤더에 두는 것이 정상입니다.

* template 정의
* class 내부에서 정의한 member function
* `inline` 함수
* 적절한 `constexpr` 함수 및 변수
* header-only library의 구현

예:

```cpp
class Task {
public:
    int id() const noexcept {
        return id_;
    }

private:
    int id_{};
};
```

class 정의 안에서 정의한 member function은 암시적으로 `inline`이므로 여러 번역 단위에서 같은 정의가 나타날 수 있습니다.

### 공개 헤더의 include

헤더가 사용하는 타입은 가능하면 헤더 자신이 필요한 선언을 확보해야 합니다.

예를 들어 `std::size_t`와 `std::string`을 직접 사용한다면 해당 표준 헤더를 직접 포함하는 편이 안전합니다.

```cpp
#include <cstddef>
#include <string>
```

다른 헤더가 우연히 `<string>`을 포함하고 있을 것이라고 기대해서는 안 됩니다.

공개 헤더에서는 다음도 사용하지 않습니다.

```cpp
using namespace std;
```

헤더를 포함한 모든 번역 단위의 이름 검색에 영향을 주기 때문입니다.

## target을 기준으로 설정합니다

현대적인 CMake에서는 directory 전체에 설정을 뿌리는 것보다 **target이 무엇을 필요로 하는지 표현하는 방식**을 우선합니다.

```cmake
cmake_minimum_required(VERSION 3.20)

project(task_app LANGUAGES CXX)

add_library(task_core
    src/task.cpp
    src/task_store.cpp
)

target_include_directories(task_core
    PUBLIC
        include
)

target_compile_features(task_core
    PUBLIC
        cxx_std_20
)

target_compile_options(task_core
    PRIVATE
        -Wall
        -Wextra
        -Wpedantic
)

add_executable(task_app
    app/main.cpp
)

target_link_libraries(task_app
    PRIVATE
        task_core
)
```

여기서 중요한 것은 각 명령이 **target 자신의 요구사항**과 **target 사용자의 요구사항**을 구분한다는 점입니다.

## `PRIVATE`, `PUBLIC`, `INTERFACE`

세 키워드는 단순히 "보이느냐"를 표시하는 접근 제어자가 아닙니다.

**사용 요구사항(usage requirements)이 어느 target까지 전파되는가**를 결정합니다.

### `PRIVATE`

현재 target을 만들 때만 필요합니다.

```cmake
target_compile_options(task_core PRIVATE -Wall)
```

`task_core` 자체를 compile할 때는 `-Wall`을 사용하지만 `task_core`를 link하는 `task_app`에게 `-Wall`을 전달하지 않습니다.

### `PUBLIC`

현재 target도 필요하고 그 target을 사용하는 쪽도 필요합니다.

```cmake
target_include_directories(task_core PUBLIC include)
```

`task_core`의 `.cpp`에서도 다음을 사용할 수 있고,

```cpp
#include "task_store.hpp"
```

`task_core`를 사용하는 `task_app`에서도 같은 공개 헤더를 찾을 수 있어야 하므로 include 경로가 전파됩니다.

### `INTERFACE`

현재 target 자체를 compile하는 데에는 필요하지 않지만 사용자는 필요합니다.

대표적으로 header-only library에서 자주 사용합니다.

```cmake
add_library(task_utils INTERFACE)

target_include_directories(task_utils
    INTERFACE
        include
)

target_compile_features(task_utils
    INTERFACE
        cxx_std_20
)
```

`task_utils`에는 compile할 `.cpp`가 없으므로 자기 자신에게 compile 설정을 적용할 대상이 없습니다.

대신 `task_utils`를 사용하는 target에 요구사항을 전달합니다.

## 공개 의존성과 private 의존성

예를 들어 `task_core`의 구현에서만 어떤 외부 library를 사용한다고 합시다.

```cpp
// task_store.cpp
#include <some_library/detail.hpp>
```

하지만 공개 헤더에는 그 library가 나타나지 않습니다.

```cpp
// task_store.hpp
class TaskStore {
    // ...
};
```

이 경우 일반적으로 다음처럼 사용할 수 있습니다.

```cmake
target_link_libraries(task_core
    PRIVATE
        some_library
)
```

반대로 공개 API에 외부 library의 타입이 나타난다면 사용자가 그 의존성을 필요로 할 수 있습니다.

```cpp
#include <some_library/result.hpp>

some_library::result load();
```

이 경우 해당 library의 사용 요구사항도 사용자에게 전달되어야 하므로 보통 `PUBLIC`이 필요합니다.

```cmake
target_link_libraries(task_core
    PUBLIC
        some_library
)
```

즉 `PUBLIC` 여부는 단순히 "중요한 library인가?"로 결정하지 않습니다.

**이 target의 공개 API를 사용하는 코드를 compile/link하기 위해 사용자도 그 의존성을 알아야 하는가?**

를 기준으로 판단합니다.

## library와 실행 파일을 나눕니다

다음과 같은 `main.cpp`는 테스트하기 어렵습니다.

```cpp
int main(int argc, char** argv) {
    // argument 해석
    // 파일 읽기
    // 상태 변경
    // 핵심 계산
    // 결과 출력
    // 오류 처리
}
```

핵심 동작까지 `main()` 안에 있으면 테스트하려면 프로그램 process 전체를 실행해야 합니다.

대신 핵심 규칙을 library로 분리할 수 있습니다.

```text
task_core
├── 입력과 무관한 핵심 타입
├── 상태 변경 규칙
├── 계산
└── 검증

task_app
├── command-line argument 처리
├── 표준 입력/출력
├── 파일 위치 결정
└── 종료 상태 결정
```

테스트는 library를 직접 link합니다.

```cmake
add_executable(task_core_tests
    tests/task_core_tests.cpp
)

target_link_libraries(task_core_tests
    PRIVATE
        task_core
)

enable_testing()

add_test(
    NAME task.core
    COMMAND task_core_tests
)
```

이 경우 의존 관계는 다음과 같습니다.

```text
              task_core
              ↑       ↑
              │       │
        task_app    task_core_tests
```

애플리케이션과 테스트가 서로 의존하는 것이 아니라 둘 다 동일한 핵심 library를 사용합니다.

## 외부 의존성을 target으로 표현합니다

가능하면 compiler나 platform별 link flag를 직접 쓰기보다 CMake가 제공하는 target을 사용합니다.

예를 들어 thread를 사용한다면 다음과 같이 표현할 수 있습니다.

```cmake
find_package(Threads REQUIRED)

target_link_libraries(task_core
    PRIVATE
        Threads::Threads
)
```

그러면 CMake가 플랫폼에 필요한 compile/link option을 결정할 수 있습니다.

"thread를 사용하므로 무조건 `-pthread`를 직접 추가한다"보다 이 방식이 이식성이 좋습니다.

## out-of-source build

source와 build 결과를 같은 디렉터리에 섞지 않습니다.

```sh
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug
ctest --test-dir build/debug --output-on-failure
```

Release도 별도로 구성할 수 있습니다.

```sh
cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/release
ctest --test-dir build/release --output-on-failure
```

구조는 다음처럼 됩니다.

```text
project/
├── CMakeLists.txt
├── include/
├── src/
├── app/
├── tests/
└── build/
    ├── debug/
    └── release/
```

CMake는 build directory에 `CMakeCache.txt`를 저장합니다.

따라서 같은 build directory에서 option과 toolchain을 계속 바꾸면 이전 설정이 남아 원인을 혼동할 수 있습니다.

특히 다음을 변경할 때는 별도 build directory를 사용하는 편이 명확합니다.

* Debug / Release
* compiler
* sanitizer
* toolchain
* 중요한 feature option

단, Visual Studio, Xcode, Ninja Multi-Config 같은 **multi-config generator**에서는 `CMAKE_BUILD_TYPE` 대신 build 시 configuration을 선택하는 방식도 사용합니다.

예:

```sh
cmake --build build --config Debug
```

따라서 `CMAKE_BUILD_TYPE`이 모든 CMake 환경에서 동일하게 동작한다고 생각해서는 안 됩니다.

## `CMakePresets.json`

반복되는 configure 설정은 preset으로 기록할 수 있습니다.

예:

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "debug",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug"
      }
    }
  ]
}
```

그러면 다음처럼 실행할 수 있습니다.

```sh
cmake --preset debug
cmake --build build/debug
```

preset의 목적은 복잡성을 추가하는 것이 아니라 **반복해서 사용하는 configure 조건을 저장하는 것**입니다.

작은 학습 프로젝트라면 기본적인 target 구성과 build 과정을 먼저 이해한 뒤 도입해도 충분합니다.

## 자주 놓치는 문제

* `.cpp`를 만들었지만 `add_library()` 또는 `add_executable()`의 source에 추가하지 않습니다.
* 헤더는 찾을 수 있지만 실제 구현 library를 `target_link_libraries()`로 연결하지 않아 link에 실패합니다.
* 공개 헤더에 필요한 include 경로를 `PRIVATE`로 지정합니다.
* 공개 API에 나타나는 dependency를 `PRIVATE`로 숨겨 consumer가 compile되지 않습니다.
* 모든 target에 전역 warning flag를 적용하여 외부 dependency까지 같은 warning 정책을 적용합니다.
* `-Werror`를 dependency에까지 전달하여 compiler 버전이 바뀌자 외부 코드 때문에 build가 실패합니다.
* source directory 안에서 build하여 object file과 generated file을 source와 섞습니다.
* header가 자신이 사용하는 표준 타입의 헤더를 직접 include하지 않고 다른 헤더의 간접 include에 의존합니다.

## 프로젝트에서 확인할 질문

* executable이 실제로 사용하는 library target은 무엇입니까?
* test는 application 전체를 실행하지 않고 핵심 library를 직접 검사할 수 있습니까?
* 어떤 `.cpp`가 어떤 target에 속합니까?
* 공개 헤더의 include path는 어느 target이 전달합니까?
* 외부 dependency는 `PRIVATE`, `PUBLIC`, `INTERFACE` 중 무엇이어야 합니까?
* thread 같은 platform dependency를 어느 target이 link합니까?
* 공개 헤더만 다른 target에서 include해도 필요한 dependency가 자동으로 따라옵니까?
* 깨끗한 checkout에서 README에 적힌 명령만으로 같은 build를 만들 수 있습니까?

## 완료 기준

* preprocessing/compile, link, runtime, test 실패를 구분합니다.
* library, executable, test target을 직접 구성합니다.
* target의 source 목록이 실제 compile되는 `.cpp`를 결정한다는 것을 이해합니다.
* `PUBLIC`, `PRIVATE`, `INTERFACE`를 사용 요구사항 전파 기준으로 선택합니다.
* C++ 표준과 warning을 target 단위로 적용합니다.
* 공개 dependency와 구현 전용 dependency를 구분합니다.
* source와 build directory를 분리합니다.
* Debug와 Release 구성을 재현할 수 있습니다.