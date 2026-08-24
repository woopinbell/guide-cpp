# C++98 template·iterator·STL

## 목표

C++98 문법으로 함수 template과 클래스 template을 작성하고 다음을 이해합니다.

* template이 언제 실제 코드로 만들어지는가
* 왜 정의가 호출 위치에서 보여야 하는가
* dependent name 앞에 왜 `typename`이 필요한가
* iterator의 `[begin, end)` 범위가 무엇을 의미하는가
* container 변경 시 iterator와 reference가 언제 무효화되는가
* STL container가 저장하는 값에 어떤 복사와 소멸 동작을 요구하는가

C++98에는 concept, range, lambda 같은 Modern C++ 기능이 없으므로 template의 요구 사항을 코드와 문서에서 더 명시적으로 관리해야 합니다.

## 함수 template

```cpp
template <class T>
const T &maximum(
    const T &left,
    const T &right
) {
    return left < right ? right : left;
}
```

이 함수가 임의의 모든 `T`에 동작하는 것은 아닙니다.

함수 본문에서 실제로 요구하는 것은:

```cpp
left < right
```

라는 표현이 유효해야 한다는 것입니다.

또한 결과를 `const T &`로 반환하므로 두 인자가 함수 호출 뒤에도 살아 있다는 일반적인 reference 수명 규칙도 적용됩니다.

## template의 요구 사항은 사용한 표현에서 생깁니다

예를 들어 다음 template은:

```cpp
template <class T>
void printTwice(const T &value) {
    std::cout << value << value;
}
```

`T`가 다음 표현을 지원해야 합니다.

```cpp
std::cout << value
```

C++98에는 다음과 같은 문법이 없습니다.

```cpp
template <Printable T>
```

따라서 요구 사항은 template definition에 사용된 연산으로 암묵적으로 결정됩니다.

public template이라면 다음을 문서화하는 것이 좋습니다.

```text
T 요구 사항:
- operator< 지원
- copy construction 가능
- 함수 실행 동안 인자가 유효함
```

단, 실제로 사용하지 않는 요구까지 임의로 추가하지 않습니다.

## instantiation

template 자체는 특정 타입의 완성된 함수가 아닙니다.

예:

```cpp
template <class T>
T add(T left, T right) {
    return left + right;
}
```

다음 코드가 사용되면:

```cpp
int value = add(1, 2);
```

compiler가 필요한 시점에 대략 `T = int`인 버전을 만듭니다.

이를 template instantiation이라고 합니다.

다른 타입을 사용하면:

```cpp
double value = add(1.5, 2.5);
```

`double`에 대한 별도 instantiation이 필요해집니다.

## 오류가 template 정의 시점에 모두 발견되는 것은 아닙니다

다음 template 자체는 선언할 수 있습니다.

```cpp
template <class T>
T add(T left, T right) {
    return left + right;
}
```

그러나 `operator+`가 없는 타입을 실제로 사용하면:

```cpp
Widget result = add(a, b);
```

해당 instantiation 과정에서 오류가 발생합니다.

C++98에는 concept으로 요구 사항을 깔끔하게 제한하는 기능이 없으므로 compiler 오류가 template 내부 여러 단계까지 이어져 길어질 수 있습니다.

따라서 복잡한 public template일수록 요구하는 연산을 최소화하고 명확히 하는 것이 중요합니다.

## 클래스 template 정의 위치

template을 특정 타입으로 instantiation하려면 compiler가 해당 정의를 볼 수 있어야 합니다.

따라서 일반적으로 template 선언과 정의를 header에 둡니다.

```cpp
template <class T>
class Array {
public:
    explicit Array(std::size_t size);

private:
    T *data_;
    std::size_t size_;
};

template <class T>
Array<T>::Array(std::size_t size)
    : data_(size ? new T[size] : 0),
      size_(size) {
}
```

다른 `.cpp`에서 다음 코드를 컴파일한다고 가정합니다.

```cpp
Array<int> values(10);
```

