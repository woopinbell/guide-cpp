# 값·수명·복사·이동

## 목표

코드를 읽을 때 단순히 "pointer인가 값인가"만 확인하지 않습니다.

다음을 추적해야 합니다.

* 객체는 언제 생성됩니까?
* 객체의 수명은 언제 끝납니까?
* 어떤 객체가 자원을 소유합니까?
* reference, pointer, view는 어느 객체를 빌려 보고 있습니까?
* 복사하면 자원이 어떻게 복제됩니까?
* 이동하면 자원이 어느 객체로 넘어갑니까?
* 원본 객체보다 observer가 오래 살아남을 가능성이 있습니까?

## 객체와 객체의 수명

`stack`과 `heap`은 흔히 저장 위치를 설명할 때 사용하는 표현이지만, 그것만으로 C++ 객체의 수명을 판단해서는 안 됩니다.

예를 들어 다음 지역 객체는 함수 호출 중 존재합니다.

```cpp
std::string make_name() {
    std::string name{"worker"};
    return name;
}
```

`name`이라는 지역 변수의 수명은 함수가 끝날 때 종료됩니다.

하지만 `return name;`이 반환하는 값이 그와 함께 사라지는 것은 아닙니다.

개념적으로는 호출자가 사용할 결과 객체가 생성되며, 실제 구현에서는 compiler가 **copy elision**, 특히 NRVO(Named Return Value Optimization)를 적용하여 지역 객체를 반환 결과가 놓일 위치에 직접 생성할 수도 있습니다.

즉 다음처럼 생각해서는 안 됩니다.

```text
지역 객체가 사라지므로 반환된 문자열도 사라진다
```

값을 반환하는 것과 지역 객체의 reference를 반환하는 것은 완전히 다릅니다.

잘못된 예:

```cpp
const std::string& make_name() {
    std::string name{"worker"};
    return name; // dangling reference
}
```

함수가 끝나면 `name`이 파괴되므로 반환한 reference는 더 이상 유효한 객체를 가리키지 않습니다.

## 동적 할당과 수명

동적 저장 영역에 있다는 사실도 객체가 계속 살아 있다는 의미는 아닙니다.

```cpp
auto owner = std::make_unique<std::string>("worker");

std::string* observer = owner.get();

owner.reset();
```

`reset()`이 실행되면 `unique_ptr`이 소유하던 `std::string` 객체가 파괴됩니다.

하지만 `observer` 변수에는 이전 주소 값이 그대로 남을 수 있습니다.

```cpp
observer != nullptr
```

이라고 해서 그 주소에 살아 있는 객체가 존재한다는 뜻은 아닙니다.

이 pointer는 이제 **dangling pointer**입니다.

```cpp
*observer // undefined behavior
```

핵심은 다음 두 사실을 구분하는 것입니다.

```text
pointer가 주소 값을 가지고 있다
≠
그 주소에 해당 객체가 살아 있다
```

## 초기화와 대입

다음 세 코드는 비슷해 보이지만 다른 연산입니다.

```cpp
Task first{"compile"};
```

새로운 `Task` 객체를 생성하고 초기화합니다.

```cpp
Task second = first;
```

새로운 `Task` 객체를 만들면서 `first`의 값을 사용하므로 **복사 생성(copy construction)** 입니다.

```cpp
second = first;
```

`second`는 이미 존재합니다.

기존 객체의 상태를 다른 값으로 바꾸므로 **복사 대입(copy assignment)** 입니다.

차이는 중요합니다.

생성자는 아직 기존 상태가 없는 객체를 만듭니다.

대입 연산자는 이미 유효한 상태와 자원을 가진 객체를 수정해야 합니다.

예를 들어 객체가 메모리를 소유한다면 대입 과정은 다음 문제를 처리해야 할 수 있습니다.

```text
기존 자원
   ↓
새 자원 확보 시도
   ↓
성공?
 ├─ yes → 기존 자원 교체
 └─ no  → 기존 상태를 어떻게 할 것인가?
```

표준 library 타입을 조합해서 만든 값 타입에서는 이런 처리가 이미 구현되어 있으므로 직접 자원 관리 코드를 작성할 필요가 크게 줄어듭니다.

## 값, reference, pointer, view

### 값

```cpp
void consume(Task task);
```

함수 내부에는 독립적인 `Task` 객체가 존재합니다.

호출 방법에 따라 그 객체는 복사되거나 이동되어 만들어질 수 있습니다.

```cpp
Task task;

consume(task);            // 보통 복사
consume(std::move(task)); // 이동 가능
consume(Task{});          // 임시 값 전달
```

