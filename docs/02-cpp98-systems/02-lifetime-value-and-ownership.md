# C++98 수명·값·소유권

## 목표

C++11의 `std::unique_ptr`, 이동 생성자, 이동 대입 연산자 없이도 메모리와 file descriptor 같은 자원의 소유자를 명확하게 관리합니다.

핵심은 단순히 `delete`를 빠뜨리지 않는 것이 아닙니다.

다음 상황에서도 객체 상태가 올바르게 유지되어야 합니다.

* 객체 복사
* 객체 대입
* 객체 소멸
* 메모리 할당 실패
* 생성자 실행 중 예외
* 컨테이너 삽입 실패
* 함수가 중간에 실패하는 경우

특히 다음 두 문제를 막아야 합니다.

* 같은 자원을 두 번 해제하는 것
* 실패 때문에 기존에 정상적이던 값을 잃는 것

## Rule of Three

클래스가 직접 자원을 소유하고 있으며 다음 셋 중 하나를 직접 구현해야 한다면 나머지 둘도 함께 검토해야 합니다.

* 소멸자
* 복사 생성자
* 복사 대입 연산자

이를 Rule of Three라고 부릅니다.

```cpp
class TextBuffer {
public:
    TextBuffer(const char *text);

    TextBuffer(const TextBuffer &other);
    TextBuffer &operator=(const TextBuffer &other);

    ~TextBuffer();

private:
    char *data_;
    std::size_t size_;
};
```

예를 들어 `data_`가 `new[]`로 확보한 메모리를 소유한다고 가정합니다.

컴파일러가 생성하는 기본 복사는 pointer가 가리키는 메모리를 복제하지 않습니다.

```text
object A
data_ ─────┐
           ▼
        [hello]
           ▲
data_ ─────┘
object B
```

즉 주소만 복사됩니다.

그 결과 다음 문제가 발생할 수 있습니다.

* A와 B가 같은 메모리를 소유한다고 생각함
* A가 삭제한 뒤 B의 pointer가 dangling pointer가 됨
* A와 B의 소멸자가 같은 주소를 각각 `delete[]`함
* 한 객체가 메모리를 수정하면 다른 객체에서도 변경이 보임

따라서 값 타입으로 사용할 객체가 raw pointer로 메모리를 소유한다면 복사 의미를 직접 정해야 합니다.

## 깊은 복사

두 객체가 서로 독립적인 값을 가져야 한다면 새로운 메모리를 확보하고 실제 내용까지 복사해야 합니다.

```cpp
TextBuffer::TextBuffer(const TextBuffer &other)
    : data_(new char[other.size_ + 1]),
      size_(other.size_) {
    std::memcpy(data_, other.data_, size_ + 1);
}
```

복사 후에는 다음 관계가 됩니다.

```text
object A              object B
data_ ──► [hello]     data_ ──► [hello]
```

내용은 같지만 서로 다른 메모리입니다.

따라서 한 객체의 내용을 변경하거나 소멸해도 다른 객체에는 영향을 주지 않습니다.

공유 소유가 정말 필요하다면 별도의 reference count 같은 관리 방법이 필요합니다. 그러나 단순한 문자열이나 buffer 같은 값 타입이라면 깊은 복사가 더 단순한 경우가 많습니다.

## 복사 대입과 강한 예외 보장

다음과 같은 대입 구현은 위험합니다.

```cpp
delete[] data_;
data_ = new char[other.size_ + 1];
```

기존 메모리를 먼저 삭제했기 때문입니다.

`new`가 실패하면 객체는 이전 값을 이미 잃었습니다.

가능하다면 다음 성질을 유지하는 것이 좋습니다.

> 대입이 성공하면 새 값을 갖고, 실패하면 기존 값이 그대로 남는다.

이를 구현하는 대표적인 방법이 copy-and-swap입니다.

```cpp
TextBuffer &TextBuffer::operator=(const TextBuffer &other) {
    TextBuffer candidate(other);

    swap(candidate);

    return *this;
}
```

실행 순서는 다음과 같습니다.

1. `other`의 복사본인 `candidate`를 만듭니다.
2. 복사 중 메모리 할당이 실패하면 예외가 발생합니다.
3. 아직 현재 객체는 건드리지 않았으므로 기존 값이 유지됩니다.
4. 복사가 성공하면 현재 객체와 `candidate`의 내부 자원을 교환합니다.
5. 함수가 끝날 때 `candidate`가 소멸하면서 이전 자원을 정리합니다.