compiler는 이 시점에 `Array<int>::Array()`의 정의를 볼 수 있어야 일반적으로 필요한 코드를 만들 수 있습니다.

## template 선언만 header에 두면 왜 문제가 생기는가

헤더:

```cpp
template <class T>
class Array {
public:
    explicit Array(std::size_t size);
};
```

`.cpp`:

```cpp
template <class T>
Array<T>::Array(std::size_t size) {
}
```

다른 번역 단위에서:

```cpp
Array<int> values(10);
```

를 사용하면 그 번역 단위는 `Array<int>` 생성자의 정의를 보지 못할 수 있습니다.

일반 함수처럼 나중에 linker가 알아서 모든 template 타입을 만들어 주는 것이 아닙니다.

이 때문에 흔히 template 정의를 header에 둡니다.

## 명시적 instantiation

지원하는 타입이 고정되어 있다면 구현을 `.cpp`에 두고 특정 타입만 명시적으로 instantiation할 수 있습니다.

개념적으로:

```cpp
template class Array<int>;
template class Array<double>;
```

처럼 필요한 타입을 미리 지정합니다.

이 경우 외부에서 새로운 타입을 마음대로 사용할 수 있는 일반 template library라기보다 지원 타입이 제한된 구현이 됩니다.

즉 장점과 제한이 있습니다.

```text
장점:
구현 일부를 .cpp에 둘 수 있음

제한:
지원하는 template argument 목록을 미리 알아야 함
```

## dependent name

다음 template을 봅니다.

```cpp
template <class Container>
void print(const Container &items) {
    Container::const_iterator it =
        items.begin();
}
```

`Container`가 template parameter이므로 compiler는 template 정의를 처음 읽는 시점에

```cpp
Container::const_iterator
```

가 타입 이름인지 알 수 없습니다.

`Container`의 실제 타입은 나중에 결정되기 때문입니다.

따라서 타입이라는 사실을 명시합니다.

```cpp
template <class Container>
void print(const Container &items) {
    typename Container::const_iterator it =
        items.begin();

    for (; it != items.end(); ++it)
        std::cout << *it << '\n';
}
```

여기서 `typename`은:

> `Container::const_iterator`를 값이나 static member 이름이 아니라 타입 이름으로 해석하라.

는 뜻입니다.

## `typename`이 필요한 이유

예를 들어 어떤 타입에는 이론적으로 다음과 같은 이름이 있을 수도 있습니다.

```cpp
Container::something
```

compiler는 `Container`가 실제로 무엇인지 아직 모르므로 `something`이

* nested type인지
* static data member인지
* 다른 종류의 이름인지

정의 시점에 확정할 수 없습니다.

dependent qualified name을 타입으로 사용하려면 `typename`으로 알려 줍니다.

단순히 `::`가 들어간 모든 타입 앞에 `typename`을 붙이는 규칙은 아닙니다. template parameter에 의존하는 이름인지가 중요합니다.

## iterator는 위치를 표현하는 객체입니다

iterator는 container의 원소를 순회할 수 있게 하는 추상화입니다.

일반적인 형태:

```cpp
Iterator it = container.begin();

while (it != container.end()) {
    use(*it);
    ++it;
}
```

iterator의 정확한 내부 구현은 container마다 다릅니다.

* `vector` iterator는 pointer와 유사할 수 있음
* `list` iterator는 node 연결을 따라갈 수 있음
* `map` iterator는 tree node를 순회할 수 있음

따라서 모든 iterator를 raw pointer라고 생각하면 안 됩니다.

## 표준 범위 `[begin, end)`

STL에서 일반적인 범위는 반열린 구간입니다.

```text
[begin, end)
```

뜻:

```text
begin은 첫 원소를 가리킴
end는 마지막 원소 다음 위치를 나타냄
```

따라서:

```cpp
for (Iterator it = first;
     it != last;
     ++it) {
    function(*it);
}
```

에서 `last` 자체는 처리하지 않습니다.

`end()`는 유효한 원소를 가리키지 않으므로 역참조하면 안 됩니다.

