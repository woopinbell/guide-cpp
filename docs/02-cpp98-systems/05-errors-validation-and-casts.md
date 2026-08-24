# C++98 오류 처리·입력 검증·캐스트

## 목표

외부 문자열을 내부 값으로 변환하기 전에 다음을 확인합니다.

* 입력 형식이 올바른가
* 필요한 token을 모두 읽었는가
* 숫자와 enum 값이 표현 가능한 범위에 있는가
* 현재 프로그램 상태에서 해당 연산을 수행할 수 있는가

또한 요청 처리 중 실패하더라도 이미 정상적으로 존재하던 상태가 일부만 변경된 채 남지 않도록 합니다.

C++98에는 `std::optional`, `std::expected` 같은 결과 타입이 없으므로 상황에 따라 다음을 선택합니다.

* 반환값
* 출력 매개변수
* enum이나 별도 결과 타입
* 예외

모든 실패를 예외로 처리하거나 모든 실패를 `bool` 하나로 표현하는 것이 목적은 아닙니다. 호출자가 실패 원인을 알아야 하는지, 해당 위치에서 복구할 수 있는지를 기준으로 선택합니다.

## 입력 처리 순서

외부 입력을 처리하는 일반적인 순서는 다음과 같습니다.

```text
문자열 분리
→ token 수와 형식 확인
→ 숫자·enum 변환
→ 값 범위 확인
→ 현재 상태와 충돌하는지 확인
→ 상태 변경
```

핵심은 **검증 중에는 가능한 한 프로그램 상태를 변경하지 않는 것**입니다.

잘못된 예:

```text
PUT 명령 확인
→ map에 key 삽입
→ 나머지 인자 확인
→ 잘못된 인자 발견
```

이렇게 하면 요청은 실패했는데 map에는 일부 변경이 남을 수 있습니다.

보통 다음 형태가 더 안전합니다.

```text
요청 전체 검증
→ 실행에 필요한 값 준비
→ 실패 가능한 준비 작업 완료
→ 실제 상태 변경
```

process 생성, 파일 생성, socket 연결처럼 외부 효과가 있는 작업도 같은 관점으로 봅니다.

## 문법 오류와 상태 오류를 구분합니다

다음 두 입력 실패는 성격이 다릅니다.

```text
PUT key
```

```text
PUT existing-key value
```

첫 번째는 명령 형식 자체가 잘못되었습니다.

두 번째는 문법은 올바르지만 현재 저장소 상태 때문에 실행할 수 없습니다.

따라서 처리 단계를 다음처럼 구분할 수 있습니다.

```text
문법 검증
→ 내부 값 생성
→ 상태 조건 검증
→ 실행
```

이 구분은 외부 응답을 다음처럼 다르게 만들 때도 유용합니다.

```text
BAD_REQUEST
CONFLICT
STORE_FULL
```

## 숫자 변환

`atoi()`는 입력 검증용으로 적합하지 않습니다.

예:

```cpp
int value = std::atoi(text);
```

결과가 `0`이면 다음을 구분하기 어렵습니다.

```text
"0"
"abc"
""
```

또한 표현 범위를 벗어난 입력을 안정적으로 처리하기 어렵습니다.

C++98에서는 `strtol()`과 end pointer를 사용하면 변환 결과와 입력 소비 여부를 따로 검사할 수 있습니다.

```cpp
#include <cerrno>
#include <climits>
#include <cstdlib>

int parseInt(const char *text) {
    char *end = 0;

    errno = 0;

    const long value = std::strtol(text, &end, 10);

    if (errno == ERANGE
        || end == text
        || *end != '\0'
        || value < INT_MIN
        || value > INT_MAX) {
        throw ParseError("invalid integer");
    }

    return static_cast<int>(value);
}
```

각 조건은 서로 다른 문제를 검사합니다.

* `errno == ERANGE`: 입력 값이 `long`으로 표현 가능한 범위를 벗어남
* `end == text`: 숫자를 하나도 읽지 못함
* `*end != '\0'`: 숫자 뒤에 처리되지 않은 문자가 남음
* `value < INT_MIN || value > INT_MAX`: `long`에는 들어가지만 `int`에는 들어가지 않음

예를 들어:

```text
"42"   → 성공
"42x"  → 실패
"x42"  → 실패
""     → 실패
```

