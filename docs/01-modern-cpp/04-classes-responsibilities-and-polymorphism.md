# 클래스·역할 분리·다형성

## 목표

클래스를 단순히 "관련 있어 보이는 함수의 묶음"으로 만들지 않습니다.

다음 질문에 코드만 보고 답할 수 있도록 구성합니다.

* 이 상태를 실제로 소유하는 객체는 무엇입니까?
* 입력 문자열을 해석하는 코드는 어디에 있습니까?
* 상태를 변경하는 규칙은 어디에 있습니까?
* 출력 형식을 결정하는 코드는 어디에 있습니까?
* 이 객체가 사용하는 다른 객체를 소유합니까, 아니면 빌려 사용합니까?
* 상속과 runtime 다형성이 실제로 필요한 이유가 있습니까?

## 상태를 가진 타입부터 정합니다

예를 들어 key-value 데이터를 관리한다고 합시다.

```cpp
class Store {
public:
    void put(Key key, Value value);
    std::optional<Value> find(const Key& key) const;

private:
    std::map<Key, Value> data_;
};
```

`Store`가 담당할 핵심은 저장된 데이터와 그 데이터를 변경하는 규칙입니다.

예를 들어 다음과 같은 규칙이 있다면 `Store`와 밀접합니다.

* 같은 key를 덮어쓸 수 있는가?
* 최대 항목 수가 있는가?
* 삭제되지 않는 항목이 있는가?
* 존재하지 않는 key 조회를 어떻게 표현하는가?

반면 다음은 별개의 이유로 바뀔 수 있습니다.

* command-line 문자열 문법
* JSON 출력 형식
* 터미널 색상
* 파일에서 command를 읽는 방법

따라서 `Store`가 다음까지 모두 할 필요는 없습니다.

```cpp
store.parse_command_line(...);
store.print_json(...);
store.read_file(...);
```

좋은 분리는 단순히 클래스나 파일 수가 많은가로 판단하지 않습니다.

**서로 다른 이유로 변경되는 코드가 불필요하게 하나에 묶여 있는지**를 봅니다.

예:

```text
명령 문법 변경
→ Parser

저장 규칙 변경
→ Store

출력 형식 변경
→ Formatter

프로그램 실행 방식 변경
→ main / adapter
```

이것은 무조건 하나씩 별도 클래스를 만들어야 한다는 뜻은 아닙니다.

작은 함수 하나로 충분하다면 별도 class 없이 함수로 둘 수도 있습니다.

## 상태와 불변식

클래스가 상태를 소유하는 가장 중요한 이유 중 하나는 **유효한 상태를 유지하는 규칙을 한곳에서 보장할 수 있기 때문**입니다.

예를 들어 port 번호를 표현합니다.

```cpp
class Port {
public:
    explicit Port(unsigned short value)
        : value_(value) {}

    unsigned short value() const noexcept {
        return value_;
    }

private:
    unsigned short value_;
};
```

만약 허용 범위가 따로 있다면 생성 시 검사할 수 있습니다.

```cpp
class Port {
public:
    explicit Port(unsigned int value) {
        if (value == 0 || value > 65535) {
            throw std::invalid_argument{"invalid port"};
        }

        value_ = static_cast<unsigned short>(value);
    }

    unsigned short value() const noexcept {
        return value_;
    }

private:
    unsigned short value_;
};
```

이제 성공적으로 생성된 `Port`는 항상 유효한 값이라는 조건을 가질 수 있습니다.

즉 사용자가 매번 다음 검사를 반복할 필요가 없습니다.

```cpp
if (port > 0 && port <= 65535) {
    // ...
}
```

## 값 타입과 작업 수행 객체

모든 클래스의 성격이 같지는 않습니다.

### 값 타입

값 타입은 주로 하나의 값이나 상태를 표현합니다.

예:

```cpp
Port port{8080};
TaskId id{42};
Money price{1000};
```

일반적으로 다음 성질이 자연스럽습니다.

* 복사가 의미 있음
* 두 값의 비교가 의미 있음
* 다른 객체와 독립적으로 존재함
* 자원 identity보다 값 자체가 중요함

### 작업 수행 객체

다른 객체나 외부 자원을 사용하여 작업을 수행하는 객체도 있습니다.

```cpp
class Server {
public:
    explicit Server(Poller& poller)
        : poller_(poller) {}

    void run();

private:
    Poller& poller_;
};
```

