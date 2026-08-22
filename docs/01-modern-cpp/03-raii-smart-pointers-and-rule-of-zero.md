# RAII·smart pointer·Rule of Zero

## 목표

파일, socket, mutex, 메모리처럼 반드시 정리해야 하는 자원을 객체 수명에 연결합니다. 정상 return뿐 아니라 예외와 중간 실패에서도 같은 정리 코드가 실행되도록 만듭니다.

## RAII가 해결하는 문제

다음 코드는 두 번째 작업이 실패하면 첫 번째 자원을 직접 정리해야 합니다.

```cpp
FILE* file = std::fopen(path, "rb");
if (file == nullptr)
    return error;

char* buffer = new char[size];
// 이후 모든 return과 예외에서 file과 buffer를 정리해야 합니다.
```

RAII 타입은 생성 중 자원을 얻고 소멸자에서 해제합니다.

```cpp
std::ifstream input{path, std::ios::binary};
std::vector<char> buffer(size);
```

함수가 어떤 경로로 끝나더라도 이미 생성된 지역 객체의 소멸자는 호출됩니다.

## 소멸자에서 실패를 다루는 방법

소멸자는 예외를 던지지 않는 것이 기본입니다. 이미 다른 예외로 stack unwinding 중인데 소멸자까지 예외를 던지면 `std::terminate`가 호출될 수 있습니다.

정리 연산의 오류를 반드시 caller에게 알려야 한다면 소멸자와 별도의 명시적 함수를 둡니다.

```cpp
class OutputFile {
public:
    void finish();       // flush와 close 실패를 보고합니다.
    ~OutputFile();       // 남은 자원을 정리하지만 예외를 던지지 않습니다.
};
```

## `unique_ptr`

한 객체만 자원을 소유한다면 `std::unique_ptr`을 먼저 고려합니다.

```cpp
auto task = std::make_unique<Task>(id, name);
queue.push_back(std::move(task));
```

복사는 금지되고 이동으로만 소유권을 넘깁니다. raw pointer가 필요한 API에는 `get()`으로 잠시 관찰 포인터를 전달할 수 있지만, 그 API가 포인터를 저장하거나 삭제하지 않는다는 조건이 필요합니다.

C API 자원에는 custom deleter를 사용할 수 있습니다.

```cpp
struct FileCloser {
    void operator()(std::FILE* file) const noexcept {
        if (file != nullptr)
            std::fclose(file);
    }
};

using FilePtr = std::unique_ptr<std::FILE, FileCloser>;
```

## `shared_ptr`는 공유 수명을 뜻합니다

`std::shared_ptr`는 편한 pointer가 아니라 여러 소유자가 같은 객체 수명을 연장한다는 뜻입니다. 누가 객체를 끝내야 하는지 명확하지 않은데 습관적으로 사용하면 종료 시점을 추적하기 어려워집니다.

서로를 `shared_ptr`로 보관하면 cycle이 생길 수 있습니다. 한쪽이 소유하지 않는 연결이라면 `std::weak_ptr`를 사용합니다.

## lock도 자원입니다

```cpp
void update(State& state) {
    std::lock_guard lock{state.mutex};
    state.value += 1;
}
```

수동 `lock()`과 `unlock()` 사이에서 예외가 나면 mutex가 잠긴 채 남을 수 있습니다. `lock_guard`나 `unique_lock`을 사용하면 scope가 끝날 때 unlock됩니다.

## Rule of Zero

가능하면 소멸자, 복사 생성자, 이동 생성자, 대입 연산자를 직접 작성하지 않습니다. 자원을 관리하는 멤버 타입에게 맡깁니다.

```cpp
struct Job {
    JobId id;
    std::string name;
    std::vector<std::string> tags;
};
```

모든 멤버가 올바른 값 의미를 제공하므로 compiler가 만든 특수 멤버 함수로 충분합니다.

직접 자원을 소유하는 타입은 소유 방식에 맞게 복사·이동을 명시합니다.

- 복사 가능한 값: 깊은 복사 또는 자원 공유 규칙을 정의합니다.
- 유일 자원: 복사를 금지하고 이동을 구현합니다.
- 고정 주소가 필요한 동기화 객체: 복사와 이동을 모두 금지할 수 있습니다.

## 생성자 실패

생성자 본문에서 예외가 발생하면 그 객체의 소멸자는 호출되지 않습니다. 대신 이미 생성된 멤버와 기반 클래스의 소멸자는 호출됩니다. 그래서 raw 자원을 얻은 뒤 멤버에 넘기기 전에 예외가 나지 않도록 주의해야 합니다.

가능하면 자원을 얻는 즉시 RAII 객체에 넣습니다.

## 자주 놓치는 문제

- 자원 handle을 가진 타입을 compiler 기본 복사에 맡깁니다.
- 이동 대입 전에 현재 자원을 정리하지 않습니다.
- 이동 뒤 원본 handle을 빈 값으로 바꾸지 않아 두 객체가 같은 자원을 해제합니다.
- 소멸자에서 I/O 실패를 예외로 던집니다.
- `shared_ptr`를 소유 관계가 아닌 단순 전달 목적으로 사용합니다.
- mutex를 잠근 상태에서 외부 callback을 호출합니다.

## 완료 기준

- 각 자원의 소유자와 해제 지점을 타입으로 표현합니다.
- 유일 소유와 공유 소유를 구분합니다.
- 소멸자에서 보고할 수 없는 오류와 명시적 종료 함수가 필요한 오류를 구분합니다.
- Rule of Zero를 우선하고 직접 특수 멤버 함수를 작성해야 할 이유를 설명합니다.
- 이동 전용 타입이 이동 뒤에도 안전하게 소멸되는지 확인합니다.
