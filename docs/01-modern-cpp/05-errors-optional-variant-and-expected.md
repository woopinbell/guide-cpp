# 오류·optional·variant·expected

## 목표

실패를 전부 `false`, `nullptr`, 빈 문자열처럼 같은 값으로 표현하지 않습니다.

먼저 다음을 구분합니다.

* 정상 결과 중 값이 없는 경우인지
* 호출자가 예상하고 복구할 수 있는 실패인지
* 현재 연산을 정상적으로 완료할 수 없는 예외적 실패인지
* 코드 자체의 전제 조건이 깨진 programmer error인지

또한 외부 입력을 받은 뒤 상태를 변경하기까지 어느 단계에서 무엇이 실패할 수 있는지 구분합니다.

## 실패 종류를 먼저 나눕니다

외부 입력을 처리하는 작업은 보통 여러 단계로 나눌 수 있습니다.

```text
외부 문자열
    ↓
문법 확인
    ↓
타입 변환
    ↓
값 범위와 의미 확인
    ↓
현재 상태와 충돌하는지 확인
    ↓
새 상태를 준비
    ↓
실제 상태 변경
```

예를 들어 `"70000"`이라는 문자열을 TCP port로 등록한다고 가정합니다.

다음 실패는 서로 의미가 다릅니다.

* `"abc"`: 숫자로 해석할 수 없는 **parse 실패**
* `"70000"`: 정수 변환에는 성공했지만 허용 범위를 벗어난 **validation 실패**
* 이미 등록된 port: 현재 프로그램 상태와 맞지 않는 **상태 충돌**
* 새 항목을 저장하는 중 메모리 할당 실패: **실행 환경의 자원 실패**
* 코드 내부에서 절대로 존재해서는 안 되는 상태 발견: **programmer error**

caller가 실패마다 다른 행동을 해야 한다면 하나의 `bool`이나 문자열로 뭉치지 않습니다.

예:

```cpp
enum class SubmitError {
    stopped,
    queue_full,
    empty_name
};
```

`queue_full`이면 나중에 다시 시도할 수 있지만 `empty_name`이면 입력을 고쳐야 할 수 있습니다. 두 경우를 단순한 `false` 하나로 반환하면 caller는 차이를 알 수 없습니다.

## 오류 표현을 선택하는 기준

대략 다음처럼 생각할 수 있습니다.

```text
값이 없지만 정상인가?
    └─ yes → optional

여러 정상적인 값 종류 중 하나인가?
    └─ yes → variant

성공 값 또는 예상 가능한 오류인가?
    └─ yes → expected / Result

현재 호출 지점에서 매번 처리하기보다
여러 호출 단계를 벗어나 처리하는 실패인가?
    └─ yes → exception 고려

프로그램 논리가 깨진 상태인가?
    └─ assertion / terminate / bug 수정 대상
```

이것은 절대적인 규칙은 아닙니다. API의 사용 방식과 오류 빈도, 호출자가 어떤 처리를 해야 하는지를 기준으로 선택합니다.

## 예외와 결과 값

### 예상 가능한 실패를 값으로 반환

실패가 해당 작업의 정상적인 분기이고 caller가 바로 처리할 가능성이 높다면 결과 값으로 표현하는 것이 읽기 쉽습니다.

C++23에서는 다음처럼 `std::expected`를 사용할 수 있습니다.

```cpp
std::expected<JobId, SubmitError> submit(Job job);
```

사용:

```cpp
auto result = submit(std::move(job));

if (!result) {
    switch (result.error()) {
    case SubmitError::stopped:
        // 서비스가 중지됨
        break;

    case SubmitError::queue_full:
        // 다시 시도할 수 있음
        break;

    case SubmitError::empty_name:
        // 입력 오류
        break;
    }
}

JobId id = *result;
```

오류 처리가 함수 signature에 드러나므로 caller가 실패 가능성을 쉽게 확인할 수 있습니다.

### 예외

예외는 단순히 "심각한 오류"라는 이유만으로 선택하지 않습니다.

예외의 장점 중 하나는 현재 함수가 정상 결과를 만들 수 없을 때 여러 호출 단계를 직접 오류 값으로 전달하지 않고 빠져나갈 수 있다는 것입니다.

```cpp
Config load_config(const std::filesystem::path& path);
```

사용:

```cpp
try {
    Config config = load_config(path);
    run(config);
} catch (const ConfigError& error) {
    report(error);
    return EXIT_FAILURE;
}
```

`load_config()` 내부가 여러 함수로 구성되어 있다고 합시다.

```text
main
↓
load_config
↓
read_file
↓
parse_document
↓
parse_field
```

