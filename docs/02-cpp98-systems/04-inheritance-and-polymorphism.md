# C++98 상속과 다형성

## 목표

상속을 단순한 코드 재사용 수단으로 사용하지 않습니다.

기반 타입의 pointer나 reference를 통해 서로 다른 구현을 같은 방식으로 호출해야 할 때 runtime polymorphism을 사용합니다.

동시에 다음 문제도 함께 고려합니다.

* virtual 함수 호출
* override signature
* 기반 pointer를 통한 삭제
* 객체 slicing
* 복사 여부
* raw pointer 소유권
* handler 등록 실패
* 이름 가리기

## virtual dispatch

다음과 같은 기반 타입이 있다고 가정합니다.

```cpp
class Handler {
public:
    virtual ~Handler() {
    }

    virtual Response handle(
        const Request &request,
        Store &store
    ) const = 0;
};
```

`handle()`은 pure virtual 함수이므로 `Handler`는 추상 클래스입니다.

다음 파생 타입이 이를 구현할 수 있습니다.

```cpp
class PutHandler : public Handler {
public:
    Response handle(
        const Request &request,
        Store &store
    ) const;
};
```

그리고 다음처럼 호출합니다.

```cpp
Handler *handler = new PutHandler;

Response response =
    handler->handle(request, store);
```

변수의 정적 타입은

```cpp
Handler *
```

이지만 실제 객체는

```cpp
PutHandler
```

입니다.

`handle()`이 virtual이므로 실행 시 실제 객체 타입을 기준으로 `PutHandler::handle()`이 호출됩니다.

이를 virtual dispatch라고 합니다.

## virtual 함수와 overload는 다른 개념입니다

다음은 overload입니다.

```cpp
void handle(int value);
void handle(const std::string &value);
```

같은 이름의 함수가 서로 다른 인자 목록을 가지며 compile 시 호출할 함수를 선택합니다.

반면 다음은 override 관계입니다.

```cpp
class Base {
public:
    virtual void handle(int value);
};

class Derived : public Base {
public:
    virtual void handle(int value);
};
```

기반 클래스의 virtual 함수를 파생 클래스가 같은 함수 타입으로 다시 정의합니다.

기반 pointer를 통해 호출했을 때 실제 객체에 맞는 구현이 선택됩니다.

## C++98에는 `override`가 없습니다

Modern C++에서는 다음처럼 작성할 수 있습니다.

```cpp
Response handle(...) const override;
```

하지만 C++98에는 `override`가 없습니다.

따라서 signature를 실수로 변경해도 컴파일러가 의도를 직접 확인해 주지 못합니다.

예:

```cpp
class Handler {
public:
    virtual Response handle(
        const Request &request,
        Store &store
    ) const = 0;
};
```

그런데 파생 클래스에서 뒤의 `const`를 빠뜨렸다고 가정합니다.

```cpp
class PutHandler : public Handler {
public:
    Response handle(
        const Request &request,
        Store &store
    );
};
```

이 함수는 기반 virtual 함수의 override가 아닙니다.

그리고 `Handler`의 pure virtual 함수가 아직 구현되지 않았으므로 `PutHandler` 역시 추상 클래스입니다.

따라서 다음 객체 생성은 실패합니다.

```cpp
PutHandler handler;
```

기반 함수가 pure virtual이 아닌 경우에는 이런 실수가 더 눈에 띄지 않을 수 있습니다.

따라서 C++98에서는 다음을 주의해서 확인합니다.

* `const`
* parameter의 reference 여부
* pointer 여부
* parameter 타입
* 함수 이름

반환 타입은 일부 공변 반환(covariant return) 예외를 제외하면 임의로 바꿀 수 없습니다.

## 기반 클래스에서도 `virtual`을 명시합니다

파생 클래스에서 override된 함수에 `virtual`을 다시 쓰지 않아도 virtual 성질은 유지됩니다.

즉 다음도 동작합니다.

```cpp
class PutHandler : public Handler {
public:
    Response handle(
        const Request &request,
        Store &store
    ) const;
};
```

하지만 C++98에서는 `override`를 사용할 수 없기 때문에 의도를 눈에 띄게 하기 위해 파생 클래스에서도 `virtual`을 반복해서 쓰는 코드 스타일도 있습니다.

```cpp
virtual Response handle(...) const;
```

이는 언어적으로 필수는 아닙니다.

## virtual destructor

다형 객체를 기반 pointer를 통해 삭제할 수 있다면 기반 클래스 소멸자는 virtual이어야 합니다.

