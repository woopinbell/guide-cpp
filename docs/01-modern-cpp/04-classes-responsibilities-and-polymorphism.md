# 클래스·역할 분리·다형성

## 목표

클래스를 단순히 관련 함수의 묶음으로 만들지 않습니다. 어떤 상태를 누가 보유하고, 입력을 누가 해석하며, 실제 상태 변경을 누가 수행하는지 코드에서 찾을 수 있게 나눕니다.

## 상태를 가진 타입부터 정합니다

```cpp
class Store {
public:
    void put(Key key, Value value);
    std::optional<Value> find(const Key& key) const;

private:
    std::map<Key, Value> data_;
};
```

`Store`는 데이터를 보유하고 저장 규칙을 지킵니다. command-line parsing이나 출력 문자열까지 맡길 필요는 없습니다.

좋은 분리는 파일 수가 많은지가 아니라 변경 이유가 분명한지로 판단합니다.

- 입력 문법이 바뀌면 parser를 수정합니다.
- 저장 제한이 바뀌면 store를 수정합니다.
- 출력 형식이 바뀌면 formatter를 수정합니다.
- 실행 방법이 바뀌면 `main` 또는 adapter를 수정합니다.

## 값 타입과 작업 수행 객체

값 타입은 주로 데이터와 유효 조건을 보유합니다.

```cpp
class Port {
public:
    explicit Port(unsigned short value);
    unsigned short value() const noexcept;
};
```

작업 수행 객체는 외부 자원이나 다른 상태를 사용해 동작합니다.

```cpp
class Server {
public:
    explicit Server(Poller& poller);
    void run();
};
```

두 성격을 한 클래스에 억지로 합치면 복사 의미와 수명 관리가 불분명해집니다.

## 생성자에서 유효 상태를 만듭니다

객체가 만들어진 뒤 별도의 `init()`을 반드시 호출해야 한다면 잘못된 상태가 존재합니다.

```cpp
class WorkerPool {
public:
    WorkerPool(std::size_t threads, std::size_t queue_capacity);
};
```

필수 값은 생성자로 받고, 만들 수 없으면 생성이 실패하게 합니다. 선택 기능이나 다시 시도할 수 있는 외부 연결은 별도 연산으로 둘 수 있습니다.

## composition을 먼저 고려합니다

```cpp
class CommandService {
public:
    CommandService(Store& store, Formatter& formatter);
};
```

`CommandService`가 `Store`의 한 종류가 아니라 `Store`를 사용한다면 상속보다 composition이 맞습니다. 상속은 기반 타입으로 사용해도 의미가 자연스러운 경우에 사용합니다.

## 다형성이 필요한 경우

실행 중 여러 구현을 같은 방식으로 호출해야 한다면 virtual function을 사용할 수 있습니다.

```cpp
class Handler {
public:
    virtual ~Handler() = default;
    virtual Response handle(const Request&) = 0;
};
```

기반 클래스 pointer로 삭제할 수 있다면 virtual destructor가 필요합니다. 그렇지 않으면 파생 클래스 소멸자가 호출되지 않아 자원이 남을 수 있습니다.

다형성을 쓰지 않아도 되는 경우도 많습니다.

- 가능한 타입이 닫혀 있고 값으로 처리할 수 있으면 `std::variant`
- compile-time 교체면 template
- 단순 callback이면 `std::function` 또는 lambda
- 구현 하나만 필요하면 구체 타입

## 의존성의 수명

참조 멤버로 외부 객체를 받으면 현재 객체보다 오래 살아야 합니다.

```cpp
class Service {
public:
    explicit Service(Store& store) : store_(store) {}
private:
    Store& store_;
};
```

이 조건을 만족시키는 곳은 보통 `main` 같은 조립 위치입니다. `Store`를 먼저 만들고 `Service`를 나중에 만들면 역순으로 소멸되어 참조가 안전합니다.

소유해야 한다면 값이나 `unique_ptr`로 받습니다. 단순히 null 가능성이 필요하다는 이유만으로 raw owning pointer를 쓰지 않습니다.

## 큰 클래스가 보내는 신호

다음이 함께 보이면 나눌 지점을 검토합니다.

- parser 상태와 업무 데이터를 동시에 보유합니다.
- file descriptor, thread, format 문자열까지 한 클래스가 관리합니다.
- private 멤버 일부만 사용하는 함수 묶음이 여러 개 있습니다.
- 테스트 하나를 만들기 위해 filesystem과 network를 모두 준비해야 합니다.
- 한 변경이 관계없는 함수까지 자주 건드립니다.

하지만 작은 클래스로 무조건 쪼개는 것도 좋지 않습니다. 함께 지켜야 하는 불변식을 여러 객체로 흩어 놓으면 상태 변경 순서를 이해하기 더 어려워질 수 있습니다.

## 자주 놓치는 문제

- 이름이 비슷하다는 이유로 상속합니다.
- 기반 클래스 소멸자를 virtual로 만들지 않고 pointer로 삭제합니다.
- 값 타입을 기반 클래스 값으로 복사해 slicing이 발생합니다.
- 생성자 안에서 virtual function을 호출해 파생 동작을 기대합니다.
- callback이나 참조가 소유 객체보다 오래 남습니다.
- 모든 함수를 하나의 `Manager`에 넣습니다.

## 완료 기준

- 각 상태의 실제 소유자를 말할 수 있습니다.
- 입력 해석, 상태 변경, 출력 형식을 별도 코드에서 검사할 수 있습니다.
- composition과 inheritance를 사용 이유로 구분합니다.
- virtual destructor와 object slicing의 위험을 설명합니다.
- 객체를 만드는 순서와 참조 멤버의 수명 조건을 연결합니다.
