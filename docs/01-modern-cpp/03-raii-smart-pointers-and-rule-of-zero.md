# RAII·smart pointer·Rule of Zero

## 목표

파일, socket, mutex, 동적 메모리처럼 **획득한 뒤 반드시 해제해야 하는 자원**을 객체 수명에 연결합니다.

핵심은 정상적인 `return` 경로에서만 정리하는 것이 아닙니다. 중간 실패나 예외가 발생하더라도 이미 획득한 자원이 자동으로 정리되어야 합니다.

다음을 판단할 수 있어야 합니다.

* 이 자원은 누가 소유합니까?
* 소유자는 언제 자원을 해제합니까?
* 함수가 중간에 끝나도 해제가 보장됩니까?
* 복사와 이동이 자원 소유권에 어떤 의미를 가집니까?
* 정리 과정에서 발생한 오류를 어디에서 보고합니까?

## RAII가 해결하는 문제

수동 자원 관리는 함수의 모든 종료 경로에서 정리 코드를 실행해야 합니다.

```cpp
FILE* file = std::fopen(path, "rb");

if (file == nullptr) {
    return error;
}

char* buffer = new char[size];

// 이후 실패하거나 return할 때마다
// delete[] buffer;
// std::fclose(file);
// 를 정확히 실행해야 합니다.
```

예를 들어 중간에 새로운 검사가 추가되면 정리를 빠뜨리기 쉽습니다.

```cpp
if (!validate(buffer)) {
    delete[] buffer;
    std::fclose(file);
    return error;
}
```

예외까지 고려하면 더 복잡해집니다.

RAII(Resource Acquisition Is Initialization)는 **자원의 유효 기간을 객체의 수명과 연결하는 방식**입니다.

```cpp
std::ifstream input{path, std::ios::binary};
std::vector<char> buffer(size);
```

`input`은 자신의 파일 자원을 관리하고 `buffer`는 자신의 동적 메모리를 관리합니다.

scope가 끝나면 이미 완전히 생성된 지역 객체들은 생성의 역순으로 파괴됩니다.

```cpp
void process() {
    std::ifstream input{path};
    std::vector<char> buffer(size);

    if (something_failed()) {
        return;
    }

    // ...
}
```

중간에서 `return`하더라도 다음이 자동으로 실행됩니다.

```text
buffer 소멸
↓
input 소멸
```

예외로 stack unwinding이 일어나는 경우에도 동일하게 이미 생성된 자동 저장 기간 객체가 파괴됩니다.

RAII의 핵심은 단순히 "소멸자를 사용한다"가 아닙니다.

```text
자원 획득
    ↓
RAII 객체가 즉시 소유
    ↓
모든 정상/예외 종료 경로
    ↓
객체 소멸
    ↓
자원 해제
```

따라서 자원을 획득한 순간부터 **소유자가 없는 시간 구간을 만들지 않는 것**이 중요합니다.

## 소멸자와 실패

소멸자는 일반적으로 예외를 외부로 던지지 않아야 합니다.

특히 다음 상황을 생각할 수 있습니다.

```cpp
void run() {
    OutputFile file;

    throw std::runtime_error{"processing failed"};
}
```

이미 첫 번째 예외 때문에 stack unwinding이 진행되는 동안 `file`의 소멸자가 다시 예외를 던지면 프로그램은 일반적으로 `std::terminate()`로 종료됩니다.

따라서 자원 정리를 담당하는 소멸자는 보통 실패를 외부에 예외로 전달하지 않습니다.

```cpp
class OutputFile {
public:
    void finish();        // 실패를 호출자에게 보고할 수 있음

    ~OutputFile() noexcept {
        // 가능한 범위에서 정리
        // 예외를 밖으로 던지지 않음
    }
};
```

여기서 중요한 문제가 하나 있습니다.

**모든 정리 실패를 무시해도 된다는 뜻은 아닙니다.**

예를 들어 출력 파일의 `flush()` 또는 `close()` 실패는 실제 데이터 손실을 의미할 수 있습니다. 이를 호출자가 반드시 알아야 한다면 명시적인 완료 연산을 제공합니다.

```cpp
file.finish();
```

`finish()`에서는 오류를 반환하거나 예외를 던질 수 있습니다.

