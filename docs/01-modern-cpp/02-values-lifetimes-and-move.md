# 값·수명·복사·이동

## 목표

코드를 읽을 때 pointer인지 값인지만 확인하지 않습니다. 객체가 언제 만들어지고 사라지는지, 함수가 값을 소유하는지 빌려 쓰는지, 복사와 이동 뒤 어느 객체가 무엇을 보유하는지 추적합니다.

## 객체 수명과 저장 위치

stack과 heap은 저장 위치를 설명하지만 수명 전체를 설명하지는 못합니다.

```cpp
std::string make_name() {
    std::string name{"worker"};
    return name;
}
```

지역 변수 `name`은 함수가 끝날 때 사라집니다. 반환값은 caller가 소유하는 별도 객체가 되며 compiler가 copy elision을 적용할 수 있습니다.

반대로 heap에 있는 객체도 소유자가 해제하면 즉시 수명이 끝납니다.

```cpp
auto owner = std::make_unique<std::string>("worker");
std::string* observer = owner.get();
owner.reset();
// observer에 주소 값은 남아 있지만 가리키던 객체는 사라졌습니다.
```

주소를 가지고 있다는 사실과 객체가 살아 있다는 사실은 다릅니다.

## 초기화와 대입

```cpp
Task first{"compile"};      // 새 객체를 초기화합니다.
Task second = first;         // 새 객체를 복사 생성합니다.
second = first;              // 이미 존재하는 객체에 대입합니다.
```

생성자와 대입 연산자는 처리해야 할 실패가 다를 수 있습니다. 대입은 기존 값이 이미 있으므로 새 값을 준비하지 못했을 때 이전 값을 유지할지 정해야 합니다.

## 값, 참조, pointer, view

- 값은 함수나 객체가 독립적으로 소유합니다.
- `T&`와 `const T&`는 호출 중 다른 객체를 빌려 씁니다.
- `T*`는 null 가능성과 소유 여부를 문서나 타입으로 구분해야 합니다.
- `std::string_view`, `std::span`, iterator는 원본을 소유하지 않습니다.

```cpp
std::string_view title(const Task& task) {
    return task.title();
}
```

반환한 view는 `task`와 내부 문자열이 살아 있는 동안에만 사용할 수 있습니다. 임시 객체의 내부 데이터를 가리키는 view를 저장하면 댕글링이 됩니다.

## 복사 의미

값 타입은 복사본을 수정해도 원본이 바뀌지 않는 것이 보통입니다.

```cpp
Task a{1, "compile"};
Task b = a;
b.rename("test");
```

raw owning pointer를 멤버로 두고 compiler가 만든 복사를 사용하면 주소만 복사되어 double delete가 생길 수 있습니다. 직접 소유할 필요가 없다면 `std::string`, `std::vector`, smart pointer처럼 이미 수명을 관리하는 타입을 사용합니다.

## 이동 의미

이동은 객체를 파괴하는 연산이 아닙니다. 자원을 새 객체로 넘기고 원본을 소멸 가능한 유효 상태로 남깁니다.

```cpp
std::vector<int> source{1, 2, 3};
std::vector<int> target = std::move(source);
```

이후 `source`에는 `empty()`, 대입, 소멸처럼 타입이 보장하는 연산을 사용할 수 있습니다. 이전 원소가 남아 있다고 가정하면 안 됩니다.

이동 전용 타입은 복사 의미가 자연스럽지 않은 자원에 사용합니다.

```cpp
class UniqueFile {
public:
    UniqueFile(const UniqueFile&) = delete;
    UniqueFile& operator=(const UniqueFile&) = delete;
    UniqueFile(UniqueFile&&) noexcept;
    UniqueFile& operator=(UniqueFile&&) noexcept;
};
```

컨테이너가 재배치할 때 이동을 선택할 수 있도록, 실제로 예외가 나지 않는 이동은 `noexcept`로 표시합니다.

## 함수 signature로 소유 여부를 드러냅니다

```cpp
void inspect(const Task& task);                 // 호출 중 읽기
void update(Task& task);                        // 호출 중 변경
void consume(Task task);                        // 독립 값 소유
void install(std::unique_ptr<Task> task);        // 유일 소유권 이전
Task* find(TaskId id);                          // nullable 비소유 결과
```

모든 큰 객체를 `const&`로 받는 것이 정답은 아닙니다. 함수가 값을 저장한다면 값으로 받아 이동하는 편이 수명 전제를 줄일 수 있습니다.

## copy elision과 `std::move`

지역 변수를 반환할 때 무조건 `std::move`하지 않습니다.

```cpp
Task make_task() {
    Task task{/* ... */};
    return task;
}
```

`return std::move(task);`는 일부 copy elision 기회를 막을 수 있습니다. 일반적인 값 반환을 먼저 사용합니다.

## 자주 놓치는 문제

- 함수가 끝난 뒤 사용할 참조를 지역 객체에서 반환합니다.
- container 재할당 뒤 이전 iterator나 pointer를 계속 사용합니다.
- 이동한 객체가 이전 값을 그대로 가진다고 검사합니다.
- observer pointer를 owner보다 오래 저장합니다.
- callback이 참조로 잡은 지역 변수가 callback보다 먼저 사라집니다.

## 완료 기준

- 객체 수명과 메모리 주소의 존재를 구분합니다.
- 초기화, 복사 생성, 이동 생성, 대입의 차이를 설명합니다.
- 함수 signature에서 소유·변경·관찰 의도를 드러냅니다.
- 비소유 view의 원본 수명 조건을 적습니다.
- 이동 뒤 원본에 허용되는 연산과 허용되지 않는 가정을 구분합니다.
