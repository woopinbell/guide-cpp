# 오류·optional·variant·expected

## 목표

실패를 전부 `false`나 문자열로 처리하지 않고, caller가 복구할 수 있는지와 프로그램이 계속 실행 가능한지에 따라 표현을 고릅니다. 입력을 읽은 뒤 상태를 변경하기까지 어느 단계에서 무엇이 실패할 수 있는지 구분합니다.

## 실패 종류를 먼저 나눕니다

```text
외부 문자열
→ 문법 확인
→ 타입 변환
→ 값 범위 확인
→ 현재 상태와 충돌 확인
→ 상태 변경
```

예를 들면 다음 실패는 서로 다릅니다.

- 숫자가 아닌 입력: parse 실패
- 숫자는 맞지만 허용 범위 밖: validation 실패
- 같은 key가 이미 존재: 현재 상태와 충돌
- 메모리 할당 실패: 실행 환경의 자원 부족
- 코드가 허용하지 않는 상태: programmer error

caller가 각 실패에 다른 행동을 해야 한다면 같은 값으로 뭉치지 않습니다.

## 예외와 결과 값

예상 가능한 분기는 값으로 반환하면 호출 지점에서 확인하기 쉽습니다.

```cpp
enum class SubmitError {
    stopped,
    queue_full,
    empty_name
};

Result<JobId, SubmitError> submit(Job job);
```

현재 함수가 정상적으로 완료할 수 없고 여러 호출 단계를 거슬러 올라가야 하는 실패는 예외가 적합할 수 있습니다.

```cpp
Config load_config(const std::filesystem::path& path);
```

파일을 열 수 없거나 내용 전체가 잘못됐다면 호출자가 `try`/`catch`로 초기화 실패를 처리할 수 있습니다.

## `optional`

값의 부재가 유일한 실패 의미일 때 사용합니다.

```cpp
std::optional<Task> find(TaskId id) const;
```

없는 이유를 여러 종류로 구분해야 한다면 `optional`로는 부족합니다. 권한 없음, 연결 실패, 데이터 없음이 모두 `nullopt`가 되면 caller가 적절히 대응할 수 없습니다.

## `variant`

가능한 값 종류가 정해져 있고 모두 정상적인 경우에 사용합니다.

```cpp
using Message = std::variant<TextMessage, Ping, Disconnect>;
```

`std::visit`에서 모든 타입을 처리하게 하면 새로운 타입을 추가했을 때 빠진 분기를 찾기 쉽습니다.

오류를 넣는 용도로도 쓸 수 있지만, value/error 의미를 반복해서 사용한다면 `expected`나 별도 `Result` 타입이 더 읽기 쉽습니다.

## `expected`

C++23의 `std::expected<T, E>`는 성공 값과 오류 값 중 하나를 보관합니다. C++20에서는 작은 `Result` 타입이나 검증된 library를 사용할 수 있습니다.

```cpp
auto result = parse_port(text);
if (!result)
    return report(result.error());
Port port = *result;
```

오류 타입에는 caller가 판단할 정보만 넣습니다. 내부 stack trace나 민감한 경로를 외부 응답에 그대로 노출하지 않습니다.

## 상태를 바꾸기 전에 실패 가능한 작업을 끝냅니다

```cpp
void Store::put(Key key, Value value) {
    if (contains(key))
        throw Conflict{};

    Entry candidate{std::move(key), std::move(value)};
    data_.insert(std::move(candidate));
}
```

검증과 새 값 생성을 먼저 끝내면 중간 실패 뒤 기존 상태를 보존하기 쉽습니다.

여러 값을 한꺼번에 바꿔야 한다면 후보 상태를 만든 뒤 `swap`할 수 있습니다.

```cpp
Config candidate = current;
apply(candidate, patch);
validate(candidate);
current.swap(candidate);
```

## 예외 안전성 수준

- no-throw: 연산이 예외를 밖으로 내보내지 않습니다.
- strong guarantee: 실패하면 호출 전 상태가 유지됩니다.
- basic guarantee: 객체는 유효하지만 값이 일부 달라질 수 있습니다.
- 보장 없음: 객체가 어떤 상태인지 알 수 없습니다.

모든 함수에 strong guarantee가 필요한 것은 아닙니다. 다만 어떤 보장을 제공하는지는 코드와 테스트에서 확인할 수 있어야 합니다.

## 외부 응답으로 바꾸는 위치

내부 타입과 외부 문자열을 섞지 않습니다.

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

내부 예외 메시지는 로그에 남길 수 있지만 외부 응답은 안정된 형식을 사용합니다.

## 자주 놓치는 문제

- 오류 종류가 필요한데 `bool` 하나로 반환합니다.
- 찾지 못함과 I/O 실패를 모두 `nullopt`로 반환합니다.
- 상태를 먼저 변경한 뒤 입력 전체를 검증합니다.
- catch-all에서 실패를 무시하고 성공을 반환합니다.
- 소멸자에서 예외를 던집니다.
- 내부 오류 문자열을 protocol 응답으로 그대로 내보냅니다.

## 완료 기준

- parse, validation, 상태 충돌, 자원 실패를 구분합니다.
- `optional`, `variant`, 결과 타입, 예외를 사용 이유에 따라 선택합니다.
- 실패 가능한 작업과 실제 상태 변경 순서를 설명합니다.
- 함수가 제공하는 예외 안전성 수준을 테스트로 확인합니다.
- 내부 실패를 외부의 안정된 응답과 종료 상태로 바꿉니다.
