# C++98 상속과 다형성

## 목표

상속을 코드 재사용 수단으로만 사용하지 않습니다. 기반 타입 pointer나 reference로 여러 구현을 같은 방식으로 호출해야 할 때 다형성을 적용하고, 삭제·복사·소유권 문제를 함께 처리합니다.

## virtual dispatch

```cpp
class Handler {
public:
    virtual ~Handler() {}
    virtual Response handle(const Request &request, Store &store) const = 0;
};
```

`Handler*`가 실제로 `PutHandler`를 가리키면 virtual call이 파생 클래스 구현을 실행합니다. C++98에는 `override`가 없으므로 signature가 조금만 달라도 override가 아니라 새 함수가 됩니다.

```cpp
class PutHandler : public Handler {
public:
    Response handle(const Request &request, Store &store) const;
};
```

기반 선언의 `const`, 참조, 인자 타입을 정확히 맞춥니다. 컴파일 경고와 기반 pointer를 통한 호출 테스트로 확인합니다.

## virtual destructor

```cpp
Handler *handler = new PutHandler;
delete handler;
```

기반 클래스 소멸자가 virtual이 아니면 파생 소멸자가 호출되지 않습니다. 기반 pointer로 삭제할 가능성이 있다면 virtual destructor가 필요합니다.

반대로 다형 삭제를 허용하지 않는 타입은 소멸자를 protected non-virtual로 두는 설계도 가능하지만, 사용 방법을 명확히 제한해야 합니다.

## pure virtual class

하나 이상의 pure virtual function이 있으면 직접 객체를 만들 수 없습니다. C++98에는 별도 `interface` 키워드가 없습니다.

기반 클래스에 상태를 억지로 넣지 않습니다. 모든 handler가 공유해야 하는 데이터가 아니라면 각 파생 타입이나 외부 객체에 둡니다.

## object slicing

```cpp
void run(Handler handler); // 파생 부분이 잘립니다.
```

기반 타입을 값으로 받으면 파생 객체의 기반 부분만 복사됩니다. 다형 객체는 pointer나 reference로 전달합니다.

```cpp
void run(Handler &handler);
```

복사 가능한 다형 객체가 필요하면 virtual `clone()`을 둘 수 있지만, C++98 raw pointer 반환에서는 반환값 소유자를 반드시 정해야 합니다.

## 이름 가리기와 overload

파생 클래스에서 같은 이름의 함수를 선언하면 기반 클래스의 다른 overload까지 가릴 수 있습니다.

```cpp
class Base {
public:
    void handle(int);
    void handle(const std::string &);
};

class Derived : public Base {
public:
    void handle(int);
};
```

C++98에서는 `using Base::handle;`로 기반 overload를 다시 노출할 수 있습니다.

## handler 소유권

```cpp
class Router {
private:
    std::map<std::string, Handler *> handlers_;
};
```

Router가 pointer를 소유한다면 다음을 처리해야 합니다.

- 생성 중 일부 handler만 등록된 상태에서 예외 발생
- 중복 key로 삽입 실패
- Router 복사 시 두 객체가 같은 pointer를 delete하는 문제
- 소멸 시 모든 handler 삭제

복사가 필요하지 않다면 복사 생성자와 대입을 private으로 선언해 막습니다.

```cpp
Router(const Router &);
Router &operator=(const Router &);
```

C++98에서 `= delete`는 사용할 수 없습니다.

## composition이 더 나은 경우

다음 관계는 상속이 아닙니다.

- Server가 Poller를 사용합니다.
- Service가 Store를 사용합니다.
- Connection이 Parser를 보유합니다.

“사용한다”면 멤버나 참조로 조합합니다. “기반 타입이 필요한 곳에 파생 타입을 넣어도 의미가 같다”면 상속을 검토합니다.

## 다중 상속

서로 독립적인 pure interface를 여러 개 구현하는 경우에는 사용할 수 있습니다. 상태를 가진 기반 클래스가 다이아몬드로 반복되면 생성·소멸과 어떤 기반 객체를 가리키는지 복잡해집니다.

virtual inheritance는 문제를 해결할 수 있지만, 단순한 조합으로 바꿀 수 있는지 먼저 확인합니다.

## 자주 놓치는 문제

- 기반 소멸자가 virtual이 아닌데 기반 pointer로 삭제합니다.
- `const`가 달라 override되지 않은 함수를 호출합니다.
- 다형 객체를 값으로 전달해 slicing이 발생합니다.
- raw pointer container의 복사를 허용합니다.
- handler 등록 실패 뒤 이미 만든 객체를 정리하지 않습니다.
- 코드 재사용만을 위해 상속합니다.

## 완료 기준

- virtual dispatch와 일반 overload를 구분합니다.
- 기반 pointer 삭제에 virtual destructor가 필요한 이유를 설명합니다.
- C++98에서 override signature를 테스트와 warning으로 확인합니다.
- object slicing을 피하고 다형 객체 수명을 명시합니다.
- raw pointer를 보유한 Router의 복사와 실패 정리를 처리합니다.
- composition으로 바꿀 수 있는 상속을 구분합니다.