## `strtol()`의 공백 처리를 명확히 정합니다

`strtol()`은 선행 공백을 자동으로 건너뜁니다.

따라서 다음 입력은 별도 검사가 없다면 숫자로 읽힐 수 있습니다.

```text
"   42"
```

반면 뒤쪽 공백은 end pointer 검사 때문에 다음 코드에서는 실패합니다.

```text
"42   "
```

즉 단순히 `strtol()`을 사용한다고 해서 원하는 입력 문법이 자동으로 정해지는 것은 아닙니다.

다음 중 무엇을 허용할지 먼저 결정해야 합니다.

```text
"42"
" 42"
"42 "
" 42 "
"+42"
"-42"
```

예를 들어 공백을 전혀 허용하지 않는 protocol이라면 변환 전에 공백을 명시적으로 거부해야 합니다.

반대로 parser 단계에서 tokenization이 이미 끝났고 token 안에는 공백이 들어올 수 없다고 보장한다면 `parseInt()`에서는 그 검사를 반복하지 않아도 됩니다.

## 입력 전체 소비를 확인합니다

다음 코드는 부족합니다.

```cpp
long value = std::strtol(text, 0, 10);
```

입력의 앞부분만 숫자로 읽어도 성공 여부를 확인할 방법이 없습니다.

예:

```text
42abc
```

`strtol()`은 `42`까지 읽을 수 있습니다.

따라서 end pointer를 사용해 반드시 원하는 범위 전체가 소비되었는지 검사합니다.

```cpp
char *end = 0;
long value = std::strtol(text, &end, 10);

if (end == text || *end != '\0')
    throw ParseError("invalid integer");
```

## 변환 성공과 대상 타입 범위는 별도 문제입니다

다음 입력이 `long`에는 들어가지만 `int`에는 들어가지 않을 수 있습니다.

따라서:

```cpp
const long value = std::strtol(text, &end, 10);
```

가 성공했다고 해서 다음 변환이 자동으로 안전해지는 것은 아닙니다.

```cpp
int result = static_cast<int>(value);
```

먼저 검사해야 합니다.

```cpp
if (value < INT_MIN || value > INT_MAX)
    throw ParseError("integer out of range");

return static_cast<int>(value);
```

cast는 범위 검사를 대신하지 않습니다.

## enum 변환도 같은 방식으로 검사합니다

외부 입력에서 enum을 만든다고 가정합니다.

```cpp
struct Status {
    enum Value {
        Queued,
        Running,
        Done
    };
};
```

다음처럼 외부 정수를 곧바로 cast하면:

```cpp
Status::Value status =
    static_cast<Status::Value>(raw);
```

`raw == 999` 같은 값도 만들 수 있습니다.

따라서 먼저 허용 범위를 확인합니다.

```cpp
Status::Value parseStatus(int raw) {
    switch (raw) {
    case 0:
        return Status::Queued;
    case 1:
        return Status::Running;
    case 2:
        return Status::Done;
    default:
        throw ParseError("invalid status");
    }
}
```

외부 문자열이라면 문자열을 직접 비교하는 방법도 가능합니다.

```cpp
if (text == "queued")
    return Status::Queued;
```

핵심은 cast가 유효한 enum 값을 검증해 주지 않는다는 것입니다.

## 실패를 타입으로 구분합니다

서로 다른 종류의 오류를 호출자가 다르게 처리해야 한다면 예외 타입도 나눌 수 있습니다.

```cpp
class ParseError : public std::runtime_error {
public:
    explicit ParseError(const std::string &message)
        : std::runtime_error(message) {
    }
};

class ConflictError : public std::runtime_error {
public:
    explicit ConflictError(const std::string &message)
        : std::runtime_error(message) {
    }
};

class StoreFullError : public std::runtime_error {
public:
    explicit StoreFullError(const std::string &message)
        : std::runtime_error(message) {
    }
};
```

최상위 요청 처리 코드는 이를 외부 protocol 값으로 바꿀 수 있습니다.

```cpp
try {
    processRequest(line);
}
catch (const ParseError &) {
    std::cout << "BAD_REQUEST\n";
}
catch (const ConflictError &) {
    std::cout << "CONFLICT\n";
}
catch (const StoreFullError &) {
    std::cout << "STORE_FULL\n";
}
```

이렇게 하면 내부 오류 표현과 외부 protocol을 분리할 수 있습니다.

