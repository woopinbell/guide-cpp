# C++98 객체에 역할 나누기

## 목표

하나의 `main()` 함수에 다음 작업을 모두 넣지 않습니다.

* 입력 읽기
* 문자열 해석
* 명령 선택
* 데이터 조회와 변경
* 결과 생성
* 출력 문자열 생성

목표는 클래스 수를 늘리는 것이 아닙니다.

프로그램을 읽을 때 다음 질문에 쉽게 답할 수 있도록 나누는 것이 목적입니다.

* 실제 데이터는 누가 보유하는가?
* 입력 문자열은 어디까지 문자열로 취급되는가?
* 어느 함수부터 프로그램 상태를 변경할 수 있는가?
* 출력 형식은 어디에서 결정되는가?
* 실패했을 때 어느 객체의 상태가 변경되었는가?

## 먼저 동작을 분해합니다

줄 단위 명령 프로그램을 예로 들면 한 요청은 대략 다음 단계를 거칩니다.

```text
한 줄 읽기
→ 문자열을 token으로 분리
→ 명령 이름과 인자 형식 확인
→ 실행할 명령 선택
→ 저장소 조회 또는 변경
→ 실행 결과 생성
→ 외부 출력 문자열 생성
→ 출력
```

이 단계마다 반드시 클래스를 하나씩 만들 필요는 없습니다.

예를 들어 단순한 문자열 formatter 하나가 함수 하나로 충분하다면 굳이 클래스로 만들 이유가 없습니다.

분리의 기준은 주로 **서로 다른 이유로 변경되는 코드가 실제로 섞여 있는가**입니다.

예를 들어 다음 둘은 별개의 변경입니다.

* `"PUT a b"` 문법을 `"SET a b"`로 변경
* 중복 key를 허용하지 않도록 저장 규칙 변경

두 변경 때문에 항상 같은 함수를 수정해야 한다면 입력 처리와 데이터 규칙이 지나치게 붙어 있을 가능성이 있습니다.

## Request는 외부 문자열과 내부 실행 사이의 값입니다

저장소가 원래 입력 문자열을 직접 해석하게 하지 않을 수 있습니다.

```cpp
struct Request {
    std::string command;
    std::vector<std::string> arguments;
};
```

예를 들어 다음 입력이

```text
PUT name alice
```

parser를 통과하면 다음과 같은 값이 될 수 있습니다.

```text
command   = "PUT"
arguments = ["name", "alice"]
```

이후 코드에서는 공백이 몇 개였는지, 원본 문자열이 어떻게 구성되어 있었는지 알 필요가 없습니다.

이렇게 하면 저장소나 handler가 문자열 tokenization까지 알아야 하는 상황을 피할 수 있습니다.

### 다만 `Request`가 무엇을 보장하는지는 명확히 해야 합니다

`Request`라는 구조체가 단순히

```cpp
std::string command;
std::vector<std::string> arguments;
```

만 가진다면 타입 자체는 다음과 같은 값도 허용합니다.

```text
command = ""
arguments = 300개
```

따라서 "`Request`를 받았으므로 이미 완전히 유효하다"는 것은 C++ 타입 시스템이 보장하는 사실이 아니라 **프로그램 내부 규칙**입니다.

예를 들어 다음 규칙을 정할 수 있습니다.

> `RequestParser::parse()`가 성공해서 반환한 `Request`만 handler에 전달한다.

그렇다면 handler는 parser가 제공하는 보장을 전제로 사용할 수 있습니다.

더 강한 보장이 필요하다면 명령별 요청 타입을 따로 만드는 방법도 있습니다.

```cpp
struct PutRequest {
    std::string key;
    std::string value;
};
```

그러면 `PutHandler`가 `arguments[0]`, `arguments[1]`처럼 위치 기반 데이터를 다시 해석할 필요가 없습니다.

하지만 작은 프로그램이라면 하나의 일반 `Request`가 더 단순할 수 있습니다.

## parser의 범위를 먼저 정합니다

parser의 최소 역할은 외부 문자열을 프로그램이 사용할 수 있는 값으로 변환하는 것입니다.

예:

* token 분리
* 빈 입력 판정
* 필요한 인자 추출
* 숫자 문자열을 정수로 변환
* enum에 대응하는 문자열 확인
* 잘못된 문법 거부

다만 다음 항목은 프로그램 설계에 따라 parser 또는 router 쪽에 둘 수 있습니다.

> command 지원 여부 확인

예를 들어 parser가 다음 명령 목록까지 알고 있다면,

```text
PUT
GET
DELETE
```

parser가 command 유효성까지 검사할 수 있습니다.

반대로 확장 가능한 handler 등록 구조라면 parser는 단순히

```text
command = "PUT"
```

까지만 만들고 실제 지원 여부는 Router가 판단하는 편이 자연스럽습니다.

즉 다음을 구분해야 합니다.

```text
"이 문자열이 명령 형식인가?"
```

와

```text
"현재 프로그램에 이 명령을 처리할 handler가 등록되어 있는가?"
```

둘은 같은 질문이 아닙니다.

## 문법 검증과 상태 변경을 섞지 않습니다

parser가 다음과 같은 일을 해서는 안 된다는 원칙을 둘 수 있습니다.

```text
문자열 일부 읽음
→ Store 변경
→ 나머지 문자열 검사
→ 문법 오류 발견
```

예를 들어

```text
PUT name alice unexpected
```

가 잘못된 요청인데 `name=alice`가 먼저 저장되어 버리면 입력 검증 실패가 프로그램 상태까지 변경합니다.

보통은 다음 순서가 안전합니다.

```text
전체 입력 검사
→ 실행 가능한 Request 생성
→ 상태 변경
```

즉 parser는 가능하면 side effect 없이 끝나는 것이 좋습니다.

## Response는 실행 결과와 출력 표현을 분리합니다

handler가 바로 다음 문자열을 반환한다고 가정해 보겠습니다.

```text
"ERR key not found\n"
```

그러면 handler가 다음 내용을 모두 알아야 합니다.

* 실행 결과가 실패라는 사실
* 오류 종류
* 문자열 형식
* `"ERR"`라는 출력 규칙
* 줄바꿈 규칙

대신 내부 결과를 값으로 표현할 수 있습니다.

```cpp
struct Response {
    enum Code {
        Ok,
        Value,
        NotFound,
        Error
    };

    Code code;
    std::string value;
};
```

handler는 의미만 결정합니다.

```text
code  = NotFound
value = ""
```

formatter가 외부 표현을 결정합니다.

```text
ERR key not found
```

이렇게 하면 출력 형식을 바꾸더라도 데이터 저장 규칙이나 handler 로직을 수정할 필요가 줄어듭니다.

다만 모든 프로그램에 `Response` 클래스가 필요한 것은 아닙니다. 실행 결과가 단순한 `bool`이나 정수 하나로 충분하다면 별도 타입을 만드는 것이 오히려 과할 수 있습니다.

## 상태는 명확한 소유자에게 둡니다

```cpp
class Store {
public:
    void putNew(
        const std::string &key,
        const std::string &value
    );

    bool get(
        const std::string &key,
        std::string &value
    ) const;

private:
    std::map<std::string, TextBuffer *> data_;
};
```

`Store`가 `data_`를 직접 보유한다면 저장 규칙도 가능한 한 `Store` 내부에 모읍니다.

예:

* key 중복 허용 여부
* 최대 항목 수
* 값 교체 규칙
* `TextBuffer` 생성과 삭제
* 항목 제거

handler가 다음처럼 직접 map을 수정하기 시작하면,

```cpp
store.data_[key] = value;
```

각 handler마다 검사 순서가 달라질 수 있습니다.

예:

```text
PutHandler
capacity 검사 → 중복 검사

ReplaceHandler
중복 검사 → capacity 검사

DeleteHandler
다른 규칙 적용
```

저장소가 지켜야 하는 규칙은 저장소 함수가 직접 지키게 하는 편이 일관성이 좋습니다.

### raw pointer를 저장한다면 소유권까지 명확해야 합니다

위의

```cpp
std::map<std::string, TextBuffer *> data_;
```

만으로는 `Store`가 `TextBuffer`를 소유하는지 알 수 없습니다.

`Store`가 소유한다고 정했다면 다음까지 함께 설계해야 합니다.

