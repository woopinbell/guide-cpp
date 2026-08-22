# C++98 template·iterator·STL

## 목표

C++98 문법으로 함수와 클래스 template을 작성하고, iterator의 유효 범위와 const 여부를 지킵니다. 표준 container가 요구하는 복사·대입·소멸 동작을 이해합니다.

## 함수 template

```cpp
template <class T>
const T &maximum(const T &left, const T &right) {
    return left < right ? right : left;
}
```

사용한 연산은 `operator<`입니다. 모든 타입에 동작한다고 표현하지 말고 어떤 연산과 의미를 요구하는지 적습니다.

C++98에는 concept이 없으므로 잘못된 타입은 함수 본문을 instantiation할 때 긴 오류를 낼 수 있습니다. public template은 작게 유지하고 필요한 연산을 README나 주석에 명시합니다.

## 클래스 template 정의 위치

compiler가 실제 타입으로 코드를 만들 때 정의를 볼 수 있어야 합니다. 일반적으로 template 선언과 정의를 header에 둡니다.

```cpp
template <class T>
class Array {
public:
    explicit Array(std::size_t size);
};

template <class T>
Array<T>::Array(std::size_t size)
    : data_(size ? new T[size] : 0), size_(size) {}
```

특정 타입만 지원한다면 `.cpp`에서 명시적으로 instantiation할 수 있지만 지원 목록이 고정됩니다.

## dependent name과 `typename`

```cpp
template <class Container>
void print(const Container &items) {
    typename Container::const_iterator it = items.begin();
    for (; it != items.end(); ++it)
        std::cout << *it << '\n';
}
```

`Container::const_iterator`가 타입이라는 사실은 template 정의 시점에 확정되지 않으므로 `typename`이 필요합니다.

## iterator 범위

표준 범위는 `[begin, end)`입니다. `end()`는 역참조하지 않습니다.

```cpp
for (Iterator it = first; it != last; ++it)
    function(*it);
```

빈 container에서는 `begin() == end()`입니다. raw pointer iterator를 사용하는 직접 구현에서는 `0 + 0` 같은 pointer 산술을 피하도록 빈 범위를 처리합니다.

## mutable과 const iterator

```cpp
Array<int>::iterator it = values.begin();
Array<int>::const_iterator read = view.begin();
```

const container에서 mutable iterator를 반환하면 caller가 const를 우회해 값을 바꿀 수 있습니다. `begin()`과 `end()`의 const overload를 제공합니다.

## iterator 무효화

container 변경 뒤 기존 iterator를 계속 쓸 수 있는지 확인합니다.

- `vector` 재할당: 모든 pointer, reference, iterator 무효화
- `vector` 중간 insert/erase: 변경 위치 이후 무효화
- `map` insert: 기존 iterator 유지
- `map` erase: 지운 원소 iterator만 무효화
- `deque`: 연산에 따라 규칙이 복잡하므로 문서 확인

무효화 규칙은 container 선택과 함수 설계에 영향을 줍니다.

## STL container의 값 요구

C++98 container는 원소를 복사합니다. 소유 자원을 가진 값 타입이 깊은 복사와 안전한 대입을 제공해야 합니다.

`std::map<std::string, Handler*>`처럼 pointer를 저장하면 container는 pointer만 복사하고 객체를 삭제하지 않습니다. 누가 pointee를 해제하는지 별도 코드가 필요합니다.

## algorithm

```cpp
std::stable_sort(records.begin(), records.end(), RecordLess());
```

비교 함수는 strict weak ordering을 지켜야 합니다. 같은 원소에 `true`를 반환하거나 비교가 순환하면 결과가 정의되지 않습니다.

`std::find`, `std::find_if`, `std::copy`, `std::transform`, `std::accumulate` 같은 algorithm을 먼저 확인합니다. 다만 상태 변화와 여러 종료 조건이 얽힌 경우에는 명시적인 loop가 더 읽기 쉽습니다.

## function object

C++98에는 lambda가 없습니다.

```cpp
struct RecordLess {
    bool operator()(const Record &left, const Record &right) const {
        return left.value < right.value;
    }
};
```

비교 기준을 상태로 받아야 한다면 생성자와 멤버를 둡니다.

## specialization

전체 template 동작이 특정 타입에서 달라야 할 때 specialization을 사용할 수 있습니다. 단순 overload로 해결할 수 있다면 overload가 더 읽기 쉽습니다.

partial specialization은 class template에만 가능하며 function template에는 직접 적용할 수 없습니다.

## 자주 놓치는 문제

- template 정의를 `.cpp`에 두고 link 오류가 납니다.
- dependent type 앞의 `typename`을 빠뜨립니다.
- const container에서 mutable iterator를 반환합니다.
- vector 재할당 뒤 이전 pointer를 사용합니다.
- pointer container가 객체 수명까지 관리한다고 생각합니다.
- 비교 함수가 같은 값에 `true`를 반환합니다.

## 완료 기준

- 함수·클래스 template을 C++98 문법으로 작성합니다.
- dependent name에 `typename`이 필요한 이유를 설명합니다.
- 반열린 iterator 범위와 빈 범위를 안전하게 처리합니다.
- mutable/const iterator를 구분합니다.
- 주요 STL container의 iterator 무효화 규칙을 확인합니다.
- container에 저장할 값의 복사·소멸 요구를 처리합니다.