잘못된 코드:

```cpp
*container.end();
```

## 빈 범위

빈 container에서는:

```cpp
container.begin()
==
container.end()
```

입니다.

따라서 일반적인 loop는 별도 empty 검사 없이도 실행되지 않습니다.

```cpp
for (Iterator it = container.begin();
     it != container.end();
     ++it) {
    ...
}
```

원소가 없으면 첫 조건부터 false입니다.

## 직접 만든 raw-pointer iterator에서는 빈 상태 표현을 주의합니다

직접 동적 배열을 구현하면서 다음 상태를 사용한다고 가정합니다.

```cpp
data_ = 0;
size_ = 0;
```

그때:

```cpp
return data_ + size_;
```

처럼 `0 + 0`이라는 pointer arithmetic에 의존하는 구현은 피하는 편이 안전합니다.

예를 들어:

```cpp
T *end() {
    return size_ == 0
        ? data_
        : data_ + size_;
}
```

처럼 빈 상태를 따로 고려하거나, 애초에 빈 배열에서도 iterator 표현이 유효하도록 저장 구조를 설계할 수 있습니다.

표준 container의 `begin()`/`end()` 계약과 직접 구현한 null pointer 연산을 같은 것으로 생각하면 안 됩니다.

## mutable iterator와 const iterator

수정 가능한 container에서는 mutable iterator를 사용할 수 있습니다.

```cpp
Array<int>::iterator it =
    values.begin();

*it = 42;
```

const container에서는 원소를 변경할 수 없어야 합니다.

```cpp
const Array<int> &view = values;

Array<int>::const_iterator it =
    view.begin();
```

따라서 직접 container를 만든다면 일반적으로 `begin()`과 `end()`에 const/non-const overload를 제공합니다.

```cpp
iterator begin();
const_iterator begin() const;

iterator end();
const_iterator end() const;
```

## const member function에서 mutable iterator를 반환하면 안 됩니다

잘못된 형태:

```cpp
iterator begin() const;
```

이렇게 하면:

```cpp
const Array<int> &view = values;
```

를 통해 mutable iterator를 얻어 원소를 변경할 가능성이 생깁니다.

const container의 논리적 읽기 전용 성질을 깨뜨릴 수 있습니다.

따라서:

```cpp
const_iterator begin() const;
```

를 반환해야 합니다.

## iterator 자체의 const와 `const_iterator`는 다릅니다

다음 둘은 다른 개념입니다.

```cpp
const Iterator it;
```

```cpp
Container::const_iterator it;
```

첫 번째는 iterator 객체 자체를 변경할 수 없다는 뜻일 수 있습니다.

즉 `++it`가 불가능해질 수 있습니다.

두 번째는 iterator를 움직일 수 있지만 그 iterator를 통해 원소를 수정할 수 없도록 하는 타입입니다.

보통 순회용 읽기 iterator에 필요한 것은 후자입니다.

## iterator category

모든 iterator가 같은 연산을 지원하지 않습니다.

대략 다음과 같은 차이가 있습니다.

```text
input iterator
forward iterator
bidirectional iterator
random access iterator
```

예를 들어 `vector` iterator에서는 다음이 가능합니다.

```cpp
it + 5
last - first
it < last
```

하지만 `list` iterator에서는 일반적으로 이런 random-access 연산을 할 수 없습니다.

`map` iterator는 양방향 이동은 가능하지만 임의 위치 덧셈은 지원하지 않습니다.

따라서 algorithm이 요구하는 iterator 종류를 확인해야 합니다.

## `std::stable_sort()`는 아무 iterator에나 사용할 수 없습니다

다음 코드는 `vector`처럼 random access iterator를 제공하는 container에서는 사용할 수 있습니다.

```cpp
std::stable_sort(
    records.begin(),
    records.end(),
    RecordLess()
);
```

하지만 `std::list` iterator에는 사용할 수 없습니다.

`stable_sort`는 random access iterator를 요구하기 때문입니다.

따라서:

```text
algorithm이 존재한다
≠
모든 container에 사용할 수 있다
```