## 예외 메시지를 protocol로 사용하지 않습니다

다음과 같은 방식은 피하는 편이 좋습니다.

```cpp
catch (const std::exception &error) {
    std::cout << error.what() << '\n';
}
```

`what()`은 내부 진단용 설명일 수 있습니다.

예:

```text
failed to open /var/lib/app/store.db
duplicate map node at Store.cpp:91
```

이를 그대로 외부 응답으로 사용하면 다음 문제가 생깁니다.

* 내부 경로나 구현 정보가 노출될 수 있음
* 오류 문구 수정이 protocol 변경이 됨
* 테스트가 사람이 읽는 문자열 표현에 지나치게 의존함

외부 protocol에는 안정적인 코드나 정해진 응답을 사용합니다.

```text
BAD_REQUEST
CONFLICT
INTERNAL_ERROR
```

내부 로그에는 별도의 상세 메시지를 남길 수 있습니다.

## 모든 실패를 세분화해야 하는 것은 아닙니다

호출자가 실패 이유를 구분할 필요가 없다면 하나의 결과로 충분할 수 있습니다.

예:

```cpp
bool Store::contains(const std::string &key) const;
```

반대로 다음 여러 상태를 구분해야 한다면 `bool`은 부족합니다.

```text
성공
존재하지 않음
권한 없음
저장소 오류
```

이런 경우 C++98에서는 enum과 구조체를 사용할 수 있습니다.

```cpp
struct LookupResult {
    enum Code {
        Found,
        NotFound,
        Failed
    };

    Code code;
    std::string value;
};
```

## 반환값과 출력 매개변수

C++98에는 `std::optional`이 없습니다.

값이 존재하지 않을 수 있는 간단한 조회는 다음처럼 표현할 수 있습니다.

```cpp
bool Store::get(
    const std::string &key,
    std::string &value
) const;
```

규칙:

```text
return true
→ value에 조회 결과가 들어 있음

return false
→ value의 내용은 사용하지 않음
```

이 규칙은 명확하게 문서화해야 합니다.

가능하다면 실패할 때 출력 매개변수를 변경하지 않도록 구현하면 사용하기 더 쉽습니다.

```cpp
bool Store::get(
    const std::string &key,
    std::string &value
) const {
    DataMap::const_iterator found = data_.find(key);

    if (found == data_.end())
        return false;

    value = found->second;
    return true;
}
```

다만 마지막 대입 자체도 `std::string` 메모리 할당 때문에 예외를 던질 수 있다는 점은 별도 문제입니다.

## 반환값과 예외는 서로 다른 용도에 적합할 수 있습니다

예를 들어 key가 없는 상황이 정상적인 조회 결과라면:

```cpp
bool get(...);
```

처럼 반환값으로 표현할 수 있습니다.

반면 내부 메모리 할당 실패나 프로그램이 정상적으로 처리하기 어려운 상황은 예외로 전파하는 편이 자연스러울 수 있습니다.

즉 다음과 같이 구분할 수 있습니다.

```text
예상 가능한 정상적 부재
→ 반환값

요청 자체가 잘못됨
→ ParseError

현재 상태와 충돌
→ ConflictError

하위 연산의 예외적 실패
→ 예외 전파
```

절대적인 규칙은 아니지만 실패 성격을 구분하는 데 도움이 됩니다.

## 상태 변경 전 검증

예를 들어 값을 직접 저장하는 map이라면:

```cpp
void Store::putNew(
    const std::string &key,
    const std::string &value
) {
    if (data_.find(key) != data_.end())
        throw ConflictError("key already exists");

    if (data_.size() >= capacity_)
        throw StoreFullError("store is full");

    TextBuffer candidate(value.c_str());

    data_.insert(
        std::make_pair(key, candidate)
    );
}
```

순서는 다음과 같습니다.

```text
중복 검사
→ capacity 검사
→ 새 값 생성
→ map 삽입
```

`TextBuffer` 생성이 실패하면 아직 map은 변경되지 않았습니다.

또한 표준 `map::insert()`가 실패해 예외를 던지는 경우에도 성공하지 않은 삽입 때문에 새 요소가 남아서는 안 됩니다.

다만 자신의 `TextBuffer` 복사 생성자나 comparator 등이 container 요구사항을 올바르게 지킨다는 전제가 필요합니다.