따라서 자기 대입도 별도의 특별 처리를 하지 않고 정상적으로 처리할 수 있습니다.

단, `swap()` 자체가 예외 없이 수행되는 단순한 pointer와 크기 값 교환이어야 이 방식의 장점을 제대로 얻을 수 있습니다.

예:

```cpp
void TextBuffer::swap(TextBuffer &other) {
    char *data = data_;
    data_ = other.data_;
    other.data_ = data;

    std::size_t size = size_;
    size_ = other.size_;
    other.size_ = size;
}
```

## 소유 pointer와 관찰 pointer

C++98의 raw pointer 타입만 봐서는 그 pointer가 객체를 **소유하는지** 알 수 없습니다.

다음 두 변수는 타입이 같습니다.

```cpp
Handler *a;
Handler *b;
```

하지만 실제 의미는 완전히 다를 수 있습니다.

* `a`: 이 pointer를 가진 객체가 마지막에 `delete`해야 함
* `b`: 다른 객체가 소유하며 여기서는 잠시 가리키기만 함

따라서 C++98에서는 소유 관계를 프로그램 설계로 명시해야 합니다.

```cpp
class Router {
private:
    std::map<std::string, Handler *> handlers_;
};
```

여기서 `Router`가 모든 `Handler`를 소유한다고 정했다면 소멸자에서 해당 객체를 정리해야 합니다.

```cpp
Router::~Router() {
    std::map<std::string, Handler *>::iterator it = handlers_.begin();

    while (it != handlers_.end()) {
        delete it->second;
        ++it;
    }
}
```

반대로 다음과 같이 반환되는 pointer가 소유권을 넘기지 않는다고 정할 수 있습니다.

```cpp
const Handler *Router::find(const std::string &command) const;
```

호출자는 이 pointer를 `delete`하면 안 됩니다.

또한 반환된 pointer의 유효 기간도 제한됩니다.

예를 들어 다음 중 하나가 발생하면 더 이상 유효하지 않을 수 있습니다.

* `Router`가 소멸함
* 해당 `Handler`가 제거됨
* 해당 객체를 소유한 다른 코드가 삭제함

raw pointer를 반환할 때는 **누가 해제하는가**뿐 아니라 **얼마나 오래 유효한가**도 함께 정해야 합니다.

## 소유권 이전

C++98에는 일반적인 이동 의미론이 없으므로 raw pointer를 통해 소유권을 넘기는 API는 특히 조심해야 합니다.

예를 들어 다음 함수가 있다고 가정합니다.

```cpp
void Router::add(const std::string &name, Handler *handler);
```

이 선언만으로는 `handler`의 소유권이 어떻게 되는지 알 수 없습니다.

최소한 다음을 정해야 합니다.

* 호출 전에는 누가 소유하는가
* 성공하면 누가 소유하는가
* 실패하면 누가 소유하는가
* 중복된 이름 때문에 등록하지 못하면 누가 `delete`하는가

예를 들어 다음 규칙을 정할 수 있습니다.

> 호출 전에는 caller가 소유한다. `add()`가 성공한 순간 Router로 소유권이 이전된다. `add()`가 실패하면 caller가 계속 소유한다.

그러면 caller는 다음과 같은 형태가 됩니다.

```cpp
Handler *handler = new Handler;

try {
    router.add("PING", handler);
    handler = 0;
}
catch (...) {
    delete handler;
    throw;
}
```

하지만 이런 API는 실수하기 쉽습니다.

가능하다면 객체의 생성과 소유 컨테이너 등록을 같은 코드 영역에서 수행하여 소유권이 애매한 raw pointer가 외부를 돌아다니는 시간을 줄이는 편이 낫습니다.

## 배열과 객체 수명

다음 코드는 두 가지 작업을 한 번에 수행합니다.

```cpp
T *items = new T[n];
```

* `n`개 객체를 저장할 메모리 확보
* `T` 객체 `n`개의 생성자 실행

반면 `std::allocator<T>`를 사용하면 **raw storage 확보**와 **객체 생성**을 분리할 수 있습니다.

```cpp
std::allocator<T> allocator;

T *memory = allocator.allocate(capacity);

allocator.construct(memory + index, value);
```

`allocate(capacity)`가 성공했다고 해서 `T` 객체가 `capacity`개 존재하는 것은 아닙니다.

단지 `T` 객체를 저장할 수 있는 메모리 공간이 확보된 것입니다.

예를 들어 다음 상태라면,

