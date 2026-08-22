# C++98 수명·값·소유권

## 목표

smart pointer와 이동 의미론 없이 heap memory와 file descriptor의 소유자를 명시합니다. 복사, 대입, 소멸 중 자원이 두 번 해제되거나 실패 뒤 기존 값이 사라지지 않게 합니다.

## Rule of Three

소멸자, 복사 생성자, 복사 대입 중 하나를 직접 작성해야 한다면 나머지도 검토합니다.

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

raw pointer가 자원을 소유한다면 compiler 기본 복사는 주소만 복제합니다. 두 객체가 같은 메모리를 삭제하거나 한 객체의 변경이 다른 객체에 보이게 됩니다.

## 깊은 복사

```cpp
TextBuffer::TextBuffer(const TextBuffer &other)
    : data_(new char[other.size_ + 1]), size_(other.size_) {
    std::memcpy(data_, other.data_, size_ + 1);
}
```

복사본이 독립 값이라면 새 메모리를 할당하고 내용까지 복사합니다. 공유가 의도라면 별도 참조 카운트 설계가 필요하지만, 단순 값 타입에서 직접 구현할 이유는 드뭅니다.

## copy-and-swap

대입에서 기존 메모리를 먼저 지우면 새 할당 실패 뒤 값이 사라집니다.

```cpp
TextBuffer &TextBuffer::operator=(const TextBuffer &other) {
    TextBuffer candidate(other);
    swap(candidate);
    return *this;
}
```

후보 복사가 실패하면 현재 객체는 바뀌지 않습니다. 성공한 뒤 pointer와 size를 교환하면 이전 메모리는 `candidate` 소멸자가 정리합니다. 자기 대입도 같은 코드로 처리됩니다.

## 소유 pointer와 관찰 pointer

C++98 raw pointer에는 소유 여부가 타입에 드러나지 않습니다. 클래스와 함수 이름, 문서, 생성·소멸 위치를 통해 명확히 해야 합니다.

```cpp
class Router {
private:
    std::map<std::string, Handler *> handlers_; // Router가 delete합니다.
};
```

소유하지 않는 pointer는 저장 기간이 owner 수명보다 짧아야 합니다.

```cpp
const Handler *Router::find(const std::string &command) const;
```

반환 pointer는 `Router`가 살아 있고 해당 handler를 제거하지 않는 동안만 유효합니다.

## 소유권 이전

이동이 없으므로 pointer 소유권 이전은 특히 위험합니다. 누가 해제하는지 함수 단위로 정합니다.

```cpp
void Router::add(const std::string &name, Handler *handler);
```

이 함수가 성공하면 Router가 소유하고, 실패하면 caller가 소유하는지 함수가 정리하는지 하나로 고정해야 합니다. 둘 다 delete하거나 아무도 delete하지 않는 경로가 없어야 합니다.

더 단순한 방법은 생성과 등록을 같은 함수 안에서 수행해 중간 pointer를 밖으로 내보내지 않는 것입니다.

## 배열과 객체 수명

`new T[n]`은 메모리 할당과 `n`개 객체 생성을 함께 수행합니다. `std::allocator<T>`를 사용하면 메모리 확보와 객체 생성을 나눌 수 있습니다.

```cpp
T *memory = allocator.allocate(capacity);
allocator.construct(memory + index, value);
```

실제로 생성한 원소만 `destroy()`해야 합니다. 할당한 capacity 전체를 파괴하면 아직 존재하지 않는 객체에 소멸자를 호출하게 됩니다.

## 생성자 실패

생성자 본문에서 예외가 나면 해당 객체의 소멸자는 호출되지 않습니다. 생성자가 직접 얻은 raw pointer를 멤버에 저장하기 전에 다른 예외가 발생하지 않도록 순서를 정합니다.

```cpp
TextBuffer::TextBuffer(const char *text)
    : data_(0), size_(length(text)) {
    data_ = new char[size_ + 1];
    copy_text(data_, text, size_);
}
```

`new`가 실패하면 `data_`는 여전히 `0`이고 살아 있는 객체로 계산하지 않습니다.

## 컨테이너와 값 타입

`std::map<Key, Value>`에 넣는 `Value`는 복사 생성과 대입이 올바르게 동작해야 합니다. 컨테이너 내부에서 언제 몇 번 복사되는지에 의존하지 않습니다.

삽입 중 `Value` 복사나 node 할당이 실패할 수 있으므로, container가 제공하는 예외 보장과 자신의 상태 변경 순서를 함께 확인합니다.

## 자주 놓치는 문제

- raw owning pointer를 가진 타입에 기본 복사를 사용합니다.
- 대입에서 현재 자원을 먼저 삭제합니다.
- 소유권 이전 함수의 실패 경로에서 caller와 callee가 모두 delete합니다.
- allocator로 확보한 메모리 전체에 객체가 있다고 가정합니다.
- 생성자 실패도 소멸자가 정리해 줄 것이라고 생각합니다.
- 컨테이너에 넣은 뒤 외부 pointer가 계속 같은 주소를 가리킨다고 가정합니다.

## 완료 기준

- Rule of Three가 필요한 이유를 설명합니다.
- 깊은 복사와 공유 소유를 구분합니다.
- copy-and-swap으로 할당 실패 뒤 기존 값을 보존합니다.
- raw pointer마다 소유자와 해제 위치를 지정합니다.
- raw storage와 생성된 객체 범위를 구분합니다.
- 생성·삽입 실패 뒤 살아 있는 객체와 자원 수를 검사합니다.