`Server`는 단순한 값이라기보다 실행 동작과 외부 dependency를 가진 객체입니다.

다음과 같은 자원을 사용할 수도 있습니다.

* socket
* event poller
* thread pool
* database connection
* filesystem

이런 객체에서는 일반적인 값 타입처럼 "복사하면 동일한 서버 하나가 더 생긴다"는 의미가 자연스럽지 않을 수 있습니다.

따라서 값 타입과 자원/작업 객체를 무리하게 같은 복사·수명 모델로 만들지 않습니다.

## 생성이 끝났다면 유효한 객체여야 합니다

다음 API를 생각할 수 있습니다.

```cpp
WorkerPool pool;
pool.init(4, 100);
```

`init()` 호출을 빠뜨릴 수 있다면 다음 상태가 존재합니다.

```text
객체는 존재하지만 아직 사용할 수 없음
```

그 결과 모든 member function에서 초기화 여부를 확인해야 할 수도 있습니다.

```cpp
if (!initialized_) {
    // error
}
```

필수 조건은 생성자에서 받는 편이 더 명확할 수 있습니다.

```cpp
class WorkerPool {
public:
    WorkerPool(
        std::size_t threads,
        std::size_t queue_capacity
    );
};
```

성공적으로 생성된 객체는 바로 사용할 수 있습니다.

```cpp
WorkerPool pool{4, 100};
pool.submit(...);
```

만들 수 없다면 생성 자체가 실패합니다.

다만 모든 초기화를 생성자에서 해야 한다는 뜻은 아닙니다.

예를 들어 다음은 별도의 작업일 수 있습니다.

* 실패 후 다시 연결 가능한 network connection
* 사용자 요청으로 시작하는 background task
* 비용이 큰 lazy initialization
* 객체 생성 후 선택적으로 활성화하는 기능

핵심은 **어떤 상태가 객체가 존재하기 위한 필수 조건인지**를 구분하는 것입니다.

## composition을 먼저 고려합니다

다음 객체는 `Store`와 `Formatter`를 사용합니다.

```cpp
class CommandService {
public:
    CommandService(Store& store, Formatter& formatter)
        : store_(store),
          formatter_(formatter) {}

private:
    Store& store_;
    Formatter& formatter_;
};
```

관계는 다음과 같습니다.

```text
CommandService
   ├── uses → Store
   └── uses → Formatter
```

`CommandService`는 `Store`의 한 종류가 아닙니다.

따라서 다음처럼 상속하는 것은 자연스럽지 않습니다.

```cpp
class CommandService : public Store {
};
```

composition은 "A가 B를 가지고 있거나 사용한다"는 관계에 적합합니다.

```text
has-a
uses-a
```

상속은 "A를 B로 취급해도 의미가 자연스럽다"는 관계에서 고려합니다.

```text
is-a
```

하지만 이름만 `is-a`처럼 보인다고 충분한 것은 아닙니다.

파생 객체를 기반 타입이 요구되는 위치에 넣어도 기반 타입의 의미와 규칙을 깨뜨리지 않아야 합니다.

## 상속을 사용하는 이유

상속은 단순한 코드 재사용 도구로 먼저 생각하지 않습니다.

특히 public inheritance는 **파생 타입을 기반 타입으로 사용할 수 있다는 의미**를 갖습니다.

예:

```cpp
class Handler {
public:
    virtual ~Handler() = default;

    virtual Response handle(const Request& request) = 0;
};
```

구현:

```cpp
class FileHandler : public Handler {
public:
    Response handle(const Request& request) override;
};

class NetworkHandler : public Handler {
public:
    Response handle(const Request& request) override;
};
```

호출자는 실제 구현 종류를 몰라도 `Handler`를 통해 사용할 수 있습니다.

```cpp
void process(Handler& handler, const Request& request) {
    Response response = handler.handle(request);
}
```

이 경우 runtime에 어떤 구현이 전달될지 바뀔 수 있다는 것이 다형성 사용의 실제 이유입니다.

## virtual destructor가 필요한 이유

다음처럼 기반 클래스 pointer를 통해 파생 객체를 삭제한다고 합시다.

```cpp
Handler* handler = new FileHandler;

delete handler;
```

`Handler`의 destructor가 virtual이 아니면 기반 pointer를 통한 삭제는 올바르게 파생 객체 전체를 파괴할 수 없으며 undefined behavior가 됩니다.