입니다.

algorithm의 value 요구 사항뿐 아니라 iterator category도 확인해야 합니다.

## iterator 무효화

container를 변경하면 기존 iterator, pointer, reference가 계속 유효한지 확인해야 합니다.

이 규칙은 container마다 다릅니다.

### `vector`

재할당이 발생하면 기존 element를 새 저장 공간으로 옮겨야 하므로 일반적으로:

```text
기존 iterator
기존 pointer
기존 reference
```

가 모두 무효화됩니다.

예:

```cpp
std::vector<int> values;

values.push_back(1);

int *first = &values[0];

values.push_back(2);
```

두 번째 `push_back()`이 capacity를 늘리기 위해 재할당했다면 `first`는 더 이상 사용할 수 없습니다.

### `vector` insert

중간 삽입에서 재할당이 발생하면 기존 iterator, pointer, reference가 모두 무효화될 수 있습니다.

재할당이 발생하지 않더라도 삽입 위치와 그 이후 요소들은 이동할 수 있으므로 해당 위치 이후를 가리키던 iterator/reference/pointer는 무효화될 수 있습니다.

따라서 단순히:

> `vector` 중간 insert는 변경 위치 이후만 무효화된다.

라고 외우면 부족합니다.

먼저 **재할당 발생 여부**를 확인해야 합니다.

### `vector` erase

erase된 위치와 그 이후의 원소가 앞으로 이동하므로 그 위치 이후를 가리키던 iterator/reference는 유효하다고 가정하면 안 됩니다.

erase 반환값을 이용해 순회를 계속하는 패턴이 흔합니다.

## `map`

일반적인 `map` 삽입은 기존 원소 자체를 이동시키지 않으므로 기존 iterator와 reference를 유지합니다.

```text
insert
→ 기존 iterator 대체로 유지
```

erase에서는 삭제된 원소를 가리키는 iterator/reference만 더 이상 유효하지 않습니다.

다른 원소를 가리키는 iterator는 계속 사용할 수 있습니다.

이 특성 때문에 요소 주소 안정성이 중요한 경우 `vector`와 다른 선택지가 될 수 있습니다.

## `deque`

`deque`는 연산별 iterator/reference 무효화 규칙이 `vector`나 `map`보다 복잡합니다.

앞·뒤 삽입과 중간 삽입, erase가 iterator와 reference에 미치는 영향이 같지 않습니다.

따라서 `deque`를 사용하면서 iterator를 장기간 보관한다면 해당 연산의 정확한 C++98 container 요구 사항을 확인해야 합니다.

단순히 "`deque`는 pointer가 안정적이다" 또는 "`vector`와 같다"고 가정하면 안 됩니다.

## iterator 무효화와 pointee 수명은 별도 문제입니다

다음을 생각합니다.

```cpp
std::map<std::string, Handler *> handlers;
```

map iterator가 유효하더라도:

```cpp
Handler *handler = it->second;
```

가 가리키는 `Handler` 객체가 별도로 `delete`되었다면 `handler`는 dangling pointer입니다.

즉 다음 두 수명을 구분해야 합니다.

```text
container element의 수명
pointee 객체의 수명
```

pointer container에서는 둘이 자동으로 같지 않습니다.

## STL container의 값 요구

C++98 표준 container는 원소를 값으로 저장하며 필요한 과정에서 복사를 수행할 수 있습니다.

예:

```cpp
std::vector<Record>
std::map<std::string, Record>
```

따라서 `Record`가 자원을 직접 소유한다면 복사가 의미상 올바르게 동작해야 합니다.

예:

```cpp
class Record {
private:
    char *data_;
};
```

compiler 기본 복사가 pointer 주소만 복사한다면 container 내부 복사 과정에서 double delete 같은 문제가 생길 수 있습니다.

C++98에서는 이런 타입에 Rule of Three를 검토해야 합니다.

## container가 정확히 몇 번 복사할지는 가정하지 않습니다

다음 코드:

```cpp
values.push_back(record);
```