소멸자는 이후 남은 자원을 정리하는 최후의 안전망 역할을 합니다.

즉 두 역할을 구분합니다.

```text
finish()
- 작업 완료 여부를 확인
- 실패를 caller에게 보고

destructor
- 남은 자원을 반드시 정리
- 일반적으로 실패를 밖으로 던지지 않음
```

## `std::unique_ptr`

하나의 객체만 동적 객체를 소유해야 한다면 `std::unique_ptr`을 먼저 고려합니다.

```cpp
auto task = std::make_unique<Task>(id, name);
```

이 시점의 소유 관계는 다음과 같습니다.

```text
task
  │
  └── owns → Task
```

복사는 허용되지 않습니다.

```cpp
auto other = task; // compile error
```

소유권을 이전하려면 이동합니다.

```cpp
queue.push_back(std::move(task));
```

이후 소유 관계는 다음처럼 바뀝니다.

```text
before

task ──→ Task


after

task      nullptr
queue ──→ Task
```

`std::unique_ptr`은 이동 후 원본이 `nullptr`이 된다는 보장을 제공합니다.

따라서 다음처럼 확인할 수 있습니다.

```cpp
if (!task) {
    // 소유권이 이전되었습니다.
}
```

## `get()`은 소유권을 넘기지 않습니다

기존 C API나 raw pointer를 받는 API에 객체를 잠시 전달해야 할 수 있습니다.

```cpp
use_task(task.get());
```

`get()`은 내부 pointer 값을 반환하지만 소유권은 여전히 `task`에 있습니다.

```text
task ───── owns ─────→ Task
                       ↑
                       │
               temporary observer
```

따라서 `use_task()`가 다음을 하지 않는다는 조건이 필요합니다.

* pointer를 `delete`함
* 소유권을 가져감
* `task`보다 오래 pointer를 저장함

API가 실제로 소유권을 가져간다면 `get()`은 맞지 않습니다.

그 경우에는 해당 API의 소유권 규칙에 맞춰 `release()`나 별도의 wrapper를 고려해야 합니다.

`release()`는 매우 다릅니다.

```cpp
Task* raw = task.release();
```

이제 `unique_ptr`은 객체를 소유하지 않습니다.

```text
task → nullptr

raw ──→ Task
```

`raw`를 누군가 다시 소유하지 않으면 memory leak이 발생합니다.

따라서 `release()`는 명확한 소유권 이전이 필요한 경우에만 사용합니다.

## C API 자원과 custom deleter

`unique_ptr`은 `new`로 만든 객체에만 사용할 수 있는 것이 아닙니다.

해제 함수를 지정하면 C API handle도 관리할 수 있습니다.

```cpp
struct FileCloser {
    void operator()(std::FILE* file) const noexcept {
        if (file != nullptr) {
            std::fclose(file);
        }
    }
};

using FilePtr = std::unique_ptr<std::FILE, FileCloser>;
```

사용:

```cpp
FilePtr file{std::fopen(path, "rb")};

if (!file) {
    throw std::runtime_error{"failed to open file"};
}
```

scope가 끝나면 `FileCloser`가 호출됩니다.

이 방식은 다음과 같은 C API 자원에도 확장할 수 있습니다.

* `FILE*`
* socket handle
* database handle
* graphics API handle
* compression library context

단, 자원의 "빈 값"이 단순한 null pointer가 아닌 경우에는 `unique_ptr`보다 별도의 RAII wrapper class가 더 자연스러울 수 있습니다.

예를 들어 Unix file descriptor는 `int`이고 유효하지 않은 값은 보통 `-1`입니다.

이 경우 다음처럼 직접 타입을 만드는 편이 명확합니다.

```cpp
class UniqueFd {
public:
    explicit UniqueFd(int fd = -1) noexcept
        : fd_(fd) {}

    ~UniqueFd() noexcept {
        if (fd_ != -1) {
            ::close(fd_);
        }
    }

    UniqueFd(const UniqueFd&) = delete;
    UniqueFd& operator=(const UniqueFd&) = delete;

    UniqueFd(UniqueFd&& other) noexcept
        : fd_(std::exchange(other.fd_, -1)) {}

private:
    int fd_;
};
```