```cpp
Handler *handler = new PutHandler;

delete handler;
```

올바른 형태:

```cpp
class Handler {
public:
    virtual ~Handler() {
    }
};
```

그러면 소멸 순서는 대략 다음과 같습니다.

```text
PutHandler::~PutHandler()
→ Handler::~Handler()
```

### non-virtual destructor일 때 결과를 정확히 이해해야 합니다

다음처럼 기반 소멸자가 virtual이 아닌 상태에서

```cpp
Base *base = new Derived;
delete base;
```

삭제하면 단순히

> 파생 소멸자가 호출되지 않는다

정도로 끝나는 것이 아닙니다.

이 동작은 **undefined behavior**입니다.

따라서 프로그램의 결과를 보장할 수 없습니다.

"Derived 자원만 누수된다" 정도로 단정해서는 안 됩니다.

## 언제 virtual destructor가 필요한가

다음 조건이라면 필요합니다.

```text
Base* 또는 Base&로 객체를 다룸
+
Base*를 통해 delete할 수 있음
```

반대로 기반 pointer를 통한 삭제 자체를 금지하는 설계라면 다른 방법도 있습니다.

예:

```cpp
class Base {
protected:
    ~Base() {
    }
};
```

protected non-virtual destructor를 사용하면 외부에서 다음 코드가 허용되지 않습니다.

```cpp
Base *base = ...;
delete base;
```

하지만 이런 설계는 사용 방법을 명확하게 제한할 때만 적절합니다.

일반적인 polymorphic interface는 virtual destructor를 두는 편이 단순합니다.

## pure virtual class

하나 이상의 pure virtual 함수가 있는 클래스는 직접 객체를 만들 수 없습니다.

```cpp
class Handler {
public:
    virtual ~Handler() {
    }

    virtual Response handle(
        const Request &request,
        Store &store
    ) const = 0;
};
```

다음은 불가능합니다.

```cpp
Handler handler;
```

C++98에는 `interface`라는 별도 언어 키워드가 없습니다.

따라서 pure virtual 함수만 제공하는 클래스를 interface처럼 사용할 수 있습니다.

### 기반 클래스에 상태가 있어도 문법적으로 문제는 없습니다

다음도 가능합니다.

```cpp
class Handler {
protected:
    int count_;
};
```

다만 모든 파생 handler가 정말 같은 상태와 규칙을 공유하는 것이 아니라면 기반 클래스에 억지로 넣지 않는 편이 낫습니다.

상태가 특정 handler에만 필요하다면 그 파생 클래스가 직접 갖는 편이 명확합니다.

## pure virtual destructor에도 정의가 필요할 수 있습니다

소멸자를 pure virtual로 선언할 수도 있습니다.

```cpp
class Base {
public:
    virtual ~Base() = 0;
};
```

하지만 파생 객체가 소멸할 때 기반 클래스 소멸자도 결국 실행되어야 하므로 정의가 필요합니다.

```cpp
Base::~Base() {
}
```

pure virtual이라는 이유만으로 구현이 전혀 필요 없는 것은 아닙니다.

## object slicing

object slicing은 파생 객체를 기반 **값 타입**으로 복사할 때 파생 클래스 부분이 사라지는 현상입니다.

다만 기존 예시는 다음과 같은 문제가 있습니다.

```cpp
void run(Handler handler);
```

앞서 `Handler`가 pure virtual 함수를 가진 추상 클래스라면 애초에 `Handler`를 값으로 받을 수 없습니다.

따라서 slicing 설명에는 추상 클래스가 아닌 예시가 더 정확합니다.

```cpp
class Animal {
public:
    virtual ~Animal() {
    }

    virtual void speak() const {
    }
};

class Dog : public Animal {
private:
    int boneCount_;
};
```

다음 함수가 있다고 가정합니다.

```cpp
void run(Animal animal);
```

그리고:

```cpp
Dog dog;
run(dog);
```

`run()`의 parameter를 만들면서 `Dog` 객체에서 `Animal` 부분만 복사됩니다.

개념적으로:

```text
Dog
┌──────────────────┐
│ Animal 부분      │
│ Dog 전용 부분    │
└──────────────────┘

        ↓ 값 복사

Animal
┌──────────────────┐
│ Animal 부분      │
└──────────────────┘
```

`Dog` 전용 상태는 전달되지 않습니다.

다형성을 유지하려면 reference나 pointer를 사용합니다.

```cpp
void run(Animal &animal);
```

또는:

```cpp
void run(const Animal &animal);
```

## abstract base에서는 slicing보다 "값으로 다룰 수 없음"이 먼저입니다

`Handler`처럼 pure virtual interface라면 다음 자체가 불가능합니다.

```cpp
Handler handler;
```

따라서 handler 계층에서는 일반적으로

```cpp
Handler &
```

또는

```cpp
Handler *
```

로 다루게 됩니다.

slicing 개념은 C++ 전체에서 중요하지만, pure interface에서는 애초에 기반 객체를 값으로 만들 수 없다는 점을 먼저 이해해야 합니다.

## 다형 객체 복사가 필요하다면 `clone()`

다형 객체의 실제 타입을 유지하면서 복사해야 할 때 virtual `clone()` 패턴을 사용할 수 있습니다.

```cpp
class Handler {
public:
    virtual ~Handler() {
    }

    virtual Handler *clone() const = 0;
};
```

파생 클래스:

```cpp
class PutHandler : public Handler {
public:
    virtual PutHandler *clone() const {
        return new PutHandler(*this);
    }
};
```

C++에서는 이런 공변 반환 타입을 허용합니다.

다만 C++98에서 raw pointer를 반환하면 반드시 소유 규칙을 정해야 합니다.

예:

> `clone()`이 반환한 pointer는 caller가 소유하며 `delete`해야 한다.

이 규칙이 없으면 복사가 성공해도 누가 자원을 정리해야 하는지 불분명합니다.

## 이름 가리기와 override를 구분합니다

다음 기반 클래스가 있습니다.

```cpp
class Base {
public:
    void handle(int value);
    void handle(const std::string &value);
};
```

파생 클래스:

```cpp
class Derived : public Base {
public:
    void handle(int value);
};
```

`Derived`에서 `handle`이라는 이름을 선언하면 기반 클래스의 같은 이름 함수들이 name lookup에서 가려집니다.

따라서 다음 호출은 기대와 다르게 실패할 수 있습니다.

```cpp
Derived derived;
derived.handle(std::string("hello"));
```

기반 클래스에는 적절한 overload가 있지만 `Derived::handle`이 이름을 가렸기 때문입니다.

C++98에서는 다음처럼 다시 노출할 수 있습니다.

```cpp
class Derived : public Base {
public:
    using Base::handle;

    void handle(int value);
};
```

### 여기서 중요한 점

이 예제의 `Base::handle()`은 virtual이 아닙니다.

따라서 이것은 override 설명이 아니라 **이름 가리기와 overload set**에 대한 설명입니다.

이 둘을 혼동하면 안 됩니다.

## override와 이름 가리기가 동시에 발생할 수도 있습니다

예:

```cpp
class Base {
public:
    virtual void handle(int value);
    void handle(const std::string &value);
};

class Derived : public Base {
public:
    virtual void handle(int value);
};
```

`Derived::handle(int)`는 virtual override입니다.

동시에 `Derived`의 `handle` 선언 때문에 `Base::handle(std::string)`은 이름 lookup에서 가려질 수 있습니다.

따라서 필요하다면:

```cpp
using Base::handle;
```

를 함께 사용합니다.

## handler 소유권

다음 Router가 있다고 가정합니다.

```cpp
class Router {
private:
    std::map<std::string, Handler *> handlers_;
};
```

이 코드만으로는 pointer를 Router가 소유하는지 알 수 없습니다.

Router가 handler 객체를 소유한다고 정했다면 다음을 모두 처리해야 합니다.

* 정상 소멸
* 중복 등록
* map 삽입 실패
* 일부 handler만 등록된 상태에서 생성 실패
* Router 복사
* Router 대입

## 정상 소멸

Router가 소유한다면 소멸할 때 삭제합니다.

```cpp
Router::~Router() {
    HandlerMap::iterator it = handlers_.begin();

    while (it != handlers_.end()) {
        delete it->second;
        ++it;
    }
}
```

여기서는 `Handler`의 virtual destructor가 중요합니다.

각 pointer가 실제로 `PutHandler`, `GetHandler` 등을 가리켜도 올바른 파생 소멸자가 호출되어야 하기 때문입니다.

## 등록 실패와 소유권

다음 함수는 의미가 불분명합니다.

```cpp
void Router::add(
    const std::string &name,
    Handler *handler
);
```

최소한 다음을 정해야 합니다.

```text
함수 호출 전 소유자
성공 후 소유자
실패 후 소유자
```

예를 들어:

> 호출 전에는 caller가 소유한다. 성공하면 Router가 소유한다. 실패하면 caller가 계속 소유한다.

라고 정할 수 있습니다.

이 경우 Router는 실패했을 때 임의로 pointer를 삭제해서는 안 됩니다.

반대로

> 함수가 pointer를 받는 즉시 Router가 소유한다.

라고 정하면 실패 시 Router가 정리해야 합니다.

둘 중 하나로 일관되게 정해야 합니다.

## container 삽입 자체도 실패할 수 있습니다

다음 코드도 실패 가능성을 생각해야 합니다.

```cpp
Handler *handler = new PutHandler;

handlers_.insert(
    std::make_pair("PUT", handler)
);
```

`new PutHandler`는 성공했지만 `map::insert()`가 node 할당 중 예외를 던질 수 있습니다.

그 경우 `handler`는 아직 map에 들어가지 않았으므로 그대로 두면 누수됩니다.

따라서

```text
객체 생성 성공
→ container 삽입 실패
```

경로도 반드시 처리해야 합니다.

C++98에는 `std::auto_ptr`가 존재하므로 제한적인 상황에서 임시 소유자 역할로 사용할 수도 있습니다. 다만 `auto_ptr`는 복사 시 소유권을 이전하는 특이한 의미를 가지므로 표준 컨테이너 값 타입처럼 일반적으로 사용해서는 안 됩니다.

직접 raw pointer를 사용한다면 예외 발생 전후의 소유자를 명시적으로 관리해야 합니다.

## Router 복사

다음 상태를 생각해 봅니다.

```text
Router A
handlers_["PUT"] ──► PutHandler
```

compiler 기본 복사를 사용하면:

```text
Router A ──┐
           ├──► 같은 PutHandler
Router B ──┘
```

가 될 수 있습니다.

두 Router가 모두 해당 pointer를 소유한다고 생각하면 소멸할 때 같은 객체를 두 번 `delete`하게 됩니다.

따라서 복사가 필요하지 않다면 C++98에서는 복사를 막을 수 있습니다.

```cpp
class Router {
public:
    Router();
    ~Router();

private:
    Router(const Router &);
    Router &operator=(const Router &);
};
```

정의는 제공하지 않습니다.

외부 코드가 복사를 시도하면 private 접근 때문에 compile 오류가 발생합니다.

C++11 이후의

```cpp
Router(const Router &) = delete;
```

와 같은 문법은 C++98에서 사용할 수 없습니다.

## 복사가 필요하다면 의미를 직접 정의해야 합니다

복사를 허용한다면 선택해야 합니다.

### handler까지 깊은 복사

각 handler에 `clone()` 같은 기능을 제공해 Router마다 독립 객체를 만듭니다.

### handler는 공유하고 별도 소유자를 둠

Router가 실제 소유자가 아니도록 설계를 변경합니다.

### Router 자체를 복사 불가능하게 유지

많은 경우 가장 단순한 선택입니다.

어떤 방법이 맞는지는 Router의 의미에 따라 달라집니다.

## composition이 더 나은 경우

다음 관계를 생각해 봅니다.

```text
Server가 Poller를 사용한다.
Service가 Store를 사용한다.
Connection이 Parser를 사용한다.
```

이 관계는 일반적으로 상속 관계가 아닙니다.

예:

```cpp
class Service {
private:
    Store &store_;
};
```

또는:

```cpp
class Connection {
private:
    Parser parser_;
};
```

이처럼 한 객체가 다른 객체를 **사용하거나 보유한다면 composition 또는 참조 관계**가 자연스럽습니다.

상속은 다음과 같은 관계를 표현할 때 검토합니다.

```text
PutHandler는 Handler로 취급할 수 있다.
GetHandler는 Handler로 취급할 수 있다.
```

즉 호출 코드가

```cpp
Handler &
```

를 요구하는 곳에 `PutHandler`를 전달해도 의미가 자연스러워야 합니다.

## "is-a"만으로 충분하지는 않습니다

흔히 상속을 "`is-a` 관계"라고 설명하지만 그것만으로 판단하면 모호합니다.

더 실용적으로는 다음을 확인합니다.

> 기반 타입을 요구하는 코드에 파생 객체를 넣었을 때 기반 타입이 약속한 사용법이 그대로 성립하는가?

예를 들어 모든 `Handler`가

```cpp
Response handle(
    const Request &,
    Store &
) const;
```

라는 의미를 지켜야 한다면 `PutHandler`도 그 사용법을 깨뜨리지 않아야 합니다.