따라서 "값 parameter는 항상 복사한다"도 정확하지 않습니다.

### `const T&`

```cpp
void inspect(const Task& task);
```

다른 객체를 빌려 읽습니다.

함수가 해당 객체를 소유하지 않으며 이 reference를 호출 이후까지 저장한다면 별도의 수명 조건을 검토해야 합니다.

### `T&`

```cpp
void update(Task& task);
```

다른 객체를 빌려 사용하면서 수정할 수 있습니다.

### `T*`

```cpp
Task* find(TaskId id);
```

pointer 자체만으로는 다음이 명확하지 않습니다.

* `nullptr`이 가능한가?
* 소유권이 있는가?
* 얼마 동안 유효한가?

따라서 API 규칙이나 더 구체적인 타입으로 의미를 표현해야 합니다.

현대 C++에서는 일반적으로 raw pointer를 **소유권 표현 용도**로 사용하는 것을 피합니다.

### 비소유 view

다음 타입은 보통 원본 데이터를 소유하지 않습니다.

* `std::string_view`
* `std::span`
* iterator
* raw pointer
* reference

예:

```cpp
std::string_view title(const Task& task) {
    return task.title();
}
```

이 함수가 안전한지는 `task.title()`이 무엇을 반환하는지에 따라 달라집니다.

예를 들어 `Task` 내부의 `std::string`을 가리키는 `string_view`라면 반환된 view의 유효 기간은 최소한 다음 조건에 의존합니다.

* `Task` 객체가 살아 있어야 함
* 내부 문자열 객체가 살아 있어야 함
* 문자열을 변경하는 연산 때문에 기존 저장 공간이 무효화되지 않아야 함

따라서 다음과 같은 코드는 위험할 수 있습니다.

```cpp
std::string_view view = title(Task{});
```

문장이 끝나면서 임시 `Task`가 파괴된다면 `view`는 그 이후 사용할 수 없는 dangling view가 됩니다.

## 수명과 유효성은 같은 질문이 아닐 수 있습니다

원본 객체가 아직 살아 있더라도 observer가 무효화될 수 있습니다.

대표적으로 `std::vector`가 있습니다.

```cpp
std::vector<int> values{1, 2, 3};

int* first = &values[0];

values.push_back(4);
```

`push_back()` 때문에 vector가 더 큰 저장 공간을 할당하고 원소를 옮겼다면 기존 원소를 가리키던 `first`는 무효화됩니다.

`values` 객체 자체는 계속 살아 있지만 그 내부 저장 공간의 주소가 바뀐 것입니다.

따라서 observer의 안전성을 판단할 때는 다음을 모두 확인해야 합니다.

```text
원본 객체의 수명
+
해당 연산의 iterator/reference/pointer invalidation 규칙
```

## 복사 의미

일반적인 값 타입은 복사본을 변경해도 원본이 독립적으로 유지되는 의미를 가집니다.

```cpp
Task a{1, "compile"};

Task b = a;

b.rename("test");
```

일반적으로 기대하는 결과는 다음과 같습니다.

```text
a → "compile"
b → "test"
```

하지만 클래스가 owning raw pointer를 직접 가진다면 compiler가 자동 생성한 복사가 위험할 수 있습니다.

```cpp
class Buffer {
private:
    char* data_;
};
```

기본 복사는 pointer 값만 복사합니다.

```text
a.data_ ──┐
          ├──→ 같은 allocation
b.data_ ──┘
```

두 객체가 모두 destructor에서 `delete[] data_`를 실행하면 같은 allocation을 두 번 해제하게 됩니다.

따라서 가능한 경우 다음처럼 이미 자원 수명을 관리하는 타입을 멤버로 사용합니다.

* `std::string`
* `std::vector`
* `std::unique_ptr`
* `std::shared_ptr` — 공유 소유가 실제로 필요한 경우

이 원칙은 흔히 **Rule of Zero**로 연결됩니다.

직접 destructor, copy constructor, move constructor, copy assignment, move assignment를 작성하지 않아도 되도록 멤버 타입 자체가 자원 관리를 담당하게 만드는 방식입니다.

## 이동 의미

이동은 객체를 파괴하는 연산이 아닙니다.

```cpp
std::vector<int> source{1, 2, 3};

std::vector<int> target = std::move(source);
```

여기서 먼저 중요한 점이 있습니다.

### `std::move` 자체가 자원을 이동시키는 것은 아닙니다

`std::move(source)`는 대략적으로 말하면 `source`를 **rvalue로 취급할 수 있게 변환하는 cast**입니다.

