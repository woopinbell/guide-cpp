# C++98 객체에 역할 나누기

## 목표

하나의 `main()`에 입력 해석, 데이터 저장, 명령 실행과 출력 형식을 모두 넣지 않습니다. 실제 상태를 누가 보유하고 어떤 함수가 어떤 값을 바꾸는지 찾기 쉬운 프로그램으로 나눕니다.

## 먼저 동작을 분해합니다

줄 단위 명령 프로그램을 예로 들면 다음 작업이 있습니다.

```text
한 줄 읽기
→ 명령과 인자로 분리
→ 명령 이름과 인자 수 확인
→ 저장소 조회 또는 변경
→ 결과 값 만들기
→ 문자열로 출력
```

각 작업을 무조건 별도 클래스로 만들 필요는 없습니다. 하지만 서로 다른 이유로 바뀌는 작업은 분리하면 테스트와 수정 범위가 줄어듭니다.

## Request와 Response

외부 문자열을 바로 저장소 함수에 전달하지 않습니다.

```cpp
struct Request {
    std::string command;
    std::vector<std::string> arguments;
};
```

parser는 문법과 인자 수를 확인하고 `Request`를 만듭니다. 저장소는 원래 입력 문자열을 알 필요가 없습니다.

결과도 문자열로 바로 만들지 않습니다.

```cpp
struct Response {
    enum Code { Ok, Value, NotFound, Error };
    Code code;
    std::string value;
};
```

handler는 `Response`를 반환하고 formatter가 외부 문자열을 만듭니다. 출력 형식이 바뀌어도 저장 규칙은 그대로 둡니다.

## 상태는 한 곳에서 보유합니다

```cpp
class Store {
public:
    void putNew(const std::string &key, const std::string &value);
    bool get(const std::string &key, std::string &value) const;
private:
    std::map<std::string, TextBuffer> data_;
};
```

capacity, 중복 key와 값 메모리는 `Store`가 관리합니다. handler가 map을 직접 수정하게 두면 여러 명령에서 검증 순서가 달라질 수 있습니다.

## parser가 하지 않는 일

parser는 외부 문자열을 유효한 `Request`로 바꾸는 데 집중합니다.

- command 지원 여부 확인
- 인자 수 확인
- 필요하다면 숫자와 enum 변환

parser가 저장소를 조회하거나 상태를 변경하면, 문법 오류가 난 입력이 side effect를 만들 수 있습니다. 모든 문법 확인을 끝낸 뒤 실행합니다.

## handler가 하는 일

handler는 검증된 요청을 상태 연산으로 바꿉니다.

```cpp
Response PutHandler::handle(const Request &request, Store &store) const {
    store.putNew(request.arguments[0], request.arguments[1]);
    return Response(Response::Ok);
}
```

명령 선택을 handler 내부의 긴 `if` 체인으로 다시 하지 않습니다. Router가 명령 이름에 맞는 handler를 고릅니다.

## 조립 위치

`main()`은 객체를 만든 순서와 수명을 보여 주는 곳입니다.

```cpp
Store store(capacity);
Router router;
RequestParser parser;
ResponseFormatter formatter;
```

반복문은 한 줄을 읽고 다음을 연결합니다.

```text
parse → route → handle → format → write
```

내부 예외를 외부 응답으로 바꾸는 것도 이 위치에서 처리할 수 있습니다. 저장소와 parser가 stdout을 직접 쓰지 않게 하면 단위 테스트가 쉬워집니다.

## 클래스 수를 늘리는 것이 목표가 아닙니다

분리한 타입이 실제로 하는 일이 한 줄이고 다른 변경 이유도 없다면 합치는 편이 낫습니다. 다음 질문으로 판단합니다.

- 독립적으로 지켜야 하는 상태가 있습니까?
- 다른 부분과 별도로 테스트할 실패가 있습니까?
- 입력 형식 변경과 데이터 규칙 변경이 같은 파일을 건드립니까?
- 이 타입을 사용하는 호출자가 둘 이상입니까?
- 생성과 소멸 순서를 별도로 관리해야 합니까?

## 의존성을 숨기지 않습니다

전역 변수 대신 생성자나 함수 인자로 전달합니다.

```cpp
class CommandService {
public:
    explicit CommandService(Store &store) : store_(store) {}
private:
    Store &store_;
};
```

참조 멤버를 사용하면 `Store`가 `CommandService`보다 오래 살아야 합니다. 조립 위치에서 생성 순서를 맞춥니다.

## 자주 놓치는 문제

- parser가 입력 검증 중 저장소까지 변경합니다.
- handler가 직접 출력해 단위 테스트가 stdout 캡처에 의존합니다.
- 여러 클래스가 같은 map을 직접 수정합니다.
- `Manager` 하나가 parser, store, network와 logging을 모두 가집니다.
- 타입을 많이 나눴지만 모든 함수가 다른 타입의 private 상태를 알아야 합니다.
- 전역 singleton으로 의존성과 소멸 순서를 숨깁니다.

## 완료 기준

- 입력 해석과 상태 변경이 다른 함수에서 일어납니다.
- 데이터를 직접 보유하는 타입이 하나로 정해져 있습니다.
- handler는 검증된 요청만 받습니다.
- 출력 형식 변경이 저장소 구현을 건드리지 않습니다.
- `main()`에서 객체 생성 순서와 참조 수명을 설명합니다.
- 각 타입을 별도 테스트할 구체적인 이유가 있습니다.