단순히 일부 코드가 비슷하다는 이유만으로 상속할 필요는 없습니다.

## 상속을 코드 재사용만을 위해 사용하지 않습니다

예를 들어 두 클래스에 같은 logging 함수가 필요하다는 이유만으로

```text
LoggingBase
   ↑
Server
Store
```

와 같은 상속 구조를 만드는 것은 적절하지 않을 수 있습니다.

공통 동작이 객체의 본질적인 타입 관계가 아니라 단순한 기능이라면 함수나 별도 객체를 사용하는 편이 더 명확할 수 있습니다.

```cpp
Logger logger;
Server server(logger);
Store store(logger);
```

## 다중 상속

C++98은 다중 상속을 지원합니다.

예를 들어 서로 독립적인 pure interface를 구현하는 경우에는 비교적 단순하게 사용할 수 있습니다.

```cpp
class Readable {
public:
    virtual ~Readable() {
    }

    virtual void read() = 0;
};

class Writable {
public:
    virtual ~Writable() {
    }

    virtual void write() = 0;
};

class File
    : public Readable,
      public Writable {
};
```

문제는 상태를 가진 기반 클래스가 여러 경로로 반복될 때입니다.

```text
      Base
     /    \
   Left  Right
     \    /
    Derived
```

일반 상속이라면 `Derived` 안에 `Base` 부분이 두 개 존재할 수 있습니다.

그 결과 다음 문제가 생깁니다.

* 어느 `Base`를 의미하는지 모호함
* 상태가 두 벌 존재함
* 변환이 모호해짐
* 생성자 호출 관계가 복잡해짐

virtual inheritance로 공통 기반 객체 하나를 공유할 수 있지만 생성 규칙과 객체 배치가 더 복잡해집니다.

단순히 객체를 멤버로 보유해서 해결할 수 있다면 composition이 더 간단한 경우가 많습니다.

## 자주 놓치는 문제

* 기반 pointer로 삭제하면서 기반 소멸자를 non-virtual로 둡니다.
* 이 경우를 단순한 "파생 소멸자 미호출"로만 이해하고 undefined behavior라는 점을 놓칩니다.
* 파생 함수에서 `const`나 parameter 타입이 달라 실제 override가 되지 않습니다.
* C++98에 `override`가 없으므로 의도와 실제 함수가 달라진 것을 놓칩니다.
* 추상 클래스인 `Handler`를 값 parameter로 사용하는 slicing 예제를 작성합니다.
* 일반 다형 객체를 값으로 전달해 slicing을 일으킵니다.
* overload와 virtual override를 같은 개념으로 생각합니다.
* 파생 클래스의 같은 이름 함수 때문에 기반 overload가 가려지는 것을 놓칩니다.
* raw pointer를 저장한 Router에 compiler 기본 복사를 허용합니다.
* handler 생성은 성공했지만 map 삽입이 실패한 경우를 처리하지 않습니다.
* 등록 실패 후 caller와 Router 중 누가 pointer를 삭제할지 정하지 않습니다.
* 코드 중복만을 이유로 상속합니다.
* 기반 타입이 요구하는 의미를 파생 타입이 실제로 만족하는지는 확인하지 않습니다.

## 완료 기준

* overload 선택과 virtual dispatch의 차이를 설명할 수 있습니다.
* C++98에서 `override` 없이 signature를 정확히 맞춰야 하는 이유를 설명할 수 있습니다.
* 기반 pointer로 삭제할 때 virtual destructor가 필요한 이유를 설명할 수 있습니다.
* non-virtual 기반 소멸자로 다형 삭제하는 것이 undefined behavior임을 알고 있습니다.
* abstract base class와 일반 기반 클래스의 차이를 설명할 수 있습니다.
* object slicing이 어떤 복사에서 발생하는지 설명할 수 있습니다.
* pure abstract `Handler`를 값으로 전달하는 예제가 성립하지 않는 이유를 설명할 수 있습니다.
* 이름 가리기와 override를 구분할 수 있습니다.
* `using Base::handle;`이 필요한 경우를 설명할 수 있습니다.
* raw pointer를 보유하는 Router의 소유권, 등록 실패, 소멸, 복사 문제를 처리할 수 있습니다.
* handler 생성 성공 뒤 container 삽입 실패가 별도 실패 경로라는 것을 설명할 수 있습니다.
* "사용한다" 관계와 다형적인 "대체 가능하다" 관계를 구분하여 composition과 inheritance를 선택할 수 있습니다.