## pointer를 저장한다면 예외 안전성이 달라집니다

다음 container를 사용한다고 가정합니다.

```cpp
std::map<std::string, TextBuffer *> data_;
```

다음 코드는 위험합니다.

```cpp
TextBuffer *owned =
    new TextBuffer(value.c_str());

data_.insert(
    std::make_pair(key, owned)
);
```

`new`는 성공했지만 `insert()`가 node 할당 중 예외를 던지면 `owned`는 map이 소유하지 않습니다.

그대로 함수가 끝나면 누수됩니다.

따라서 소유권 규칙과 예외 경로를 따로 처리해야 합니다.

```cpp
TextBuffer *owned =
    new TextBuffer(value.c_str());

try {
    std::pair<DataMap::iterator, bool> result =
        data_.insert(
            std::make_pair(key, owned)
        );

    if (!result.second) {
        delete owned;
        throw ConflictError("key already exists");
    }
}
catch (...) {
    // 이미 insert 성공 후 다른 코드가 예외를 던지는 구조가 아니라는
    // 전제가 필요합니다.
    // 실제 구현에서는 소유권 이전 시점을 더 명확하게 구성합니다.
    throw;
}
```

이처럼 raw pointer container는 값 container보다 소유권 실패 경로를 훨씬 세밀하게 설계해야 합니다.

중복을 미리 검사했다고 해도 `insert()` 자체의 메모리 할당 실패 가능성은 남습니다.

## 중복 사전 검사와 실제 삽입 결과를 구분합니다

다음 코드는 단일 thread에서 같은 map을 자신만 수정한다면 충분할 수 있습니다.

```cpp
if (data_.find(key) != data_.end())
    throw ConflictError(...);

data_.insert(...);
```

하지만 더 일반적으로는 `insert()`가 반환하는 성공 여부 자체를 확인하는 방법도 있습니다.

```cpp
std::pair<DataMap::iterator, bool> result =
    data_.insert(std::make_pair(key, value));

if (!result.second)
    throw ConflictError("key already exists");
```

다만 이 경우 `value` 준비 과정과 insert 실패 시 자원 정리를 함께 고려해야 합니다.

특히 raw owning pointer를 값으로 넣는 경우에는 단순히 `bool`만 확인하는 것으로 충분하지 않습니다.

## 여러 상태를 바꾼다면 원자적으로 보이게 만듭니다

예를 들어 객체가 다음 세 값을 함께 유지한다고 가정합니다.

```text
name
buffer
size
```

이 중 두 개만 바뀐 뒤 세 번째 작업이 실패하면 객체 invariant가 깨질 수 있습니다.

가능하다면 새 상태를 먼저 완성합니다.

```text
새 상태 후보 생성
→ 모든 검증 완료
→ 기존 객체와 swap
```

예:

```cpp
Record candidate(newName, newValue);
swap(candidate);
```

그러면 후보 생성이 실패한 동안 현재 객체에는 손대지 않습니다.

이것이 항상 가능한 것은 아니지만, 상태를 먼저 파괴하고 새 값을 만드는 것보다 실패 복구가 쉽습니다.

## 예외가 지나가는 범위

모든 함수가 모든 예외를 잡을 필요는 없습니다.

예외는 **그 위치에서 의미 있게 복구하거나 다른 결과로 변환할 수 있을 때** 잡는 것이 좋습니다.

예:

```text
RequestParser
→ ParseError 발생 가능

Store
→ ConflictError / StoreFullError 발생 가능

Handler
→ 특별히 복구할 방법이 없으면 그대로 전파

main 또는 요청 경계
→ 외부 응답 코드로 변환
```

중간 함수에서 다음처럼 하는 것은 대개 좋지 않습니다.

```cpp
try {
    store.putNew(key, value);
}
catch (...) {
}
```

오류를 숨긴 채 이후 코드를 계속 실행하기 때문입니다.

## catch한 뒤 계속 실행할 수 있는지 확인합니다

예외를 잡았다고 해서 프로그램 상태가 자동으로 정상인 것은 아닙니다.

다음 두 경우는 다릅니다.

### 강한 보장을 제공하는 함수

```text
실패
→ 호출 전 상태 그대로
```

이 경우 catch한 뒤 다른 요청을 계속 처리하기 쉽습니다.

### 일부 변경을 남길 수 있는 함수