따라서 **polymorphic base를 기반 pointer/reference를 통해 소유·삭제할 가능성이 있다면 virtual destructor가 필요합니다.**

```cpp
class Handler {
public:
    virtual ~Handler() = default;

    virtual Response handle(const Request&) = 0;
};
```

실제 코드에서는 raw owning pointer보다 다음과 같이 사용하는 편이 일반적입니다.

```cpp
std::unique_ptr<Handler> handler =
    std::make_unique<FileHandler>();
```

`unique_ptr<Handler>`이 파괴될 때 `Handler`의 virtual destructor를 통해 `FileHandler`까지 올바르게 파괴됩니다.

반대로 모든 기반 클래스가 무조건 virtual destructor를 가져야 하는 것은 아닙니다.

애초에 polymorphic base로 사용할 의도가 없고 기반 pointer를 통한 삭제를 허용하지 않는 타입이라면 필요하지 않을 수 있습니다.

## object slicing

다형적 타입을 값으로 복사하면 파생 부분이 잘릴 수 있습니다.

```cpp
class Animal {
public:
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    std::string name;
};

Dog dog;
dog.name = "Milo";

Animal animal = dog;
```

`animal`은 새로운 `Animal` 객체입니다.

`Dog`의 `name` 부분은 복사 대상에 존재하지 않습니다.

개념적으로:

```text
Dog
┌─────────────┐
│ Animal part │
├─────────────┤
│ Dog part    │
└─────────────┘

      value copy

Animal
┌─────────────┐
│ Animal part │
└─────────────┘
```

이것이 object slicing입니다.

runtime 다형성을 유지하려면 보통 reference나 pointer를 사용합니다.

```cpp
Animal& animal = dog;
```

또는:

```cpp
std::unique_ptr<Animal> animal =
    std::make_unique<Dog>();
```

## 생성자와 virtual dispatch

생성자에서 virtual function을 호출하더라도 아직 완성되지 않은 파생 객체의 override가 호출된다고 기대하면 안 됩니다.

예:

```cpp
class Base {
public:
    Base() {
        initialize();
    }

    virtual void initialize() {
        // Base 구현
    }
};

class Derived : public Base {
public:
    void initialize() override {
        // Derived 구현
    }
};
```

`Derived`를 생성할 때 먼저 `Base` 부분이 생성됩니다.

그동안 `Derived` 부분은 아직 완전히 생성되지 않았습니다.

따라서 `Base` 생성자 안에서의 virtual 호출은 `Derived::initialize()`로 dispatch되지 않습니다.

소멸 중에도 비슷한 원칙이 적용됩니다. 파생 부분이 이미 파괴된 뒤 기반 destructor에서 파생 동작을 기대할 수 없습니다.

따라서 생성자와 소멸자에서 virtual dispatch를 이용해 파생 객체의 완성된 상태를 사용하는 설계는 피합니다.

## virtual function이 항상 필요한 것은 아닙니다

"구현이 여러 개"라는 이유만으로 runtime inheritance가 반드시 필요한 것은 아닙니다.

### 가능한 타입 집합이 닫혀 있다면 `std::variant`

예:

```cpp
using Command = std::variant<
    AddCommand,
    RemoveCommand,
    ListCommand
>;
```

가능한 종류가 고정되어 있고 값으로 처리할 수 있다면 적합할 수 있습니다.

### compile-time 교체라면 template

```cpp
template <typename Formatter>
void render(const Data& data, Formatter& formatter);
```

실행 중 implementation을 교체할 필요가 없다면 virtual dispatch가 필요하지 않을 수 있습니다.

### 동작 하나만 전달한다면 lambda

```cpp
run_job([] {
    // 작업
});
```

또는 필요한 경우:

```cpp
std::function<void()> callback;
```

단순 callback을 위해 base class hierarchy를 만들 필요는 없습니다.

### 구현이 하나뿐이라면 구체 타입

```cpp
FileStore store;
```

현재 실제 요구사항에 교체 가능한 구현이 없다면 불필요한 abstraction을 미리 만들 필요도 없습니다.

## dependency의 소유와 수명

다음 클래스는 `Store`를 소유하지 않습니다.

```cpp
class Service {
public:
    explicit Service(Store& store)
        : store_(store) {}

private:
    Store& store_;
};
```

`store_`는 다른 객체를 빌려 사용합니다.