`parse_field()`에서 복구할 방법이 없고 최종적으로 `main`이 초기화 실패를 처리해야 한다면 예외를 사용하여 여러 단계를 빠져나가는 것이 자연스러울 수 있습니다.

반대로 "없으면 다른 값을 사용한다"처럼 빈번한 정상 분기를 매번 예외로 표현하면 제어 흐름을 읽기 어려울 수 있습니다.

## programmer error는 일반 입력 오류와 다릅니다

다음은 사용자 입력 실패와 성격이 다릅니다.

```cpp
void process(State state) {
    assert(state != State::impossible);
}
```

프로그램의 올바른 코드라면 발생해서는 안 되는 상태는 일반적인 `expected` 오류로 계속 전달하기보다 다음 중 하나가 더 적절할 수 있습니다.

* `assert`
* invariant 검사
* 예외 후 프로세스 종료
* 즉시 실패하여 bug를 드러내는 처리

중요한 것은 programmer error를 `"invalid input"` 같은 정상적인 외부 오류로 숨기지 않는 것입니다.

## `std::optional`

`std::optional<T>`는 **값이 있거나 없는 두 상태만으로 의미가 충분한 경우**에 적합합니다.

```cpp
std::optional<Task> find(TaskId id) const;
```

사용:

```cpp
auto task = store.find(id);

if (!task) {
    // 그런 Task가 없음
    return;
}

use(*task);
```

여기에서 `nullopt`의 의미가 오직 하나라면 명확합니다.

```text
Task 존재
또는
Task 없음
```

하지만 다음처럼 여러 실패를 하나로 합치면 부족합니다.

```cpp
std::optional<Data> load();
```

`nullopt`가 다음 중 무엇인지 알 수 없다면 caller가 대응하기 어렵습니다.

* 파일 없음
* 접근 권한 없음
* 파일 손상
* 연결 실패
* 데이터 없음

이 경우에는 `expected<Data, LoadError>` 같은 오류 정보가 있는 타입이 적합합니다.

### `optional`은 반드시 "오류"를 의미하지 않습니다

다음은 실패가 아니라 정상적인 부재입니다.

```cpp
std::optional<std::string> middle_name;
```

중간 이름이 없는 것은 오류가 아닙니다.

따라서 `optional`을 "간단한 오류 반환 타입"으로만 이해하면 안 됩니다.

핵심 의미는 **값의 존재 여부**입니다.

## `std::variant`

`std::variant`는 정해진 여러 타입 중 정확히 하나를 저장합니다.

```cpp
using Message = std::variant<
    TextMessage,
    Ping,
    Disconnect
>;
```

여기서 세 타입은 성공과 실패 관계가 아닙니다.

모두 정상적인 `Message`의 한 형태입니다.

처리는 `std::visit`으로 작성할 수 있습니다.

```cpp
std::visit(
    [](const auto& message) {
        handle(message);
    },
    message
);
```

또는 overload helper를 사용해 각 타입의 처리를 명확하게 분리할 수 있습니다.

```cpp
std::visit(
    overloaded{
        [](const TextMessage& message) {
            handle_text(message);
        },
        [](const Ping& ping) {
            handle_ping(ping);
        },
        [](const Disconnect& disconnect) {
            handle_disconnect(disconnect);
        }
    },
    message
);
```

새 variant alternative를 추가하면 기존 visitor에서 처리가 부족한 부분을 compiler가 발견하는 데 도움이 될 수 있습니다.

## `variant`와 `expected`의 차이

다음 타입도 기술적으로 만들 수 있습니다.

```cpp
std::variant<Value, Error> result;
```

하지만 이 구조가 코드 전체에서 반복된다면 의미를 직접 표현하는 타입이 더 명확합니다.

```cpp
std::expected<Value, Error>
```

둘의 의미는 다릅니다.

```text
variant<A, B>
→ A와 B는 여러 가능한 값 종류

expected<T, E>
→ T는 성공
→ E는 실패
```

타입 자체가 의도를 드러냅니다.

## `std::expected`

C++23의 `std::expected<T, E>`는 성공 값 `T` 또는 오류 값 `E` 중 하나를 저장합니다.

예:

```cpp
enum class PortError {
    empty,
    invalid_number,
    out_of_range
};

std::expected<Port, PortError>
parse_port(std::string_view text);
```

사용:

```cpp
auto result = parse_port(text);

if (!result) {
    return report(result.error());
}

Port port = *result;
```

성공 시:

```text
expected
└── Port
```

실패 시:

```text
expected
└── PortError
```

C++20을 사용한다면 다음을 선택할 수 있습니다.

* 프로젝트에서 정의한 작은 `Result<T, E>`
* 검증된 외부 library의 expected 타입
* 문제 규모에 따라 `variant<T, E>`