## `std::shared_ptr`는 공유 소유를 표현합니다

`std::shared_ptr`는 단순히 "복사 가능한 편한 smart pointer"가 아닙니다.

여러 객체가 하나의 대상 객체의 **수명을 공동으로 소유한다는 의미**입니다.

```cpp
auto task = std::make_shared<Task>();

auto a = task;
auto b = task;
```

개념적으로:

```text
task ─┐
a    ─┼──→ Task
b    ─┘
```

마지막 `shared_ptr` 소유자가 사라질 때 `Task`가 파괴됩니다.

따라서 다음과 같은 상황에는 적절할 수 있습니다.

* 여러 비동기 작업이 동일 객체의 생존을 보장해야 함
* 객체의 수명 종료 시점을 한 소유자로 특정하기 어려운 실제 공유 소유 모델
* 여러 구성 요소가 동등한 소유자임

반대로 단순히 여러 함수에서 객체를 사용한다는 이유로 `shared_ptr`가 필요한 것은 아닙니다.

```cpp
void inspect(const Task& task);
```

처럼 non-owning reference로 충분한 경우가 많습니다.

`shared_ptr`를 습관적으로 사용하면 "누가 이 객체를 죽이는가?"라는 질문의 답이 불분명해집니다.

## `shared_ptr` cycle

다음처럼 서로가 서로를 소유하면 문제가 생깁니다.

```cpp
struct A {
    std::shared_ptr<B> b;
};

struct B {
    std::shared_ptr<A> a;
};
```

관계:

```text
A ──shared──→ B
↑             │
└──shared─────┘
```

외부의 모든 `shared_ptr`가 없어져도 내부 reference count가 서로를 유지합니다.

따라서 객체가 파괴되지 않습니다.

한쪽 관계가 **소유가 아니라 관찰 관계**라면 `std::weak_ptr`를 사용합니다.

```cpp
struct B {
    std::weak_ptr<A> a;
};
```

사용할 때는 먼저 살아 있는지 확인합니다.

```cpp
if (auto owner = a.lock()) {
    owner->do_something();
}
```

`lock()`은 대상이 살아 있다면 임시 `shared_ptr`를 반환하고, 이미 파괴되었다면 빈 `shared_ptr`를 반환합니다.

## lock도 RAII로 관리합니다

mutex의 lock/unlock도 반드시 짝을 이루어야 하는 자원 관리 문제입니다.

수동 코드는 위험합니다.

```cpp
state.mutex.lock();

state.value += 1;

might_throw();

state.mutex.unlock();
```

`might_throw()`가 예외를 던지면 `unlock()`까지 도달하지 못합니다.

mutex는 잠긴 상태로 남고 이후 thread가 영원히 대기할 수 있습니다.

RAII lock을 사용합니다.

```cpp
void update(State& state) {
    std::lock_guard lock{state.mutex};

    state.value += 1;
}
```

scope가 끝나면 자동으로 unlock됩니다.

```text
lock_guard 생성
    ↓
mutex lock
    ↓
작업
    ↓
scope 종료
    ↓
lock_guard 소멸
    ↓
mutex unlock
```

`std::lock_guard`는 단순한 scope lock에 적합합니다.

더 복잡한 제어가 필요하면 `std::unique_lock`을 사용할 수 있습니다.

예:

* 나중에 lock
* 중간에 unlock
* 다시 lock
* `std::condition_variable`과 사용

## mutex를 잡은 채 외부 코드를 호출하지 않습니다

다음 코드는 위험할 수 있습니다.

```cpp
void Registry::notify() {
    std::lock_guard lock{mutex_};

    callback_();
}
```

`callback_()`의 구현을 현재 클래스가 통제하지 않는다면 다음 문제가 발생할 수 있습니다.

* callback이 오래 걸림
* callback이 다른 mutex를 획득하여 deadlock 발생
* callback이 다시 `Registry`를 호출하여 동일 mutex를 획득하려 함
* callback이 예외를 던짐
* lock 보유 시간이 예상보다 길어짐

가능하다면 lock 안에서는 보호된 상태만 읽거나 변경하고, 외부 호출은 lock 밖에서 수행합니다.

예:

```cpp
Callback callback;

{
    std::lock_guard lock{mutex_};
    callback = callback_;
}

callback();
```