따라서 필수 조건은 다음과 같습니다.

```text
Store의 수명 > Service가 Store를 사용하는 기간
```

다음 local variable 배치는 안전합니다.

```cpp
int main() {
    Store store;
    Service service{store};

    // ...
}
```

local variable은 생성의 역순으로 소멸됩니다.

```text
생성

Store
↓
Service

소멸

Service
↓
Store
```

따라서 `Service`가 파괴될 때까지 `Store`가 살아 있습니다.

반대로 다음은 잘못될 수 있습니다.

```cpp
Service make_service() {
    Store store;
    return Service{store};
}
```

함수가 끝나면 `store`가 파괴되고 반환된 `Service`의 reference member는 dangling reference가 됩니다.

## member의 생성·소멸 순서도 중요합니다

클래스 내부 member도 수명 순서를 가집니다.

```cpp
class Application {
private:
    Store store_;
    Service service_;
};
```

member는 **선언된 순서대로 생성되고 역순으로 소멸됩니다.**

따라서 다음처럼 생성할 수 있습니다.

```cpp
Application::Application()
    : store_{},
      service_{store_} {}
```

생성:

```text
store_
↓
service_
```

소멸:

```text
service_
↓
store_
```

따라서 `service_`가 `store_`를 reference로 사용하는 동안 `store_`가 살아 있습니다.

중요한 점은 initializer list에 적힌 순서가 아니라 **클래스에서 member를 선언한 순서가 실제 생성 순서를 결정한다는 것**입니다.

다음처럼 작성해도:

```cpp
Application::Application()
    : service_{store_},
      store_{} {}
```

선언이

```cpp
Store store_;
Service service_;
```

라면 실제로는 `store_`가 먼저 생성됩니다.

혼동을 피하려면 initializer list도 선언 순서와 동일하게 작성합니다.

## dependency를 소유해야 한다면

객체가 dependency의 생존 자체를 책임져야 한다면 reference가 아니라 ownership을 표현해야 합니다.

값으로 충분하다면:

```cpp
class Service {
private:
    Store store_;
};
```

runtime에 implementation을 선택하면서 유일 소유해야 한다면:

```cpp
class Service {
public:
    explicit Service(std::unique_ptr<Store> store)
        : store_(std::move(store)) {}

private:
    std::unique_ptr<Store> store_;
};
```

핵심은 pointer 형태가 아니라 실제 의미입니다.

```text
reference / pointer
→ 보통 non-owning 사용을 표현

value / unique_ptr
→ ownership 표현

shared_ptr
→ 실제 공유 ownership 표현
```

"null일 수 있다"는 이유만으로 raw owning pointer를 사용할 필요는 없습니다.

## 큰 클래스가 보내는 신호

큰 클래스 자체가 항상 잘못된 것은 아닙니다.

하지만 다음이 함께 나타난다면 서로 다른 역할이 섞였는지 검토할 가치가 있습니다.

* parser 상태와 업무 상태를 동시에 보유합니다.
* file descriptor, thread, protocol parsing, 출력 formatting을 모두 관리합니다.
* private member 일부만 사용하는 method 집단이 명확히 나뉩니다.
* parser 변경 때문에 network 관련 코드까지 자주 수정합니다.
* 테스트 하나를 실행하려면 filesystem, network, thread를 모두 준비해야 합니다.
* 한 기능 변경이 관계없는 여러 member function에 반복적으로 영향을 줍니다.

예를 들어:

```text
ServerManager
├── socket 생성
├── HTTP parsing
├── 사용자 저장
├── JSON 생성
├── log 파일 쓰기
├── thread 시작
└── config 파일 해석
```

라면 서로 다른 이유로 변경되는 코드가 하나에 몰렸을 가능성이 높습니다.

하지만 반대로 무조건 작은 클래스로 쪼개는 것도 문제입니다.

하나의 상태 불변식을 함께 지켜야 하는데 여러 객체로 흩어버리면 다음과 같은 코드가 생길 수 있습니다.

```cpp
balanceStore.withdraw(...);
historyStore.record(...);
limitStore.update(...);
```

세 작업이 반드시 하나의 상태 변경으로 함께 성공해야 한다면 무리한 분리가 오히려 상태 일관성을 이해하기 어렵게 만들 수 있습니다.

따라서 분리 기준은 단순히 줄 수나 함수 수가 아닙니다.