* 소멸자에서 모든 `TextBuffer` 삭제
* 항목 삭제 시 해당 pointer 삭제
* 삽입 실패 시 새 객체 정리
* `Store` 복사를 허용할지 결정

단순히 "상태는 Store가 관리한다"만으로는 메모리 소유까지 자동으로 해결되지 않습니다.

## handler는 실행 규칙을 담당합니다

검증된 요청을 실제 상태 연산으로 바꾸는 코드를 handler에 둘 수 있습니다.

```cpp
Response PutHandler::handle(
    const Request &request,
    Store &store
) const {
    store.putNew(
        request.arguments[0],
        request.arguments[1]
    );

    return Response(Response::Ok);
}
```

여기서는 다음 역할만 수행합니다.

```text
PUT 요청
→ Store의 putNew() 실행
→ 성공 결과 생성
```

handler가 다시 다음처럼 명령 종류를 판별하기 시작하면,

```cpp
if (request.command == "PUT") {
    ...
}
else if (request.command == "GET") {
    ...
}
```

handler를 분리한 의미가 줄어듭니다.

어떤 handler를 호출할지는 Router가 결정하게 할 수 있습니다.

```text
"PUT" → PutHandler
"GET" → GetHandler
```

## parser와 handler가 같은 검증을 반복해야 하는 것은 아닙니다

예를 들어 `PUT`은 정확히 두 인자를 요구한다고 가정합니다.

parser에서 이미 이를 보장한다면 handler가 다시

```cpp
if (request.arguments.size() != 2)
```

를 확인할 필요는 없습니다.

하지만 그 보장이 불분명하다면 handler가 `arguments[1]`에 바로 접근하는 것은 위험합니다.

따라서 다음 중 하나를 분명하게 정해야 합니다.

### 방식 1

parser가 모든 구조 검증을 담당합니다.

```text
parse 성공
=
handler가 안전하게 사용할 수 있는 Request
```

### 방식 2

parser는 tokenization만 하고 handler가 명령별 검증을 담당합니다.

```text
parser
→ 일반 Request

handler
→ 명령별 인자 검증
```

둘 다 가능하지만 문서와 코드가 서로 다른 가정을 해서는 안 됩니다.

## 조립 위치

`main()`은 모든 세부 구현을 수행하는 곳이라기보다 프로그램의 주요 객체를 연결하는 위치로 사용할 수 있습니다.

```cpp
Store store(capacity);

Router router;
RequestParser parser;
ResponseFormatter formatter;
```

이런 위치를 흔히 composition root라고 부르기도 합니다.

중요한 점은 용어가 아니라 여기에서 객체의 실제 관계가 보인다는 것입니다.

예:

```text
Store는 main이 소유
Router는 handler를 소유
Handler는 Store를 인자로 받음
Parser는 Store를 모름
Formatter는 Store를 모름
```

메인 반복문은 대략 다음처럼 연결할 수 있습니다.

```text
read
→ parse
→ route
→ handle
→ format
→ write
```

## 예외를 외부 결과로 바꾸는 위치

내부 코드가 예외를 사용한다면 외부 인터페이스로 나가기 전에 이를 응답으로 변환해야 할 수 있습니다.

예:

```cpp
try {
    Request request = parser.parse(line);
    Response response = router.dispatch(request, store);

    std::cout << formatter.format(response);
}
catch (const ParseError &error) {
    std::cout << formatter.formatParseError(error);
}
```

이렇게 하면 `Store`와 parser가 직접 `std::cout`을 사용할 필요가 없습니다.

따라서 테스트도 다음처럼 할 수 있습니다.

```text
입력값 전달
→ 반환값 또는 예외 확인
```

stdout을 capture해서 문자열을 분석하는 테스트에 덜 의존하게 됩니다.

## 출력하지 않는 것과 오류를 무시하는 것은 다릅니다

`Store`가 stdout을 직접 사용하지 않는다고 해서 오류 정보를 버려도 된다는 뜻은 아닙니다.

예를 들어 중복 key라면 다음 중 하나로 결과를 전달해야 합니다.

* 반환값
* enum
* 예외
* 별도 결과 타입