를 보고:

> 정확히 한 번 복사된다.

라고 프로그램 의미를 설계하면 안 됩니다.

표준 container 구현 과정에서 허용되는 복사가 더 일어날 수 있습니다.

따라서 값 타입의 복사는 몇 번 호출되어도 논리적으로 올바른 독립 값을 만들어야 합니다.

복사 횟수가 매우 중요한 성능 문제라면 별도 측정과 container 설계를 검토해야 하지만 correctness를 복사 횟수에 의존시키면 안 됩니다.

## pointer를 container에 넣으면 pointer만 저장합니다

다음 container:

```cpp
std::map<std::string, Handler *>
```

가 저장하는 값은 `Handler` 객체가 아니라 `Handler *`라는 pointer 값입니다.

따라서 container가 소멸해도 자동으로 다음 작업을 하지 않습니다.

```cpp
delete handler;
```

예:

```cpp
std::map<std::string, Handler *> handlers;

handlers["PUT"] = new PutHandler;
```

container가 없어져도 pointee를 별도로 삭제하지 않으면 누수됩니다.

따라서 반드시 다음을 정해야 합니다.

```text
누가 Handler를 소유하는가?
언제 delete하는가?
container 복사 시 어떻게 하는가?
erase할 때 먼저 delete하는가?
```

## STL container와 소유권은 다른 문제입니다

다음을 구분합니다.

```cpp
std::vector<Value>
```

container가 `Value` 객체 자체의 수명을 관리합니다.

반면:

```cpp
std::vector<Value *>
```

container는 pointer 값의 수명만 관리합니다.

pointer가 가리키는 객체의 소유자는 별도로 정해야 합니다.

이 차이를 놓치면 raw pointer 기반 container에서 누수와 double delete가 발생하기 쉽습니다.

## algorithm

STL algorithm을 사용하면 container 종류와 연산 자체를 분리할 수 있습니다.

예:

```cpp
std::find(
    values.begin(),
    values.end(),
    target
);
```

```cpp
std::copy(
    source.begin(),
    source.end(),
    destination
);
```

```cpp
std::accumulate(
    values.begin(),
    values.end(),
    initial
);
```

C++98에서는 `<numeric>`의 `std::accumulate`, `<algorithm>`의 여러 algorithm을 사용할 수 있습니다.

기존 algorithm이 정확히 목적에 맞는다면 직접 loop를 다시 구현할 필요가 없습니다.

## algorithm을 무조건 사용하는 것도 목표는 아닙니다

다음처럼 여러 상태를 동시에 갱신하고 복수 조건으로 중간 종료하는 코드라면 명시적인 loop가 더 읽기 쉬울 수 있습니다.

```cpp
for (...) {
    if (...)
        break;

    if (...)
        updateA();

    updateB();
}
```

"STL algorithm을 사용했다"는 사실 자체가 코드 품질을 보장하지 않습니다.

무엇을 수행하는지 더 명확해지는지를 기준으로 선택합니다.

## 비교 함수와 strict weak ordering

정렬 algorithm에 전달하는 비교 함수는 단순히 `bool`을 반환하기만 하면 되는 것이 아닙니다.

예:

```cpp
struct RecordLess {
    bool operator()(
        const Record &left,
        const Record &right
    ) const {
        return left.value < right.value;
    }
};
```

비교 관계는 strict weak ordering 요구를 만족해야 합니다.

가장 쉽게 확인할 규칙 중 하나는:

```cpp
comp(x, x)
```

가 `false`여야 한다는 것입니다.

잘못된 예:

```cpp
return left.value <= right.value;
```

같은 값에서:

```cpp
comp(x, x) == true
```

가 되므로 정렬 algorithm이 요구하는 비교 관계를 만족하지 않습니다.

## 비교 관계가 순환하면 안 됩니다

다음 관계가 동시에 성립하는 식의 comparator는 문제가 됩니다.

```text
A < B
B < C
C < A
```

정렬 algorithm은 일관된 순서 관계를 전제로 합니다.