```text
capacity = 8
size     = 3
```

객체가 실제로 존재하는 범위는 보통 다음뿐입니다.

```text
[0] [1] [2] [raw] [raw] [raw] [raw] [raw]
```

따라서 실제로 생성한 객체에만 `destroy()`를 호출해야 합니다.

```cpp
for (std::size_t i = 0; i < size; ++i)
    allocator.destroy(memory + i);

allocator.deallocate(memory, capacity);
```

아직 생성하지 않은 `[size, capacity)` 영역에 `destroy()`를 호출하면 존재하지 않는 객체의 소멸자를 호출하는 것이므로 잘못된 동작입니다.

이 구분은 직접 vector와 비슷한 컨테이너를 구현할 때 특히 중요합니다.

* capacity: 저장할 수 있도록 확보된 메모리 수
* size: 실제로 생성되어 살아 있는 객체 수

둘은 같은 개념이 아닙니다.

## 생성자 실패

여기는 특히 주의해야 합니다.

객체의 생성자가 예외로 끝나면 **그 객체 자체의 소멸자는 호출되지 않습니다.**

예를 들어:

```cpp
TextBuffer::TextBuffer(const char *text)
    : data_(0),
      size_(length(text)) {
    data_ = new char[size_ + 1];

    copy_text(data_, text, size_);
}
```

`new`가 실패한다면 아직 메모리를 얻지 못했으므로 `data_` 누수는 없습니다.

문제는 **`new`는 성공했지만 그 이후 코드가 예외를 던지는 경우**입니다.

```cpp
data_ = new char[size_ + 1];

// 여기서 예외가 발생
copy_text(data_, text, size_);
```

`TextBuffer` 객체의 생성이 완료되지 않았으므로 `TextBuffer::~TextBuffer()`는 호출되지 않습니다.

따라서 `copy_text()`가 예외를 던질 수 있다면 `data_`가 누수될 수 있습니다.

즉 기존 설명처럼

> raw pointer를 멤버에 저장하면 이후 소멸자가 정리해 준다

라고 생각하면 안 됩니다.

C++에서는 **생성이 완료된 멤버 객체**의 소멸자는 자동으로 호출되지만, 생성 중인 가장 바깥 객체 자신의 소멸자는 호출되지 않습니다.

raw pointer처럼 자체적으로 자원을 정리하지 않는 멤버는 자동 정리되지 않습니다.

따라서 두 가지 방법 중 하나가 필요합니다.

### 이후 연산이 예외를 던지지 않게 구성

예를 들어 `copy_text()`가 단순 byte copy이며 예외를 던지지 않는다면 다음 구성은 안전할 수 있습니다.

```cpp
TextBuffer::TextBuffer(const char *text)
    : data_(0),
      size_(length(text)) {
    data_ = new char[size_ + 1];
    std::memcpy(data_, text, size_ + 1);
}
```

### 예외가 가능한 작업이라면 직접 정리

```cpp
TextBuffer::TextBuffer(const char *text)
    : data_(0),
      size_(length(text)) {
    char *candidate = new char[size_ + 1];

    try {
        copy_text(candidate, text, size_);
    }
    catch (...) {
        delete[] candidate;
        throw;
    }

    data_ = candidate;
}
```

핵심은 **생성자가 실패해도 그 시점까지 획득한 자원이 모두 정리되어야 한다는 것**입니다.

## 멤버 객체와 생성자 실패

raw pointer와 일반 멤버 객체의 차이도 중요합니다.

```cpp
class Example {
private:
    std::string name_;
    Resource resource_;
};
```

`resource_` 생성 중 예외가 발생하면 이미 생성이 끝난 `name_`의 소멸자는 자동으로 호출됩니다.

C++ 런타임이 생성에 성공한 멤버 객체는 역순으로 정리하기 때문입니다.

그러나 다음 raw pointer는 다릅니다.

```cpp
char *data_;
```

pointer 자체는 단순한 값이므로 `delete[]`를 자동으로 호출하지 않습니다.

이 차이가 RAII가 중요한 이유입니다.

C++98에서도 가능한 경우 직접 raw resource를 관리하는 클래스 하나에 소유를 집중시키고, 다른 클래스는 그 자원 관리 클래스를 값 멤버로 사용하는 편이 안전합니다.

## 컨테이너와 값 타입

```cpp
std::map<Key, Value>
```

처럼 객체 자체를 컨테이너에 저장한다면 `Value`의 복사 동작이 올바르게 정의되어 있어야 합니다.