```text
이 상태들을 항상 함께 변경해야 하는가?
이 코드들은 같은 이유로 변경되는가?
독립적으로 테스트할 실제 의미가 있는가?
```

를 봅니다.

## `Manager`라는 이름을 경계하는 이유

다음과 같은 클래스 이름은 역할을 설명하지 못하는 경우가 많습니다.

```text
DataManager
SystemManager
TaskManager
NetworkManager
```

이름 자체가 잘못된 것은 아니지만, "무엇을 하는 객체인지" 대신 "여러 가지를 관리한다"는 말로 책임을 뭉뚱그릴 수 있습니다.

예를 들어 실제 역할이 저장이라면:

```text
TaskStore
```

명령 해석이라면:

```text
CommandParser
```

실행 조정이라면:

```text
TaskScheduler
```

처럼 구체적인 역할을 드러내는 이름이 코드를 읽기 쉽습니다.

## 자주 놓치는 문제

* 이름이 비슷하거나 코드를 재사용하고 싶다는 이유만으로 public inheritance를 사용합니다.
* polymorphic base를 기반 pointer로 삭제하면서 destructor를 virtual로 만들지 않습니다.
* 파생 객체를 기반 객체 값으로 복사해 object slicing을 만듭니다.
* 생성자에서 virtual function을 호출하고 파생 override가 실행될 것이라고 기대합니다.
* non-owning reference나 callback이 실제 owner보다 오래 살아남습니다.
* reference member가 있다는 사실만 보고 source object의 실제 수명을 확인하지 않습니다.
* member declaration order와 initializer list 순서를 혼동합니다.
* 단순 callback 하나를 위해 불필요한 inheritance hierarchy를 만듭니다.
* 실제 공유 소유가 아닌데 dependency를 `shared_ptr`로 전달합니다.
* 서로 다른 변경 이유의 코드를 모두 하나의 `Manager` 클래스에 넣습니다.
* 반대로 하나의 불변식을 함께 지켜야 하는 상태를 지나치게 여러 클래스로 분리합니다.

## 코드를 읽을 때 확인할 질문

클래스를 보면 다음 순서로 확인합니다.

1. 이 객체가 직접 소유하는 상태는 무엇입니까?
2. 어떤 상태의 유효 조건을 이 클래스가 보장합니까?
3. dependency는 소유합니까, 빌려 사용합니까?
4. 빌린다면 dependency가 충분히 오래 살아 있다는 보장은 어디에 있습니까?
5. 입력 해석과 실제 상태 변경이 불필요하게 섞여 있습니까?
6. 상속은 substitutability 때문에 필요한 것입니까, 단순 코드 재사용 때문입니까?
7. runtime 다형성이 실제로 필요합니까?
8. `variant`, template, lambda, 구체 타입으로 더 단순하게 표현할 수 있습니까?
9. polymorphic base를 기반 pointer로 삭제할 수 있다면 destructor가 virtual입니까?
10. 파생 객체가 값으로 기반 타입에 복사되어 slicing될 가능성은 없습니까?
11. 생성자와 소멸자에서 파생 virtual 동작을 기대하고 있지 않습니까?
12. 클래스를 나눴을 때 하나의 불변식이 여러 객체 사이로 불필요하게 흩어지지는 않습니까?

## 완료 기준

* 각 상태의 실제 소유자를 설명합니다.
* 값 타입과 자원·작업 수행 객체의 성격 차이를 설명합니다.
* 성공적으로 생성된 객체가 어떤 유효 조건을 만족하는지 설명합니다.
* 입력 해석, 상태 변경, 출력 형식을 필요에 따라 독립적으로 검사할 수 있습니다.
* composition과 public inheritance를 관계의 의미로 구분합니다.
* runtime polymorphism이 실제로 필요한 경우와 대안을 구분합니다.
* virtual destructor가 필요한 조건을 설명합니다.
* object slicing이 무엇이며 reference/pointer 사용과 어떤 차이가 있는지 설명합니다.
* 생성자·소멸자에서 virtual dispatch가 제한되는 이유를 설명합니다.
* non-owning reference의 수명 조건을 실제 객체 생성·소멸 순서와 연결합니다.
* member의 생성 순서가 선언 순서로 결정된다는 것을 설명합니다.
* 큰 클래스를 단순 크기가 아니라 변경 이유와 상태 불변식을 기준으로 분리할지 판단합니다.