핵심은 오류를 **발견한 코드와 표시하는 코드를 분리하는 것**입니다.

## 의존성을 숨기지 않습니다

전역 변수 대신 필요한 객체를 생성자나 함수 인자로 전달합니다.

```cpp
class CommandService {
public:
    explicit CommandService(Store &store)
        : store_(store) {
    }

private:
    Store &store_;
};
```

여기서 `store_`는 reference 멤버입니다.

따라서 `CommandService`는 `Store`를 소유하지 않습니다.

그리고 다음 조건이 필요합니다.

```text
CommandService가 살아 있는 동안
Store도 살아 있어야 한다.
```

예:

```cpp
Store store;
CommandService service(store);
```

지역 객체는 선언의 역순으로 소멸하므로 다음 순서로 정리됩니다.

```text
service 소멸
→ store 소멸
```

이 경우 `service`가 살아 있는 동안 `store`도 존재합니다.

반대로 수명 관계가 복잡해지면 단순히 생성 순서만 보고 안전하다고 판단하면 안 됩니다. 객체가 pointer로 밖에 저장되거나 더 오래 살아가는 경우 실제 소유 관계를 따로 확인해야 합니다.

## 클래스 수를 늘리는 것이 목표가 아닙니다

다음과 같은 클래스가 있다고 가정합니다.

```cpp
class UpperCaseFormatter {
public:
    std::string format(const std::string &value) const;
};
```

프로그램 전체에서 한 곳에서만 사용하고 아무 상태도 없으며 단순 함수 하나로 충분하다면

```cpp
std::string formatUpperCase(const std::string &value);
```

가 더 나을 수 있습니다.

분리 여부를 판단할 때 다음 질문이 유용합니다.

* 독립적으로 유지해야 하는 상태가 있는가?
* 별도로 테스트할 의미 있는 실패 조건이 있는가?
* 다른 부분과 독립적으로 변경될 가능성이 있는가?
* 자원 소유와 수명을 따로 관리해야 하는가?
* 해당 코드가 다른 세부 구현을 지나치게 많이 알아야 하는가?

반면 다음 기준은 절대적인 기준이 아닙니다.

> 호출자가 둘 이상인가?

호출자가 하나뿐이어도 복잡한 자원 소유나 독립적인 규칙이 있다면 타입으로 분리할 가치가 있습니다.

반대로 호출자가 여러 개여도 단순한 stateless 함수라면 클래스가 필요하지 않을 수 있습니다.

## 자주 놓치는 문제

* parser가 입력 검증 도중 Store를 변경합니다.
* parser와 Router가 어떤 명령을 검증하는지 역할이 불분명합니다.
* "`Request` 타입이므로 유효하다"고 생각하지만 실제로는 아무 값이나 넣을 수 있습니다.
* handler가 직접 stdout에 출력합니다.
* 여러 handler가 Store 내부 container를 직접 수정합니다.
* `Manager` 클래스 하나가 parsing, 저장, 네트워크, logging까지 모두 수행합니다.
* 클래스를 많이 만들었지만 각 클래스가 다른 클래스의 내부 구현을 전부 알아야 합니다.
* raw pointer가 들어 있는 Store의 실제 소유자를 정하지 않습니다.
* 전역 singleton으로 객체 관계와 소멸 순서를 숨깁니다.
* 단순 함수로 충분한 작업까지 모두 클래스로 만듭니다.

## 완료 기준

* 입력 문자열 해석과 프로그램 상태 변경이 분리되어 있습니다.
* `Request`가 어느 수준까지 검증된 값인지 설명할 수 있습니다.
* 명령 지원 여부를 parser와 Router 중 누가 판단하는지 정해져 있습니다.
* 실제 데이터를 보유하고 변경 규칙을 적용하는 타입이 명확합니다.
* handler가 Store 내부 container를 직접 수정하지 않습니다.
* 출력 형식을 변경해도 저장 규칙을 수정할 필요가 없습니다.
* `main()`에서 주요 객체의 생성과 수명 관계를 설명할 수 있습니다.
* reference 멤버가 소유를 의미하지 않는다는 것을 설명할 수 있습니다.
* 각 타입을 분리한 구체적인 이유가 있습니다.