이 요구를 위반하면 algorithm이 정상적인 정렬 결과를 제공한다고 기대할 수 없습니다.

따라서 comparator는 단순한 구현 세부사항이 아니라 algorithm의 중요한 입력 조건입니다.

## `stable_sort`의 stable 의미

다음 두 record의 정렬 key가 같다고 가정합니다.

```text
A key=10
B key=10
```

입력 순서가:

```text
A, B
```

였다면 stable sort는 동등하게 비교되는 요소의 상대 순서를 유지합니다.

```text
A, B
```

일반 `sort`는 이 상대 순서 보존을 보장하지 않습니다.

따라서 동일 key 사이의 기존 순서가 의미가 있을 때 `stable_sort`를 선택할 수 있습니다.

## function object

C++98에는 lambda가 없으므로 호출 가능한 객체가 필요할 때 function object를 사용할 수 있습니다.

```cpp
struct RecordLess {
    bool operator()(
        const Record &left,
        const Record &right
    ) const {
        return left.value < right.value;
    }
};
```

사용:

```cpp
std::stable_sort(
    records.begin(),
    records.end(),
    RecordLess()
);
```

`RecordLess()`는 임시 function object이고 algorithm 내부에서는 함수처럼 호출할 수 있습니다.

```cpp
comparator(left, right);
```

실제로는:

```cpp
comparator.operator()(left, right);
```

가 호출됩니다.

## 상태를 가진 function object

비교 기준이 runtime 설정에 따라 달라질 수도 있습니다.

```cpp
class RecordLess {
public:
    explicit RecordLess(bool descending)
        : descending_(descending) {
    }

    bool operator()(
        const Record &left,
        const Record &right
    ) const {
        if (descending_)
            return right.value < left.value;

        return left.value < right.value;
    }

private:
    bool descending_;
};
```

사용:

```cpp
std::stable_sort(
    records.begin(),
    records.end(),
    RecordLess(true)
);
```

lambda capture가 없는 C++98에서는 이런 방식으로 필요한 상태를 객체에 저장할 수 있습니다.

## function pointer와 function object를 구분합니다

상태가 필요 없는 단순 함수라면 일반 함수 pointer로도 충분할 수 있습니다.

```cpp
bool recordLess(
    const Record &left,
    const Record &right
) {
    return left.value < right.value;
}
```

사용:

```cpp
std::stable_sort(
    records.begin(),
    records.end(),
    recordLess
);
```

즉 C++98에서 lambda가 없다고 해서 항상 클래스를 하나 만들어야 하는 것은 아닙니다.

* 상태 없음 → 일반 함수도 가능
* 상태 필요 → function object가 유용
* 여러 연산을 한 타입으로 표현 → function object가 유용

## specialization

일반 template과 다른 동작이 특정 타입에만 필요하면 specialization을 사용할 수 있습니다.

예:

```cpp
template <class T>
struct Printer {
    static void print(const T &value) {
        std::cout << value;
    }
};
```

특정 타입의 전체 specialization을 둘 수 있습니다.

```cpp
template <>
struct Printer<bool> {
    static void print(bool value) {
        std::cout
            << (value ? "true" : "false");
    }
};
```

## specialization이 항상 최선은 아닙니다

단순히 함수 호출 하나의 동작만 타입에 따라 달라진다면 overload가 더 직접적일 수 있습니다.

```cpp
void print(bool value);

template <class T>
void print(const T &value);
```

specialization은 template 동작 자체의 타입별 변형이 필요할 때 사용합니다.

불필요하게 사용하면 어떤 버전이 선택되는지 추적하기 어려워질 수 있습니다.

## partial specialization

class template은 partial specialization을 지원합니다.

개념적으로:

```cpp
template <class T>
class Traits;
```

의 일부 형태를 별도로 처리할 수 있습니다.

하지만 function template에는 class template과 같은 형태의 partial specialization을 직접 사용할 수 없습니다.

함수에서 일부 타입 패턴만 다르게 처리하고 싶다면 overload가 일반적인 대안입니다.

