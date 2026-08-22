# Modern C++ 프로그램·빌드·CMake

## 목표

C++ source가 실행 파일이 되는 과정을 구분하고, library·application·test를 각각 CMake target으로 만듭니다. CMake 명령을 많이 외우는 것보다 다음 질문에 답하는 것이 중요합니다.

- 어느 `.cpp`가 어느 target에 포함됩니까?
- 누가 공개 헤더의 include 경로를 제공합니까?
- C++ 표준과 compiler warning은 어느 target에 적용됩니까?
- compile 오류와 link 오류를 어떻게 구분합니까?

## 번역 단위와 link

compiler는 프로젝트 전체를 한 번에 읽지 않습니다. 각 `.cpp`와 그 파일이 포함한 헤더를 별도의 번역 단위로 처리합니다.

```text
main.cpp + headers       → main.o
job.cpp + headers        → job.o

main.o + job.o           → linker → executable
```

- compile 오류는 현재 번역 단위의 문법, 이름, 타입 문제입니다.
- link 오류는 선언한 함수 정의가 없거나 같은 정의가 여러 번 들어온 경우가 많습니다.
- 실행 실패는 입력, 상태, 수명 또는 실행 환경을 확인합니다.
- test 실패는 관찰한 결과가 요구사항과 다르다는 뜻입니다.

`undefined reference`가 나오면 헤더 문법보다 target의 source 목록과 link 대상을 먼저 봅니다.

## 헤더에 둘 내용

공개 헤더에는 호출자가 알아야 할 선언을 둡니다. 구현에만 필요한 helper와 private 자료는 가능하면 `.cpp`에 둡니다.

```cpp
// include/task_store.hpp
#pragma once

#include <string>

class TaskStore {
public:
    void add(std::string name);
    std::size_t size() const noexcept;
};
```

헤더에 함수 정의나 전역 변수를 잘못 두면 여러 번 정의될 수 있습니다. template과 `inline` 함수처럼 정의가 호출 위치에 필요한 경우는 예외입니다.

공개 헤더에서는 `using namespace`를 쓰지 않습니다. 그 헤더를 포함한 모든 파일의 이름 검색 결과가 달라집니다.

## target을 기준으로 설정합니다

```cmake
cmake_minimum_required(VERSION 3.20)
project(task_app LANGUAGES CXX)

add_library(task_core
    src/task.cpp
    src/task_store.cpp
)

target_include_directories(task_core PUBLIC include)
target_compile_features(task_core PUBLIC cxx_std_20)
target_compile_options(task_core PRIVATE -Wall -Wextra -Wpedantic)

add_executable(task_app app/main.cpp)
target_link_libraries(task_app PRIVATE task_core)
```

`PUBLIC`, `PRIVATE`, `INTERFACE`는 단순한 공개 범위 표기가 아닙니다.

- `PRIVATE`: 현재 target을 빌드할 때만 필요합니다.
- `PUBLIC`: 현재 target과 이를 사용하는 target 모두 필요합니다.
- `INTERFACE`: 현재 target 자체에는 필요 없고 사용자에게만 전달합니다.

예를 들어 공개 헤더가 `<string>`만 사용한다면 별도 include 경로를 전달할 필요가 없습니다. 반면 공개 헤더가 외부 library header를 포함한다면 그 의존성도 사용자에게 전달해야 합니다.

## library와 실행 파일을 나눕니다

`main.cpp`에 상태 변경과 파일 처리를 모두 넣으면 테스트가 process 실행에만 의존합니다. 핵심 코드를 library로 두면 함수와 타입을 직접 검사할 수 있습니다.

```cmake
add_executable(task_core_tests tests/task_core_tests.cpp)
target_link_libraries(task_core_tests PRIVATE task_core)

enable_testing()
add_test(NAME task.core COMMAND task_core_tests)
```

application target은 command-line argument, 표준 입출력, 종료 상태를 처리하고 실제 규칙은 library에 맡깁니다.

## out-of-source build

source와 build 결과를 같은 디렉터리에 섞지 않습니다.

```sh
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug
ctest --test-dir build/debug --output-on-failure
```

Debug와 Release는 별도 build 디렉터리를 사용합니다. 한 디렉터리에서 option을 반복해서 바꾸면 cache 때문에 현재 설정을 착각하기 쉽습니다.

반복되는 설정은 `CMakePresets.json`에 기록할 수 있습니다. 다만 작은 프로젝트를 시작하기 위해 package manager나 복잡한 preset부터 도입할 필요는 없습니다.

## 자주 놓치는 문제

- source를 만들고 `add_library()` 또는 `add_executable()`에 추가하지 않습니다.
- `target_link_libraries()` 없이 header include만 성공해 link 단계에서 실패합니다.
- 모든 target에 전역 warning flag를 강제해 외부 코드까지 `-Werror`로 빌드합니다.
- 공개 header가 필요한 include 경로를 `PRIVATE`로 둡니다.
- source 디렉터리에서 직접 build해 생성 파일을 commit합니다.

## 프로젝트에서 확인할 질문

- executable이 실제로 사용하는 library target은 무엇입니까?
- test가 application이 아니라 library를 직접 검사할 수 있습니까?
- thread나 filesystem 같은 의존성은 누가 link합니까?
- 공개 header를 다른 디렉터리에서 include해도 빌드됩니까?
- 새 checkout에서 README 명령만으로 같은 결과를 만들 수 있습니까?

## 완료 기준

- compile, link, runtime, test 실패를 구분합니다.
- library, executable, test target을 직접 구성합니다.
- `PUBLIC`, `PRIVATE`, `INTERFACE`를 사용 이유와 함께 선택합니다.
- C++ 표준과 warning을 target 단위로 적용합니다.
- 별도 build 디렉터리에서 Debug와 Release를 재현합니다.