```text
A 변경
→ B 변경 실패
→ 예외
```

이 경우 catch한 뒤 객체를 그대로 계속 사용해도 되는지 판단해야 합니다.

따라서 예외 처리만큼 중요한 것이 각 함수의 **실패 후 상태**입니다.

## 소멸자와 stack unwinding

예외가 발생하면 이미 완전히 생성된 지역 객체는 scope를 빠져나가면서 소멸합니다.

```cpp
void process() {
    TextBuffer buffer(...);
    Request request(...);

    dangerousOperation();
}
```

`dangerousOperation()`이 예외를 던지면 이미 생성된 `request`, `buffer`는 역순으로 소멸합니다.

따라서 자원을 객체 소멸자에서 정리하도록 만든 RAII 타입을 사용하면 중간 함수마다 cleanup 코드를 반복할 필요가 줄어듭니다.

raw pointer나 file descriptor 자체는 자동으로 자원을 해제하지 않으므로 별도의 소유 객체가 필요합니다.

## `static_cast`

`static_cast`는 compile 시 관계가 알려진 일반적인 명시적 변환에 사용합니다.

예:

```cpp
long value = 42;

if (value < INT_MIN || value > INT_MAX)
    throw ParseError("out of range");

int narrowed = static_cast<int>(value);
```

중요한 점은:

```text
static_cast
≠
범위 검사
```

라는 것입니다.

### upcast

파생 pointer를 기반 pointer로 바꾸는 upcast도 가능합니다.

```cpp
Derived *derived = ...;
Base *base = static_cast<Base *>(derived);
```

하지만 정상적인 public inheritance에서는 보통 명시적 cast조차 필요하지 않습니다.

```cpp
Base *base = derived;
```

가 더 자연스럽습니다.

따라서 "`static_cast`는 upcast에 사용한다"보다 **upcast도 표현할 수 있지만 일반적인 public upcast는 암묵 변환으로 충분하다**고 이해하는 편이 정확합니다.

### downcast

`static_cast`로 기반 pointer를 파생 pointer로 내릴 수도 있지만 실제 객체 타입을 runtime에 확인하지 않습니다.

```cpp
Derived *derived =
    static_cast<Derived *>(base);
```

`base`가 실제로 `Derived` 객체를 가리킨다는 보장이 없다면 매우 위험합니다.

runtime 확인이 필요하면 `dynamic_cast`를 사용합니다.

## `dynamic_cast`

다형 클래스 계층에서 실제 객체 타입을 runtime에 확인할 때 사용할 수 있습니다.

```cpp
Base *base = getObject();

Derived *derived =
    dynamic_cast<Derived *>(base);

if (derived == 0)
    return false;
```

pointer 대상 cast에 실패하면 `0`을 반환합니다.

reference라면:

```cpp
Derived &derived =
    dynamic_cast<Derived &>(base);
```

실패 시 `std::bad_cast` 예외가 발생합니다.

### 다형 타입이 필요합니다

runtime downcast를 하려는 기반 타입은 일반적으로 최소 하나의 virtual 함수를 가진 다형 타입이어야 합니다.

예:

```cpp
class Base {
public:
    virtual ~Base() {
    }
};
```

`dynamic_cast`는 임의의 관련 없는 타입을 안전하게 바꾸는 기능이 아닙니다.

## `const_cast`

`const_cast`는 `const`나 `volatile` 한정자를 변경할 때 사용합니다.

예:

```cpp
const char *text = ...;

char *mutableText =
    const_cast<char *>(text);
```

하지만 cast가 실제 객체를 mutable하게 만들어 주는 것은 아닙니다.

원래 객체 자체가 const라면:

```cpp
const int value = 42;

int *pointer =
    const_cast<int *>(&value);

*pointer = 10;
```

이 변경은 undefined behavior입니다.

반대로 원래 객체는 mutable인데 const view를 통해 접근하고 있던 경우라면 const를 제거하는 것 자체가 항상 곧 undefined behavior는 아닙니다.

```cpp
int value = 42;
const int *view = &value;

int *pointer =
    const_cast<int *>(view);

*pointer = 10;
```

여기서는 실제 객체 `value`가 원래 const가 아닙니다.

그래도 일반 코드에서는 const를 제거해야 하는 설계 자체를 먼저 의심하는 편이 좋습니다.