단, 실제 코드에서는 복사한 상태가 호출 시점에도 유효한지 등 별도의 동시성 규칙을 확인해야 합니다.

## Rule of Zero

가능하면 다음 특수 멤버 함수를 직접 구현하지 않습니다.

* destructor
* copy constructor
* copy assignment operator
* move constructor
* move assignment operator

예:

```cpp
struct Job {
    JobId id;
    std::string name;
    std::vector<std::string> tags;
};
```

각 멤버가 자신의 자원을 올바르게 관리합니다.

따라서 compiler가 생성한 복사·이동·소멸 동작만으로 자연스러운 값 의미를 얻을 수 있습니다.

```cpp
Job a{...};
Job b = a;            // 멤버별 복사
Job c = std::move(a); // 멤버별 이동
```

이것이 **Rule of Zero**의 핵심입니다.

```text
직접 자원 관리 코드를 작성하지 않고
자원 관리 타입을 멤버로 사용한다.
```

Rule of Zero는 단순히 코드 줄 수를 줄이기 위한 규칙이 아닙니다.

직접 관리해야 하는 다음 문제들을 표준 타입에 맡길 수 있습니다.

* self-assignment
* 예외 안전성
* 부분 생성 실패
* 이동 후 상태
* double free
* 누수
* 복사와 이동의 일관성

## 직접 특수 멤버 함수가 필요한 경우

모든 타입이 Rule of Zero가 가능한 것은 아닙니다.

OS handle을 직접 감싸는 낮은 수준의 RAII 타입처럼 실제 자원의 소유자가 되는 타입은 복사와 이동 규칙을 직접 정의해야 할 수 있습니다.

### 유일 소유 자원

예:

```cpp
class UniqueSocket {
public:
    UniqueSocket(const UniqueSocket&) = delete;
    UniqueSocket& operator=(const UniqueSocket&) = delete;

    UniqueSocket(UniqueSocket&& other) noexcept;
    UniqueSocket& operator=(UniqueSocket&& other) noexcept;

    ~UniqueSocket() noexcept;
};
```

복사는 금지하고 이동으로만 소유권을 이전합니다.

### 복사 가능한 자원

복사 자체가 의미 있다면 복사가 무엇을 뜻하는지 정의해야 합니다.

예를 들어 독립적인 buffer라면 깊은 복사가 자연스럽습니다.

```text
original ──→ allocation A

copy     ──→ allocation B
```

두 객체가 같은 raw allocation을 소유하게 만들어서는 안 됩니다.

### 복사도 이동도 부적절한 객체

mutex처럼 객체 자체의 identity와 주소가 중요하고 복제할 의미가 없는 타입도 있습니다.

```cpp
class State {
public:
    State(const State&) = delete;
    State& operator=(const State&) = delete;

private:
    std::mutex mutex_;
};
```

이 경우 포함된 멤버 때문에 타입 자체가 자동으로 non-copyable/non-movable이 될 수도 있습니다.

## 이동 구현에서 주의할 점

이동은 자원 소유자를 바꾸는 연산입니다.

예:

```cpp
UniqueFd::UniqueFd(UniqueFd&& other) noexcept
    : fd_(std::exchange(other.fd_, -1)) {}
```

이동 전:

```text
other.fd_ = 10
```

이동 후:

```text
this->fd_ = 10
other.fd_ = -1
```

원본은 여전히 소멸할 수 있는 상태여야 합니다.

그렇지 않으면 두 객체가 같은 자원을 해제할 수 있습니다.

이동 대입에서는 현재 객체가 이미 자원을 가지고 있을 수 있다는 점도 중요합니다.

```cpp
a = std::move(b);
```

이때 `a`의 기존 자원을 먼저 안전하게 처리해야 합니다.

개념적으로:

```text
a가 기존 자원 A 소유
b가 자원 B 소유

       move assignment

자원 A 해제
a가 B 소유
b는 빈 상태
```

가능하면 직접 이 로직을 구현하기보다 `unique_ptr`, container 등 이미 이동을 올바르게 구현한 타입을 멤버로 사용합니다.

## 생성자 실패와 부분 생성

생성 도중 예외가 발생했다고 합시다.