## template과 overload 선택은 별도 규칙입니다

다음이 함께 있을 수 있습니다.

```cpp
template <class T>
void print(const T &value);

void print(int value);
```

`int`를 전달하면 overload resolution 규칙에 따라 일반 함수가 선택될 수 있습니다.

즉:

```text
template specialization
```

과

```text
함수 overload
```

는 같은 기능이 아닙니다.

어떤 문제를 해결하려는지 구분해야 합니다.

## 자주 놓치는 문제

* template 선언만 header에 두고 정의를 `.cpp`에 숨긴 뒤 다른 번역 단위에서 사용합니다.
* template이 실제 타입으로 instantiation되는 시점을 이해하지 못합니다.
* 모든 template parameter가 어떤 타입이든 동작한다고 생각합니다.
* dependent type 앞의 `typename`을 빠뜨립니다.
* `typename`을 모든 `::` 타입 앞에 기계적으로 붙입니다.
* `end()`를 마지막 원소라고 생각하고 역참조합니다.
* 빈 container의 `begin() == end()`를 고려하지 않습니다.
* 직접 구현한 null pointer 기반 iterator에서 pointer arithmetic을 가볍게 사용합니다.
* const container에서 mutable iterator를 반환합니다.
* `const iterator`와 `const_iterator`를 같은 개념으로 생각합니다.
* 모든 iterator가 `it + n`을 지원한다고 생각합니다.
* `std::stable_sort`를 `list` 같은 non-random-access iterator에 사용하려 합니다.
* vector 재할당 뒤 이전 pointer, reference, iterator를 사용합니다.
* vector insert에서 재할당 여부를 무시합니다.
* map iterator가 살아 있다는 이유로 pointee 객체도 살아 있다고 생각합니다.
* pointer container가 pointee 객체까지 자동으로 삭제한다고 생각합니다.
* container가 내부적으로 정확히 몇 번 복사하는지에 프로그램 correctness를 의존시킵니다.
* comparator가 같은 값에 대해 `true`를 반환합니다.
* comparator가 일관된 순서 관계를 제공하지 않습니다.
* lambda가 없다는 이유로 모든 callback을 class로 만듭니다.
* 함수 overload로 충분한 문제에 복잡한 specialization을 사용합니다.

## 완료 기준

* 함수 template과 클래스 template을 C++98 문법으로 작성할 수 있습니다.
* template declaration과 instantiation의 차이를 설명할 수 있습니다.
* template 정의를 일반적으로 header에 두는 이유를 번역 단위와 연결해 설명할 수 있습니다.
* 명시적 instantiation을 사용하면 지원 타입이 제한되는 이유를 설명할 수 있습니다.
* dependent name과 `typename`의 관계를 설명할 수 있습니다.
* `[begin, end)` 범위에서 `end()`를 역참조하면 안 되는 이유를 설명할 수 있습니다.
* 빈 범위를 별도 특수 처리 없이 순회할 수 있는 이유를 이해합니다.
* 직접 만든 iterator에서 null pointer와 빈 범위를 안전하게 표현합니다.
* mutable iterator와 `const_iterator`를 구분합니다.
* iterator 객체의 const와 원소 const 접근을 구분합니다.
* iterator category에 따라 사용할 수 있는 연산이 달라짐을 설명할 수 있습니다.
* `stable_sort`가 random access iterator를 요구한다는 것을 확인할 수 있습니다.
* `vector`, `map`, `deque`의 iterator 무효화 규칙이 서로 다름을 이해합니다.
* container iterator 수명과 pointer pointee 수명을 별도로 판단할 수 있습니다.
* 값 container와 pointer container의 소유권 차이를 설명할 수 있습니다.
* 자원 소유 타입을 container에 저장할 때 Rule of Three가 필요한지 판단할 수 있습니다.
* strict weak ordering을 만족하지 않는 comparator의 문제를 설명할 수 있습니다.
* 일반 함수, function object, template specialization, overload를 상황에 맞게 구분할 수 있습니다.