표준에 없는 타입을 직접 구현할 때는 단순한 저장 기능을 넘어 복사·이동·예외 안전성까지 올바르게 구현해야 하므로 필요 이상으로 복잡한 `Result`를 직접 만들지는 않습니다.

## 오류 타입에는 판단에 필요한 정보를 넣습니다

오류를 enum 하나로 충분히 표현할 수도 있습니다.

```cpp
enum class ParseError {
    invalid_number,
    out_of_range
};
```

caller가 추가 정보가 필요하다면 구조체로 만들 수 있습니다.

```cpp
struct ParseError {
    ParseErrorCode code;
    std::size_t position;
};
```

그러면 다음처럼 처리할 수 있습니다.

```cpp
auto result = parse(text);

if (!result) {
    std::cerr
        << "parse error at "
        << result.error().position
        << '\n';
}
```

오류 타입에 "현재 caller가 실제로 판단하는 데 필요한 정보"를 넣습니다.

반대로 내부 구현의 모든 정보를 외부 오류 타입에 넣으면 결합이 커질 수 있습니다.

특히 외부 protocol 응답에는 다음을 그대로 노출하지 않습니다.

* 내부 파일 경로
* SQL query
* stack trace
* secret/token
* 상세한 내부 예외 메시지

## 상태 변경 전 실패 가능 작업을 끝냅니다

다음 작업을 생각할 수 있습니다.

```cpp
void Store::put(Key key, Value value) {
    if (contains(key)) {
        throw Conflict{};
    }

    Entry candidate{
        std::move(key),
        std::move(value)
    };

    data_.insert(std::move(candidate));
}
```

의도는 실제 상태를 변경하기 전에 가능한 검증과 새 값 준비를 최대한 끝내는 것입니다.

```text
입력 검증
↓
새 값 준비
↓
실패 가능 작업 수행
↓
최종 상태 변경
```

이렇게 하면 중간에 실패했을 때 기존 상태가 그대로 남도록 만들기 쉬워집니다.

다만 `insert()` 자체도 allocation 때문에 실패할 수 있다는 점은 별도로 고려해야 합니다. 어떤 예외 안전성을 제공하는지는 사용하는 container 연산의 보장까지 확인해야 합니다.

## 강한 예외 안전성과 commit 방식

여러 필드를 한 번에 변경해야 한다고 합시다.

다음처럼 원본을 직접 조금씩 수정하면 중간 실패 시 일부만 바뀔 수 있습니다.

```cpp
current.host = patch.host;
current.port = parse_port(patch.port); // 여기에서 실패
current.timeout = patch.timeout;
```

대신 후보 상태를 만들 수 있습니다.

```cpp
Config candidate = current;

apply(candidate, patch);
validate(candidate);

current.swap(candidate);
```

개념적으로:

```text
current
   │
   └── 복사 → candidate
                ↓
             변경 적용
                ↓
              검증
                ↓
              성공?
             /     \
           no       yes
           │         │
candidate 폐기    swap
           │         │
current 유지    새 상태 적용
```

최종 `swap`이 `noexcept`라면 strong guarantee를 구현하기 좋은 형태입니다.

이 방식을 일반적으로 **prepare then commit** 형태로 생각할 수 있습니다.

## 예외 안전성 수준

예외 안전성은 "예외를 사용하느냐"와 다른 개념입니다.

함수가 실패했을 때 프로그램 상태가 어떤 보장을 받는지를 말합니다.

### no-throw guarantee

연산이 예외를 밖으로 내보내지 않습니다.

```cpp
void swap(Value& other) noexcept;
```

"절대 실패하지 않는다"와 반드시 같은 의미는 아닙니다. 내부에서 오류를 다른 방식으로 처리할 수도 있습니다.

### strong guarantee

실패하면 호출 전 상태가 유지됩니다.

```text
성공
→ 새 상태

실패
→ 이전 상태 그대로
```

흔히 transaction과 비슷한 "성공하거나 아무것도 바꾸지 않음"으로 설명합니다.

### basic guarantee

실패 후에도 객체는 유효하고 자원 누수나 invariant 파괴는 없지만 값 일부는 달라질 수 있습니다.

```text
실패
→ 객체는 계속 사용 가능
→ 그러나 값이 호출 전과 같다는 보장은 없음
```

### no guarantee

실패 후 객체 상태에 의미 있는 보장을 제공하지 않습니다.

가능하면 외부에 노출되는 중요한 타입에서는 피합니다.

## "유효한 상태"의 의미

basic guarantee에서 말하는 유효 상태는 단순히 프로그램이 crash하지 않는다는 뜻이 아닙니다.

예를 들어 클래스 invariant가 다음이라면:

```text
size_ <= capacity_
data_ == nullptr 이면 capacity_ == 0
```

예외 후에도 이 조건들은 유지되어야 합니다.

