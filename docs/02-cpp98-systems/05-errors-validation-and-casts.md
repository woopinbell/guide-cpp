# C++98 오류 처리·입력 검증·캐스트

## 목표

외부 문자열을 내부 값으로 바꾸기 전에 전체 입력과 범위를 검사하고, 실패한 요청이 기존 상태를 일부만 변경하지 않게 합니다. 예외, 반환값과 출력 매개변수를 상황에 맞게 사용합니다.

## 입력 처리 순서

```text
문자열 분리
→ token 수 확인
→ 숫자·enum 변환
→ 값 범위 확인
→ 현재 상태와 충돌 확인
→ 상태 변경
```

문법 확인이 끝나기 전에 map에 값을 넣거나 process를 시작하지 않습니다.

## 숫자 변환

`atoi()`는 실패와 0을 구분할 수 없고 overflow 처리도 불분명합니다. `strtol()`과 end pointer를 사용합니다.

```cpp
int parseInt(const char *text) {
    char *end = 0;
    errno = 0;
    const long value = std::strtol(text, &end, 10);

    if (errno == ERANGE || end == text || *end != '\0'
        || value < INT_MIN || value > INT_MAX) {
        throw ParseError("invalid integer");
    }
    return static_cast<int>(value);
}
```

앞뒤 공백을 허용할지 먼저 정합니다. 일부만 소비한 `42x`를 성공으로 처리하지 않습니다.

## 실패를 타입으로 구분합니다

```cpp
class ParseError : public std::runtime_error { /* ... */ };
class ConflictError : public std::runtime_error { /* ... */ };
class StoreFullError : public std::runtime_error { /* ... */ };
```

최상위 반복문은 실패 종류를 안정된 응답으로 바꿀 수 있습니다.

```cpp
catch (const ParseError &) {
    std::cout << "BAD_REQUEST\n";
}
catch (const ConflictError &) {
    std::cout << "CONFLICT\n";
}
```

내부 예외 메시지를 외부 protocol로 사용하지 않습니다. 구현 세부사항이나 파일 경로가 노출되고 문구 변경이 protocol 변경이 될 수 있습니다.

## 반환값과 출력 매개변수

C++98에는 `optional`이 없습니다. 값이 없을 수 있는 조회는 다음처럼 표현할 수 있습니다.

```cpp
bool Store::get(const std::string &key, std::string &value) const;
```

`true`일 때만 `value`를 사용한다는 규칙을 문서화합니다. 실패 종류가 둘 이상이면 enum 또는 별도 결과 타입을 만듭니다.

## 상태 변경 전 검증

```cpp
void Store::putNew(const std::string &key, const std::string &value) {
    if (data_.find(key) != data_.end())
        throw ConflictError("key already exists");
    if (data_.size() >= capacity_)
        throw StoreFullError("store is full");

    TextBuffer owned(value.c_str());
    data_.insert(std::make_pair(key, owned));
}
```

검증과 값 복사를 먼저 끝내면 실패 뒤 key 수가 변하지 않습니다. 여러 필드를 갱신할 때는 후보 객체를 만든 뒤 `swap()`합니다.

## 예외가 지나가는 범위

소멸자가 자원을 정리하도록 만들면 중간 함수에서 모든 예외를 잡을 필요가 없습니다. 복구할 방법이 있는 위치에서만 잡습니다.

- parser는 `ParseError`를 던집니다.
- store는 conflict와 capacity 오류를 던집니다.
- `main`은 외부 응답으로 바꿉니다.

catch한 뒤 아무 처리 없이 계속 실행하면 오류를 숨길 수 있습니다. 현재 상태가 계속 사용 가능한지 확인합니다.

## cast

### `static_cast`

정수 범위를 검사한 뒤 타입을 바꾸거나, 명시적인 upcast에 사용합니다.

### `dynamic_cast`

virtual function이 있는 기반 타입에서 실제 파생 타입을 확인합니다.

```cpp
Derived *derived = dynamic_cast<Derived *>(base);
if (derived == 0)
    return false;
```

pointer cast 실패는 `0`, reference cast 실패는 `std::bad_cast`입니다.

### `const_cast`

원래 객체가 const인데 변경하면 정의되지 않은 동작입니다. 단순히 오래된 C API signature에 맞추기 위해 사용하더라도 API가 실제로 수정하지 않는지 확인합니다.

### `reinterpret_cast`

socket address, byte buffer 같은 낮은 수준 코드에서 필요할 수 있지만, 정렬·실제 타입·수명을 자동으로 보장하지 않습니다.

## overflow

signed integer overflow는 정의되지 않은 동작입니다. 계산 전에 검사하거나 더 넓은 타입에서 계산합니다.

```cpp
if (right > 0 && left > INT_MAX - right)
    throw ArithmeticError("overflow");
```

곱셈과 음수 경계는 더 복잡하므로 입력 범위와 연산별 검사를 분리합니다.

## 자주 놓치는 문제

- `atoi()` 결과 0을 성공으로 처리합니다.
- 숫자의 앞부분만 읽고 뒤 문자를 무시합니다.
- map을 먼저 수정하고 중복·용량을 나중에 검사합니다.
- 모든 예외를 `catch (...)`로 잡고 성공 응답을 냅니다.
- 범위 검사 없이 `static_cast<int>`합니다.
- `reinterpret_cast`가 실제 객체 타입을 바꾼다고 생각합니다.

## 완료 기준

- 입력 전체 소비와 정수 범위를 확인합니다.
- 문법 오류, 상태 충돌, 용량 부족과 내부 실패를 구분합니다.
- 실패 가능한 작업을 상태 변경 전에 수행합니다.
- C++98에서 조회 부재와 여러 오류 종류를 표현합니다.
- 네 가지 C++ cast의 목적과 한계를 설명합니다.
- overflow와 partial update를 실제 실패 테스트로 재현합니다.