```cpp
class Worker {
public:
    Worker()
        : first_{},
          second_{} {
        throw std::runtime_error{"failed"};
    }

private:
    Resource first_;
    Resource second_;
};
```

생성자 본문에서 예외가 발생하면 `Worker` 객체 자체는 완전히 생성되지 않았으므로 `Worker::~Worker()`는 호출되지 않습니다.

하지만 이미 생성이 끝난 다음 항목은 자동으로 파괴됩니다.

* 완전히 생성된 base class
* 완전히 생성된 member

그리고 생성의 역순으로 파괴됩니다.

따라서 멤버 자체가 RAII 타입이라면 부분 생성 실패도 안전하게 처리할 수 있습니다.

위험한 패턴은 다음처럼 raw 자원을 먼저 얻고 아직 RAII owner에 넣지 않은 상태에서 예외가 발생하는 것입니다.

```cpp
Resource* raw = acquire_resource();

might_throw();

owner_.reset(raw);
```

`might_throw()`가 실패하면 `raw`를 해제할 주체가 없습니다.

대신 획득 즉시 RAII 객체에 넣습니다.

```cpp
auto resource = make_resource();

might_throw();

owner_ = std::move(resource);
```

즉 중요한 원칙은 다음입니다.

```text
자원을 얻는다
↓
가능한 즉시 소유권을 RAII 객체에 넣는다
↓
그 다음 실패 가능한 작업을 한다
```

## 자주 놓치는 문제

* 자원 handle을 가진 타입에 compiler 기본 복사를 허용하여 두 객체가 같은 자원을 해제합니다.
* 자원을 얻은 뒤 RAII owner에 넣기 전에 예외 가능 작업을 수행합니다.
* 이동 뒤 원본 handle을 빈 상태로 만들지 않습니다.
* 이동 대입에서 대상 객체의 기존 자원을 처리하지 않습니다.
* 소멸자에서 반드시 보고해야 하는 I/O 실패를 예외로 던집니다.
* 단순히 여러 곳에서 사용한다는 이유로 `shared_ptr`를 사용합니다.
* `shared_ptr` cycle을 만들어 객체가 영원히 파괴되지 않습니다.
* `unique_ptr::get()`으로 넘긴 observer가 owner보다 오래 살아남습니다.
* `release()`를 호출한 뒤 새 owner를 만들지 않아 자원이 누수됩니다.
* mutex를 수동으로 `lock()`한 뒤 예외 경로에서 `unlock()`을 빠뜨립니다.
* mutex를 잡은 채 외부 callback이나 긴 작업을 수행합니다.

## 코드를 읽을 때 확인할 질문

자원을 가진 코드를 보면 다음을 확인합니다.

1. 자원을 획득하는 위치는 어디입니까?
2. 획득 직후 누가 소유합니까?
3. 정상 종료 시 누가 해제합니까?
4. 중간 `return`에서도 해제됩니까?
5. 예외가 발생해도 해제됩니까?
6. 타입을 복사하면 소유권은 어떻게 됩니까?
7. 타입을 이동하면 원본은 안전하게 소멸할 수 있습니까?
8. `shared_ptr`라면 실제로 공동 소유가 필요한 이유가 있습니까?
9. 소멸자가 보고할 수 없는 오류는 명시적인 완료 함수에서 처리해야 합니까?

## 완료 기준

* 각 자원의 소유자와 해제 시점을 타입으로 설명합니다.
* RAII가 정상 종료뿐 아니라 예외와 중간 실패에서도 필요한 이유를 설명합니다.
* `unique_ptr`의 이동이 소유권 이전임을 설명합니다.
* `get()`과 `release()`의 차이를 구분합니다.
* 유일 소유와 실제 공유 소유를 구분합니다.
* `shared_ptr` cycle과 `weak_ptr`의 역할을 설명합니다.
* mutex를 RAII lock으로 관리합니다.
* Rule of Zero를 우선 적용하고 직접 특수 멤버 함수를 작성해야 하는 경우를 구분합니다.
* 생성 도중 실패할 때 완성된 멤버가 어떻게 정리되는지 설명합니다.
* 이동 전용 타입의 원본이 이동 뒤에도 안전하게 소멸되는지 확인합니다.