### 오래된 C API와 사용할 때

어떤 C API가 실제로 문자열을 수정하지 않지만 signature만 다음처럼 되어 있을 수 있습니다.

```cpp
void legacy(char *text);
```

이때:

```cpp
legacy(
    const_cast<char *>(text)
);
```

를 사용하려면 해당 함수가 정말 메모리를 수정하지 않는다는 외부 보장이 필요합니다.

함수가 수정한다면 원래 객체가 const인 경우 문제가 발생합니다.

## `reinterpret_cast`

`reinterpret_cast`는 타입 간 의미 관계를 보존하는 일반 변환이 아닙니다.

낮은 수준의 표현을 다른 pointer 타입으로 전달해야 하는 코드에서 사용될 수 있습니다.

예를 들어 socket API에서:

```cpp
sockaddr_in address;
```

를 `sockaddr *` 형태의 C API에 전달할 때 특정 구현 패턴에서 cast가 등장할 수 있습니다.

하지만 `reinterpret_cast`가 다음을 확인해 주지는 않습니다.

* 해당 주소에 실제로 그 타입의 객체가 존재하는가
* 올바르게 정렬되어 있는가
* 해당 객체의 수명이 시작되어 있는가
* 그 타입을 통해 접근하는 것이 언어 규칙상 허용되는가
* 실제 메모리 배치가 기대와 같은가

즉:

```text
reinterpret_cast
=
컴파일러에게 저수준 변환을 명시

reinterpret_cast
≠
실제 객체 타입 변경
```

입니다.

## byte buffer와 객체는 같은 것이 아닙니다

예를 들어:

```cpp
char buffer[sizeof(Header)];
```

가 있다고 해서 다음 cast만으로 `Header` 객체가 정상적으로 존재하게 되는 것은 아닙니다.

```cpp
Header *header =
    reinterpret_cast<Header *>(buffer);
```

alignment, 객체 수명, 메모리 표현 등의 조건을 따로 만족해야 합니다.

단순한 network byte stream을 struct pointer로 cast해서 바로 읽는 코드는 padding, alignment, byte order 문제도 생길 수 있습니다.

대부분의 protocol parsing에서는 필요한 byte를 읽어서 각 field를 명시적으로 변환하는 편이 더 안전합니다.

## 네 가지 C++ cast를 비교합니다

| cast               | 주 용도                 | 해주지 않는 것               |
| ------------------ | -------------------- | ---------------------- |
| `static_cast`      | 일반적인 명시적 변환          | 범위 검사, runtime 타입 확인   |
| `dynamic_cast`     | 다형 객체의 runtime 타입 확인 | 임의 타입 간 변환             |
| `const_cast`       | cv 한정자 변경            | 실제 const 객체의 안전한 변경    |
| `reinterpret_cast` | 저수준 표현 변환            | 객체 타입·정렬·수명·메모리 안전성 보장 |

cast를 사용했다는 사실 자체가 안전성을 의미하지 않습니다.

## signed integer overflow

signed integer overflow는 C++에서 undefined behavior입니다.

예:

```cpp
int value = INT_MAX;
++value;
```

이 결과가 항상 `INT_MIN`으로 돌아간다고 가정해서는 안 됩니다.

### 덧셈은 계산 전에 검사합니다

양수 방향:

```cpp
if (right > 0 && left > INT_MAX - right)
    throw ArithmeticError("overflow");
```

음수 방향도 필요합니다.

```cpp
if (right < 0 && left < INT_MIN - right)
    throw ArithmeticError("overflow");
```

전체적으로:

```cpp
if ((right > 0 && left > INT_MAX - right)
    || (right < 0 && left < INT_MIN - right)) {
    throw ArithmeticError("overflow");
}

const int result = left + right;
```

검사 자체가 overflow를 일으키지 않도록 식을 구성해야 합니다.

## "더 넓은 타입에서 계산"도 항상 해결책은 아닙니다

예를 들어:

```cpp
long result =
    static_cast<long>(left)
    + static_cast<long>(right);
```

를 사용할 수는 있습니다.

하지만 이것이 안전하려면 `long`이 실제로 계산에 필요한 범위를 충분히 표현할 수 있어야 합니다.

C++98에서 다음을 무조건 가정하면 안 됩니다.

```text
long은 언제나 int보다 훨씬 넓다
```