실제 이동은 그 결과를 받아 호출되는 move constructor가 수행합니다.

개념적으로:

```text
std::move(source)
      ↓
"source의 자원을 이동 대상으로 사용할 수 있다"는 표현
      ↓
std::vector move constructor 호출
      ↓
실제 자원 이전
```

따라서 다음 코드는 아무것도 이동하지 않습니다.

```cpp
std::move(source);
```

결과를 사용하는 연산이 없기 때문입니다.

## moved-from 객체

표준 library 객체가 이동된 뒤에는 일반적으로 **valid but unspecified state**, 즉 "유효하지만 값은 특정할 수 없는 상태"에 놓입니다.

```cpp
std::vector<int> source{1, 2, 3};
std::vector<int> target = std::move(source);
```

다음과 같이 생각하면 안 됩니다.

```cpp
assert(source.empty());
```

많은 구현에서 비어 있을 가능성이 높더라도 일반적인 이동 의미만으로 그것을 요구해서는 안 됩니다.

반면 객체는 여전히 유효하므로 해당 상태에서 허용된 연산을 수행할 수 있습니다.

예:

```cpp
source.clear();
source = {4, 5, 6};
source.push_back(7);
```

또한 다음은 호출 자체는 가능합니다.

```cpp
source.empty();
```

다만 반환값을 이동 전 상태로부터 예측해서는 안 됩니다.

즉 구분해야 합니다.

```text
호출할 수 있는가?
    vs
무슨 결과가 나올 것이라고 가정할 수 있는가?
```

개별 타입이 moved-from 상태에 대해 더 강한 보장을 문서화했다면 그 보장을 사용할 수 있습니다.

## 이동 전용 타입

복사라는 의미가 자연스럽지 않은 자원은 이동만 허용하는 경우가 많습니다.

대표적으로 파일 handle의 유일 소유권이 있습니다.

```cpp
class UniqueFile {
public:
    UniqueFile(const UniqueFile&) = delete;
    UniqueFile& operator=(const UniqueFile&) = delete;

    UniqueFile(UniqueFile&& other) noexcept;
    UniqueFile& operator=(UniqueFile&& other) noexcept;

    ~UniqueFile();
};
```

두 `UniqueFile`이 동일한 OS file handle을 각각 독립적으로 소유한다고 복사하면 누가 닫아야 하는지 불명확해집니다.

따라서 복사는 금지하고 소유권을 하나의 객체에서 다른 객체로 이동시킵니다.

```text
before

source ──→ file handle
target    없음


after move

source    비소유 상태
target ──→ file handle
```

## `noexcept`와 이동

실제로 예외를 던지지 않는 move constructor는 `noexcept`로 선언하는 것이 중요합니다.

```cpp
UniqueFile(UniqueFile&& other) noexcept;
```

특히 `std::vector` 같은 container가 재할당할 때 기존 원소를 새 저장 공간으로 옮겨야 합니다.

```text
old storage

[A][B][C]

      ↓ reallocation

new storage

[A'][B'][C']
```

원소 타입이 복사 가능하고 move constructor가 예외를 던질 가능성이 있다면 container는 강한 예외 안전성을 유지하기 위해 이동 대신 복사를 선택할 수 있습니다.

반대로 이동이 `noexcept`라면 container는 자원을 안전하게 이동할 수 있음을 알 수 있습니다.

다만 이를 다음처럼 과장해서 이해하면 안 됩니다.

```text
noexcept가 없으면 vector는 절대로 이동하지 않는다
```

타입이 복사 불가능한 경우 등에는 throwing move를 사용해야 할 수도 있습니다.

핵심은 `noexcept`가 **container가 안전하게 이동을 선택할 수 있는 중요한 정보**라는 점입니다.

## 함수 signature로 사용 의도를 드러냅니다

```cpp
void inspect(const Task& task);
```

호출 중 `Task`를 읽기 위해 빌립니다.

```cpp
void update(Task& task);
```

호출 중 기존 `Task`를 수정합니다.

```cpp
void consume(Task task);
```

함수 내부에서 독립적인 값을 가집니다.

함수가 그 값을 내부 상태에 저장한다면 다음 패턴을 사용할 수 있습니다.

```cpp
class Worker {
public:
    explicit Worker(std::string name)
        : name_(std::move(name)) {}

private:
    std::string name_;
};
```

호출자가 lvalue를 넘기면 한 번 복사한 뒤 멤버로 이동할 수 있습니다.

```cpp
std::string name = "worker";
Worker worker{name};
```

호출자가 rvalue를 넘기면 불필요한 복사를 피할 수 있습니다.