C++98 표준 라이브러리 컨테이너는 내부 구현 과정에서 객체를 복사할 수 있습니다.

따라서 다음과 같이 생각하면 안 됩니다.

> `insert()`를 한 번 호출했으니 복사도 정확히 한 번 일어날 것이다.

프로그램은 컨테이너 구현이 내부적으로 정확히 몇 번 복사하는지에 의존해서는 안 됩니다.

`Value`가 복사 가능한 타입이라면 복사가 몇 번 발생해도 의미상 올바른 값을 만들어야 합니다.

## 컨테이너와 pointer 주소

다음 두 경우를 구분해야 합니다.

```cpp
std::map<Key, Value>
```

와

```cpp
std::map<Key, Value *>
```

첫 번째는 `Value` 객체 자체를 컨테이너가 보관합니다.

두 번째는 pointer 값만 컨테이너에 보관합니다.

`Value *`를 저장했다고 해서 컨테이너가 자동으로 해당 객체를 소유하거나 삭제하는 것은 아닙니다.

```cpp
std::map<std::string, Handler *> handlers;
```

이 경우 `Handler`의 실제 수명과 해제는 별도로 관리해야 합니다.

또한 컨테이너의 요소 주소 안정성은 컨테이너 종류마다 다릅니다. `vector`, `map`, `list`를 모두 같은 방식으로 생각해서는 안 됩니다.

특정 컨테이너의 요소 pointer나 iterator를 장기간 저장하려면 그 컨테이너의 삽입·삭제 연산이 iterator와 reference를 언제 무효화하는지 확인해야 합니다.

## 실패와 상태 변경 순서

자원을 관리하는 코드에서는 일반적으로 **실패할 수 있는 작업을 먼저 수행하고, 기존 상태 변경을 나중에 수행하는 것**이 안전합니다.

예를 들어:

```text
새 자원 확보
    ↓
새 값 완성
    ↓
기존 객체와 교환
    ↓
이전 자원 정리
```

이 순서를 사용하면 앞 단계가 실패했을 때 기존 정상 상태를 유지하기 쉽습니다.

반대로

```text
기존 자원 삭제
    ↓
새 자원 확보
```

순서를 사용하면 새 자원 확보가 실패했을 때 복구하기 어려워집니다.

copy-and-swap도 이 원칙을 적용한 방법입니다.

## 자주 놓치는 문제

* raw owning pointer를 가진 타입에 compiler 기본 복사를 사용합니다.
* 복사 대입에서 기존 자원을 먼저 삭제합니다.
* 자기 대입을 고려하지 않은 코드를 작성합니다.
* 소유권 이전 함수의 실패 경로에서 caller와 callee가 모두 `delete`합니다.
* 반대로 어느 쪽도 `delete`하지 않는 경로를 만듭니다.
* `allocator.allocate()`가 객체 생성까지 수행한다고 생각합니다.
* `capacity` 전체에 객체가 존재한다고 생각합니다.
* 생성자에서 예외가 발생하면 현재 객체의 소멸자가 호출된다고 생각합니다.
* 생성 중 확보한 raw pointer가 자동으로 정리된다고 생각합니다.
* 컨테이너가 raw pointer가 가리키는 객체까지 자동으로 관리한다고 생각합니다.
* 모든 표준 컨테이너에서 요소 주소와 iterator가 같은 방식으로 유지된다고 생각합니다.

## 완료 기준

* Rule of Three가 왜 필요한지 shallow copy와 double deletion을 이용해 설명할 수 있습니다.
* 값 복사와 공유 소유를 구분할 수 있습니다.
* copy-and-swap이 할당 실패 뒤 기존 값을 보존하는 이유를 설명할 수 있습니다.
* 각 raw pointer에 대해 소유자와 해제 위치를 정할 수 있습니다.
* owning pointer와 non-owning pointer를 구분할 수 있습니다.
* 소유권 이전 함수의 성공·실패 규칙을 명확하게 정의할 수 있습니다.
* raw storage와 실제 생성된 객체를 구분할 수 있습니다.
* `size`와 `capacity`가 객체 수명 측면에서 무엇을 의미하는지 설명할 수 있습니다.
* 생성자 실패 시 어떤 소멸자가 호출되고 어떤 자원은 직접 정리해야 하는지 설명할 수 있습니다.
* 컨테이너 삽입이나 복사가 실패하더라도 기존 정상 상태가 손상되지 않도록 상태 변경 순서를 설계할 수 있습니다.