표준이 보장하는 것은 최소 범위 관계이지 특정한 32비트/64비트 배치가 아닙니다.

따라서 portable code에서는 대상 타입 범위와 사용 플랫폼의 보장을 분명하게 확인해야 합니다.

## 곱셈 overflow는 더 복잡합니다

다음은 잘못된 검사입니다.

```cpp
if (left * right > INT_MAX)
```

검사를 하기 전에 `left * right` 자체가 이미 overflow할 수 있기 때문입니다.

곱셈은 부호 조합과 `INT_MIN` 같은 경계를 고려해서 **곱하기 전에** 검사해야 합니다.

가능하다면 입력 범위를 먼저 제한하는 것도 좋은 방법입니다.

예:

```text
입력 허용 범위: -10000 ~ 10000
```

처럼 application 자체가 좁은 범위를 요구한다면 그 조건을 먼저 검사하여 연산을 단순하게 만들 수 있습니다.

## partial update

다음 함수가 있다고 가정합니다.

```cpp
void update(Account &account) {
    account.name = newName;
    account.balance = parseBalance(text);
}
```

`name` 변경은 성공했지만 `parseBalance()`가 실패하면 객체에는 일부 변경만 남습니다.

대신:

```cpp
std::string candidateName = newName;
int candidateBalance = parseBalance(text);

account.set(
    candidateName,
    candidateBalance
);
```

처럼 실패 가능한 준비 작업을 먼저 끝낼 수 있습니다.

또는 전체 객체를 후보로 만들 수 있습니다.

```cpp
Account candidate(account);

candidate.setName(newName);
candidate.setBalance(parsedBalance);

account.swap(candidate);
```

핵심은 모든 변경을 반드시 transaction으로 구현하라는 것이 아니라, **실패 시 어느 상태가 남는지 의도적으로 정하라는 것**입니다.

## 자주 놓치는 문제

* `atoi()` 결과 `0`을 정상적인 `"0"`과 변환 실패 모두로 사용합니다.
* `strtol()`이 선행 공백을 자동으로 소비한다는 사실을 놓칩니다.
* `"42x"`의 앞부분만 읽고 성공으로 처리합니다.
* `strtol()` 성공만 확인하고 대상 `int` 범위를 검사하지 않습니다.
* enum을 정수 cast만으로 검증했다고 생각합니다.
* map을 먼저 수정하고 나중에 중복이나 capacity를 검사합니다.
* raw pointer 생성 후 container 삽입 실패 경로에서 메모리를 잃습니다.
* 모든 예외를 `catch (...)`로 잡고 정상 응답을 반환합니다.
* 예외를 잡으면 객체 상태가 자동으로 정상이라고 생각합니다.
* 범위 검사 없이 `static_cast<int>`를 사용합니다.
* `dynamic_cast`가 아무 클래스에나 runtime 검사를 제공한다고 생각합니다.
* `const_cast`가 실제 const 객체를 안전하게 mutable로 만든다고 생각합니다.
* `reinterpret_cast`가 실제 객체 타입을 변경한다고 생각합니다.
* overflow 검사식 자체에서 overflow를 발생시킵니다.
* 더 넓은 타입이 항상 충분히 넓다고 가정합니다.

## 완료 기준

* 숫자 입력에서 실제로 문자를 하나 이상 소비했는지 확인합니다.
* 원하는 입력 범위 전체가 소비되었는지 확인합니다.
* 선행·후행 공백과 부호 허용 규칙을 명확히 정합니다.
* `long` 변환 성공과 `int` 범위 검사를 별도로 수행합니다.
* 문법 오류, 상태 충돌, 용량 부족과 내부 실패를 구분합니다.
* 정상적인 "값 없음"과 예외적인 실패를 서로 다른 방법으로 표현할 수 있습니다.
* 실패 가능한 준비 작업을 실제 상태 변경 전에 수행합니다.
* raw pointer 생성 성공 후 container 삽입 실패 경로를 처리합니다.
* 예외가 발생했을 때 객체가 어떤 상태로 남는지 설명할 수 있습니다.
* 네 가지 C++ cast가 무엇을 검사하고 무엇을 검사하지 않는지 설명할 수 있습니다.
* signed overflow를 계산 전에 검사합니다.
* partial update가 발생하는 테스트를 만들고 실패 후 상태를 확인할 수 있습니다.