즉 해당 객체의 public member function을 다시 호출하거나 안전하게 파괴할 수 있어야 합니다.

## 오류를 외부 응답으로 바꾸는 위치

내부 오류 표현과 외부 protocol 표현을 동일하게 만들 필요는 없습니다.

예:

```cpp
try {
    Response response = service.execute(request);
    write(format(response));
} catch (const ParseError&) {
    write("BAD_REQUEST\n");
} catch (const ConflictError&) {
    write("CONFLICT\n");
} catch (const std::exception&) {
    write("INTERNAL_ERROR\n");
}
```

내부에서는 다음처럼 상세한 정보가 있을 수 있습니다.

```text
ParseError
- field: port
- position: 7
- input: ...
```

하지만 외부에는 안정된 protocol 결과만 보냅니다.

```text
BAD_REQUEST
```

이렇게 하면 다음을 분리할 수 있습니다.

```text
내부 오류
→ 진단과 로그에 적합한 상세 정보

외부 오류
→ 공개 API 또는 protocol에 약속한 안정된 형식
```

외부 응답 문자열을 내부 깊은 함수에서 직접 만들기 시작하면 protocol 형식 변경이 내부 코드 전체에 퍼질 수 있습니다.

## catch할 위치

예외는 보통 **실제로 처리할 수 있는 위치**에서 잡습니다.

좋지 않은 예:

```cpp
try {
    do_work();
} catch (const std::exception&) {
    // 아무것도 하지 않음
}
```

이렇게 하면 프로그램은 실패한 사실조차 잃어버립니다.

또 다음처럼 의미 없이 다시 던지는 catch도 불필요합니다.

```cpp
try {
    do_work();
} catch (...) {
    throw;
}
```

반면 다음처럼 맥락을 추가하거나 다른 오류 체계로 변환한다면 의미가 있습니다.

```cpp
try {
    return read_config_file(path);
} catch (const FileError& error) {
    throw ConfigError{
        "failed to load configuration",
        error
    };
}
```

## 자주 놓치는 문제

* 여러 오류 종류가 필요한데 `bool` 하나만 반환합니다.
* 정상적인 값 부재와 실제 실패를 모두 `nullopt`로 반환합니다.
* 여러 정상 값 종류를 나타내는 `variant`와 성공/실패를 나타내는 `expected`를 구분하지 않습니다.
* 사용자가 고칠 수 있는 입력 오류와 programmer error를 동일하게 처리합니다.
* 상태를 먼저 변경한 뒤 나머지 입력을 검증합니다.
* 여러 필드를 순차적으로 수정하다 중간 실패하여 부분 상태가 남습니다.
* catch-all에서 오류를 무시하고 성공한 것처럼 계속 실행합니다.
* 처리할 수 없는 예외를 너무 깊은 함수에서 무조건 catch합니다.
* 내부 예외 메시지를 protocol 또는 사용자 응답에 그대로 노출합니다.
* strong guarantee를 주장하지만 실제로 사용하는 container 연산의 예외 보장을 확인하지 않습니다.

## 코드를 읽을 때 확인할 질문

1. 이 경우는 정상적인 값 부재입니까, 실패입니까?
2. 실패라면 caller가 종류를 구분해야 합니까?
3. 현재 호출 지점에서 바로 복구하는 것이 자연스럽습니까?
4. `optional`, `expected`, exception 중 무엇이 의도를 가장 직접적으로 표현합니까?
5. programmer error를 정상적인 사용자 오류로 숨기고 있지는 않습니까?
6. 실제 상태를 변경하기 전에 검증과 값 준비를 끝낼 수 있습니까?
7. 중간에 allocation이나 다른 함수 호출이 실패하면 기존 상태가 어떻게 됩니까?
8. 이 함수는 no-throw, strong, basic 중 어떤 보장을 제공합니까?
9. 내부 오류를 외부 API나 protocol 오류로 변환하는 위치는 어디입니까?
10. 오류를 catch한 뒤 실제로 처리하고 있습니까?

## 완료 기준

* parse, validation, 상태 충돌, 자원 부족, programmer error를 구분합니다.
* `optional`이 값의 부재를 표현한다는 것을 설명합니다.
* `variant`가 여러 정상 값 중 하나를 표현한다는 것을 설명합니다.
* `expected`가 성공 값과 예상 가능한 오류를 구분한다는 것을 설명합니다.
* 예외가 여러 호출 단계를 벗어나는 실패 전달 방식이 될 수 있음을 설명합니다.
* 상태 변경 전 검증과 값 준비를 수행하는 이유를 설명합니다.
* no-throw, strong, basic guarantee의 차이를 설명합니다.
* 내부 오류 정보를 외부의 안정된 응답 형식과 분리합니다.