```cpp
Worker worker{"worker"};
```

따라서 "큰 타입은 무조건 `const&`"라는 규칙은 지나치게 단순합니다.

함수가 값을 **저장해야 한다면** 값 parameter가 적절할 수 있습니다.

소유권을 명시적으로 이전한다면 다음과 같은 타입이 더 분명합니다.

```cpp
void install(std::unique_ptr<Task> task);
```

호출자는 일반적으로 다음처럼 전달합니다.

```cpp
auto task = std::make_unique<Task>();

install(std::move(task));
```

이후 `task`는 보통 `nullptr` 상태가 됩니다. `std::unique_ptr`은 이동 후 상태가 `nullptr`이라는 명확한 보장을 제공합니다.

비소유 nullable 결과라면 다음처럼 raw pointer가 적절할 수도 있습니다.

```cpp
Task* find(TaskId id);
```

이 경우 API 문서에서는 최소한 다음을 명확히 해야 합니다.

* `nullptr`을 반환할 수 있는가?
* 반환된 pointer는 누가 소유하는가?
* 어떤 연산까지 유효한가?

## copy elision과 `std::move`

지역 객체를 값으로 반환할 때는 일반적으로 그냥 객체 이름을 반환합니다.

```cpp
Task make_task() {
    Task task{/* ... */};
    return task;
}
```

compiler는 NRVO를 적용하여 `task`를 최종 반환 위치에 직접 생성할 수 있습니다.

다음처럼 작성하는 것은 일반적으로 피합니다.

```cpp
return std::move(task);
```

`std::move(task)`는 `task`라는 이름 자체를 반환하는 형태가 아니므로 NRVO 적용을 방해할 수 있습니다.

따라서 지역 객체 반환에서는 특별한 이유가 없다면 다음을 사용합니다.

```cpp
return task;
```

또한 현대 C++에서는 "값 반환은 무조건 비싸다"는 전제를 두지 않는 것이 중요합니다.

copy elision과 이동 의미 때문에 값 반환이 자연스럽고 효율적인 API가 많습니다.

## 자주 놓치는 문제

* 지역 객체의 reference 또는 pointer를 반환하고 함수가 끝난 뒤 사용합니다.
* 임시 객체 내부를 가리키는 `string_view`를 저장합니다.
* owner보다 observer pointer/reference가 오래 살아남습니다.
* `vector` 재할당 뒤 기존 iterator, pointer, reference를 계속 사용합니다.
* 이동한 객체가 이동 전 값을 그대로 가지고 있다고 가정합니다.
* `std::move()` 자체가 이동을 수행한다고 생각합니다.
* owning raw pointer를 가진 클래스에서 compiler가 만든 기본 복사를 그대로 사용합니다.
* callback이 지역 변수를 reference로 capture했는데 callback은 지역 변수보다 오래 살아 있습니다.

예:

```cpp
std::function<void()> callback;

{
    std::string name = "worker";

    callback = [&name] {
        std::cout << name;
    };
}

callback(); // name은 이미 파괴됨
```

capture 방식만 볼 것이 아니라 **callback이 언제 실행되는가**까지 확인해야 합니다.

## 코드를 읽을 때 확인할 질문

객체나 parameter를 볼 때 다음 순서로 확인하면 좋습니다.

```text
1. 누가 객체를 생성했는가?
2. 누가 객체를 소유하는가?
3. 언제 파괴되는가?
4. 현재 변수는 값인가 observer인가?
5. observer라면 원본보다 오래 살아남을 수 있는가?
6. 원본이 살아 있어도 재할당 등으로 observer가 무효화될 수 있는가?
7. 복사하면 독립된 상태가 되는가?
8. 이동하면 원본에 대해 무엇이 보장되는가?
```

## 완료 기준

* 객체의 수명과 pointer에 주소 값이 남아 있는 것을 구분합니다.
* 원본 객체의 수명과 iterator/reference/pointer의 유효 기간을 별도로 판단합니다.
* 초기화, 복사 생성, 이동 생성, 복사 대입, 이동 대입을 구분합니다.
* 값, reference, pointer, view가 나타내는 사용 방식을 설명합니다.
* 비소유 view를 반환하거나 저장할 때 원본의 수명 조건을 확인합니다.
* `std::move`가 실제 이동 연산이 아니라 rvalue 변환이라는 점을 설명합니다.
* moved-from 객체의 "valid but unspecified" 의미를 설명합니다.
* `noexcept` move가 container의 재배치 선택에 왜 영향을 주는지 설명합니다.
* 지역 객체 반환에서 불필요한 `std::move`를 사용하지 않습니다.
