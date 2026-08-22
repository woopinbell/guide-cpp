# Modern C++에서 C++98로 옮기기

## 목적

Modern C++에서 익힌 값·수명·오류 처리 원칙을 C++98에서도 유지하기 위한 대응표입니다. 최신 문법을 비슷하게 흉내 내는 것이 아니라, 사용할 수 없는 기능이 해결하던 문제를 다시 확인합니다.

## 주요 대응

| Modern C++ | C++98에서의 처리 |
| --- | --- |
| move semantics | 깊은 복사, 복사 금지 또는 명시적인 소유권 이전 함수 |
| Rule of Zero | 자원을 직접 소유하면 Rule of Three |
| `std::unique_ptr` | raw pointer 소유자를 한 클래스에 고정하고 복사 금지 |
| `std::shared_ptr` | 가능하면 값 또는 명시적인 owner로 단순화; 필요 시 직접 참조 카운트는 신중히 설계 |
| `nullptr` | `0`; overload 모호성을 줄이는 API 설계 |
| `enum class` | enum을 타입/클래스 scope 안에 두고 외부 정수 입력 검증 |
| lambda | function object 또는 함수 pointer |
| range-for | 명시적인 iterator loop |
| `auto` | 전체 iterator와 타입 이름 작성 |
| `override` | signature 일치, warning, 기반 pointer 호출 테스트 |
| `noexcept` | `throw()` 명세 또는 문서와 구현으로 예외 없음 보장 |
| `optional<T>` | `bool` + output parameter, 상태 값 또는 별도 결과 클래스 |
| `variant` | tagged union, enum + 값 필드 또는 다형성 |
| `expected<T, E>` | 성공 여부와 value/error를 보관하는 별도 클래스 |
| `std::jthread` | pthread/thread wrapper, stop flag, 명시적인 join |
| `std::filesystem` | POSIX API 또는 플랫폼별 wrapper |
| concepts | template 요구 연산 문서화, 작은 compile-fail 검사 |

## 소유권

Modern C++에서는 signature에 소유권을 드러낼 수 있습니다.

```cpp
void install(std::unique_ptr<Handler> handler);
```

C++98 raw pointer에서는 성공과 실패 뒤 소유자를 직접 정합니다.

```cpp
void addHandler(const std::string &name, Handler *handler);
```

다음 중 하나를 문서와 코드로 고정합니다.

- 함수 진입과 동시에 소유권을 받으며 실패해도 함수가 delete합니다.
- 성공할 때만 소유권을 받고 실패하면 caller가 delete합니다.

가능하면 생성과 등록을 같은 객체 내부에서 수행해 pointer가 외부를 오가는 구간을 줄입니다.

## 복사와 이동

이동 전용 자원을 C++98 container에 값으로 넣기 어렵습니다. 선택지는 다음과 같습니다.

- 객체 복사를 막고 pointer owner container에 저장합니다.
- 자원을 나타내는 값만 복사하고 실제 handle은 별도 manager가 보유합니다.
- 깊은 복사가 자연스러운 값 타입으로 바꿉니다.

Modern 설계를 문법만 바꿔 그대로 유지하려 하지 않습니다.

## 오류 표현

Modern C++:

```cpp
Result<Value, Error> parse(std::string_view text);
```

C++98:

```cpp
bool parse(const std::string &text, Value &value, Error &error);
```

또는 예상하지 못한 실패는 예외 타입으로 구분합니다. output parameter는 성공한 경우에만 유효하다는 조건을 적습니다.

## callback

Modern C++ lambda가 capture한 값의 수명을 compiler가 관리합니다. C++98 function object는 필요한 참조나 pointer를 멤버로 보유하므로 수명 조건을 직접 확인합니다.

```cpp
class Predicate {
public:
    explicit Predicate(const Config &config) : config_(config) {}
    bool operator()(const Item &item) const;
private:
    const Config &config_;
};
```

`Predicate`가 `Config`보다 오래 남으면 참조가 무효가 됩니다.

## thread와 종료

`jthread`처럼 소멸 시 자동 join되는 표준 타입이 없으면 wrapper를 만들거나 모든 생성 경로와 종료 경로에서 join을 확인합니다.

```text
thread 생성 성공 수 기록
→ stop flag 설정
→ 대기 중 thread 깨움
→ 생성된 thread만 join
→ 공유 상태 소멸
```

thread 생성 중 일부만 성공한 경우도 처리합니다.

## 적용 순서

1. Modern 코드가 해결하는 실제 문제를 적습니다.
2. 최신 기능이 제공하는 수명·타입·오류 보장을 확인합니다.
3. C++98에서 같은 보장을 제공할 최소 타입과 정리 순서를 만듭니다.
4. 복사 실패, 중간 생성 실패, 종료 실패 테스트를 추가합니다.
5. 문법이 아니라 관찰 가능한 결과를 비교합니다.

## 완료 기준

- Modern 기능을 제거했을 때 사라지는 보장을 설명합니다.
- C++98 raw pointer의 소유권과 실패 뒤 해제자를 정합니다.
- Rule of Zero 설계를 Rule of Three 또는 복사 금지로 옮깁니다.
- `optional`, `variant`, 결과 타입을 C++98 값으로 표현합니다.
- lambda와 thread wrapper의 참조 수명·종료 순서를 검사합니다.
