# algorithm·range·template·concept

## 목표

컨테이너를 순회할 때 항상 직접 `for`문부터 작성하지 않습니다.

검색, 선택, 변환, 정렬처럼 이미 이름이 있는 작업이라면 표준 algorithm과 range를 사용해 코드의 의도를 직접 표현할 수 있습니다.

동시에 다음을 판단해야 합니다.

* 어떤 container가 필요한 연산에 적합합니까?
* iterator, pointer, reference는 언제 무효화됩니까?
* range view는 원본을 소유합니까?
* template은 실제로 어떤 연산을 요구합니까?
* concept은 어느 정도까지 제약해야 합니까?
* 알고리즘의 이론상 복잡도와 실제 비용은 어떻게 다를 수 있습니까?

## container부터 고르지 말고 필요한 연산부터 적습니다

먼저 프로그램이 자주 수행하는 연산을 확인합니다.

예:

```text
- 뒤에 항목 추가
- 전체 순차 순회
- index로 접근
- 가끔 정렬
```

이 경우 `std::vector`가 자연스러운 선택일 가능성이 높습니다.

대표적인 성격은 다음과 같습니다.

* 순차 저장과 index 접근: `std::vector`
* key 기준 정렬 순회와 로그 시간 검색: `std::map`
* 순서가 필요 없고 평균 상수 시간 key 검색: `std::unordered_map`
* 양끝 삽입·삭제: `std::deque`
* LIFO 인터페이스만 필요: `std::stack`

다만 `std::stack`은 독립적인 저장 구조라기보다 다른 container 위에 제한된 LIFO 인터페이스를 제공하는 **container adapter**입니다.

기본적으로 `std::deque`를 내부 container로 사용하지만 다른 적절한 container를 지정할 수도 있습니다.

### Big-O만으로 container를 선택하지 않습니다

예를 들어 `std::map::find()`는 `O(log n)`이고 `std::vector`의 선형 검색은 `O(n)`입니다.

그렇다고 작은 데이터에서도 항상 `map`이 빠른 것은 아닙니다.

`vector`는 원소가 연속 메모리에 있어 cache locality가 좋고 추가적인 node allocation과 pointer 추적이 없습니다.

따라서 다음을 함께 봅니다.

* 실제 원소 수
* 검색 횟수
* 삽입/삭제 빈도
* 순서 보장 필요 여부
* 메모리 사용
* cache locality
* 구현 단순성

이론상 복잡도는 중요한 출발점이지만 실제 workload도 고려합니다.

## algorithm으로 의도를 표현합니다

직접 반복문:

```cpp
auto found = tasks.end();

for (auto it = tasks.begin(); it != tasks.end(); ++it) {
    if (it->ready()) {
        found = it;
        break;
    }
}
```

표준 algorithm:

```cpp
const auto found =
    std::ranges::find_if(
        tasks,
        [](const Task& task) {
            return task.ready();
        }
    );
```

두 코드는 모두 가능합니다.

하지만 후자는 읽는 즉시 "조건에 맞는 첫 원소를 찾는다"는 목적을 알 수 있습니다.

다음과 같은 작업이라면 표준 algorithm을 먼저 확인합니다.

* 찾기: `find`, `find_if`
* 조건 검사: `all_of`, `any_of`, `none_of`
* 정렬: `sort`
* 개수 세기: `count`, `count_if`
* 변환: `transform`
* 제거 대상 이동: `remove`, `remove_if`
* 합계와 누적: `accumulate`, `fold_left` 등

직접 loop가 잘못된 것은 아닙니다.

다음처럼 한 번의 순회에서 여러 상태를 변경하거나 복잡한 early exit가 필요하면 loop가 오히려 명확할 수 있습니다.

```cpp
for (const Task& task : tasks) {
    if (task.cancelled()) {
        cancelled += 1;
        continue;
    }

    if (task.failed()) {
        failed += 1;

        if (failed > limit) {
            break;
        }
    }
}
```

목표는 "algorithm을 무조건 사용"하는 것이 아니라 코드가 하는 일을 가장 구체적으로 드러내는 표현을 선택하는 것입니다.

## iterator와 반열린 범위

전통적인 표준 algorithm은 보통 다음 범위를 받습니다.

```text
[first, last)
```

의미:

* `first`가 가리키는 원소는 포함
* `last`가 가리키는 위치는 포함하지 않음

예:

```cpp
std::sort(values.begin(), values.end());
```

`end()`는 마지막 원소가 아닙니다.

```text
values:

[10][20][30]
 ↑           ↑
begin()      end()
```

마지막 원소를 가리키는 iterator는 `end() - 1`일 수 있지만, 이는 random-access iterator일 때만 가능한 표현입니다.

빈 range에서는:

```cpp
values.begin() == values.end()
```

이므로 반열린 범위 표현은 빈 범위를 별도 예외 처리하지 않아도 자연스럽게 표현합니다.

## `end()` iterator를 역참조하지 않습니다

검색 결과를 확인할 때 다음 순서가 필요합니다.

```cpp
auto it = std::ranges::find(values, target);

if (it != values.end()) {
    use(*it);
}
```

다음은 잘못되었습니다.

```cpp
use(*it);

if (it != values.end()) {
    // ...
}
```

검색에 실패했다면 `it == end()`이고 `end()`는 원소를 가리키지 않으므로 역참조할 수 없습니다.

## iterator invalidation

iterator가 한 번 만들어졌다고 계속 유효한 것은 아닙니다.

예:

```cpp
std::vector<int> values{1, 2, 3};

auto it = values.begin();

values.push_back(4);
```

`push_back()` 과정에서 capacity가 부족했다면 vector가 새 메모리를 할당하고 기존 원소를 옮깁니다.

```text
before

old memory
[1][2][3]
 ↑
 it

reallocation

new memory
[1][2][3][4]

old memory는 더 이상 원소 저장 공간이 아님
```

이 경우 기존:

* iterator
* pointer
* reference

가 모두 무효화됩니다.

하지만 중요한 세부사항이 있습니다.

**`vector::push_back()`이 항상 모든 iterator를 무효화하는 것은 아닙니다.**

재할당이 발생하지 않았다면 기존 원소를 가리키는 iterator와 reference는 일반적으로 유지되고 새로운 `end()` 위치만 달라집니다.

따라서 정확한 판단은 해당 container와 연산의 invalidation 규칙을 확인해야 합니다.

## container마다 invalidation 규칙이 다릅니다

예를 들어 일반적으로 다음과 같은 차이가 있습니다.

* `vector`: 재할당 시 기존 원소의 iterator/reference/pointer가 무효화됨
* `list`: 삽입 자체는 기존 원소 iterator를 보통 무효화하지 않음
* `map`: 삽입이 기존 원소 iterator/reference를 보통 무효화하지 않음
* `unordered_map`: rehash가 발생하면 iterator가 무효화될 수 있음

따라서 "container 변경 후 iterator는 모두 위험하다"처럼 하나의 규칙으로 외우기보다 **container별 보장**을 확인합니다.

## range

C++20 range algorithm은 iterator pair 대신 range 자체를 전달할 수 있습니다.

기존 방식:

```cpp
std::sort(values.begin(), values.end());
```

range 방식:

```cpp
std::ranges::sort(values);
```

range는 단순히 문법을 줄이는 기능만은 아닙니다.

iterator와 sentinel 개념, view 조합, projection 등 더 일반적인 범위 처리를 지원합니다.

## view는 값을 저장한 새 container가 아닙니다

다음 코드를 봅니다.

```cpp
auto ready =
    tasks
    | std::views::filter(
        [](const Task& task) {
            return task.ready();
        }
    );
```

`ready`에 준비된 `Task`들이 복사되어 저장되는 것이 아닙니다.

대부분의 view는 원본 range를 참조하면서 반복할 때 조건을 계산합니다.

```text
tasks
[Task][Task][Task][Task]
   ↑
   │
filter_view
   │
   └─ iteration 시 ready() 검사
```

따라서 다음 장점이 있습니다.

* 중간 container allocation을 피할 수 있음
* 여러 view를 조합할 수 있음
* 필요한 시점까지 계산을 미룰 수 있음

하지만 원본과 수명 관계를 반드시 확인해야 합니다.

## view의 lazy evaluation

다음 코드에서:

```cpp
auto ready =
    tasks
    | std::views::filter(
        [](const Task& task) {
            return task.ready();
        }
    );
```

`ready`를 만드는 순간 모든 `task.ready()`가 실행된다고 가정하면 안 됩니다.

실제 조건 검사는 보통 view를 순회할 때 일어납니다.

```cpp
for (const Task& task : ready) {
    use(task);
}
```

따라서 같은 view를 여러 번 순회하면 조건 계산도 다시 이루어질 수 있습니다.

```cpp
for (const Task& task : ready) {
    // filter 계산
}

for (const Task& task : ready) {
    // 다시 계산될 수 있음
}
```

계산 비용이 크거나 결과 snapshot이 필요하다면 실제 container로 materialize하는 편이 적합할 수 있습니다.

## view와 원본 수명

다음처럼 지역 container를 가리키는 view를 바깥에 반환하면 위험할 수 있습니다.

```cpp
auto make_ready_view() {
    std::vector<Task> tasks = load_tasks();

    return tasks | std::views::filter(
        [](const Task& task) {
            return task.ready();
        }
    );
}
```

함수 종료 시 `tasks`가 파괴된다면 반환된 view가 그 원본을 참조할 수 없습니다.

비소유 결과에서는 항상 다음 질문을 해야 합니다.

```text
view가 참조하는 원본은 누구인가?
원본은 view보다 오래 살아 있는가?
```

단, 모든 range/view가 반드시 동일하게 non-owning인 것은 아닙니다. C++ ranges에는 전달된 rvalue range를 내부에 소유할 수 있도록 하는 `owning_view` 같은 형태도 있습니다.

따라서 "모든 view는 무조건 비소유"라고 외우기보다는 사용하는 view가 무엇을 보관하는지 확인해야 합니다.

실무에서는 일반적인 `filter_view`, `transform_view` 등을 사용할 때 원본 수명을 특히 주의합니다.

## 원본 변경과 view

원본이 살아 있다고 view가 반드시 안전한 것은 아닙니다.

```cpp
auto ready =
    tasks
    | std::views::filter(pred);

auto it = ready.begin();

tasks.push_back(new_task);
```

`tasks`가 `std::vector`이고 재할당되었다면 view 자체가 보관하는 원본 객체는 여전히 존재하지만 이전 iterator는 무효화될 수 있습니다.

따라서 다음 두 질문을 구분합니다.

```text
원본 객체가 살아 있는가?
+
현재 iterator/reference가 아직 유효한가?
```

## 결과를 오래 보관해야 하는 경우

결과가 원본과 독립적으로 살아야 한다면 값을 복사하거나 이동하여 새 container를 만듭니다.

예:

```cpp
std::vector<Task> result;

for (const Task& task : tasks) {
    if (task.ready()) {
        result.push_back(task);
    }
}
```

이 결과는 원본 `tasks`가 사라져도 유지됩니다.

반면 복사 비용을 피하고 원본 객체를 가리키는 목록만 만들 수도 있습니다.

```cpp
std::vector<std::reference_wrapper<const Task>> result;

for (const Task& task : tasks) {
    if (matches(task, query)) {
        result.emplace_back(std::cref(task));
    }
}
```

이 결과는 `Task` 객체를 복사하지 않습니다.

하지만 lifetime은 원본에 의존합니다.

```text
tasks 안의 Task
      ↑
      │
reference_wrapper
```

따라서 `result`를 `tasks`보다 오래 사용하면 안 됩니다.

## `reference_wrapper`는 reference의 수명을 늘리지 않습니다

```cpp
std::reference_wrapper<const Task>
```

를 container에 넣었다고 해서 `Task`의 수명이 연장되는 것은 아닙니다.

이는 복사 가능한 wrapper 안에 참조 관계를 표현할 뿐입니다.

따라서 다음은 여전히 잘못될 수 있습니다.

```cpp
auto make_result() {
    std::vector<Task> tasks = load();

    std::vector<
        std::reference_wrapper<const Task>
    > result;

    result.emplace_back(tasks.front());

    return result;
}
```

함수 종료 후 `tasks`가 파괴되므로 반환된 reference는 dangling입니다.

## 원본을 바꾸지 않는 정렬된 조회

검색 결과를 특정 순서로 보여주고 싶다고 합시다.

원본 container의 순서가 다른 코드에서도 의미가 있다면 조회 때문에 원본 자체를 정렬하면 안 될 수 있습니다.

```cpp
std::vector<std::reference_wrapper<const Task>> result;

for (const Task& task : tasks) {
    if (matches(task, query)) {
        result.emplace_back(std::cref(task));
    }
}
```

그 다음 reference 목록만 정렬합니다.

`std::reference_wrapper` 자체를 비교하는 것이 아니라 가리키는 `Task`를 기준으로 projection이나 comparator를 명확히 작성하는 편이 좋습니다.

예:

```cpp
std::ranges::sort(
    result,
    {},
    [](const auto& ref) {
        return ref.get().id();
    }
);
```

이렇게 하면 원본 `tasks`의 순서는 유지됩니다.

## 정렬 comparator의 조건

정렬 comparator는 단순히 `bool`을 반환하기만 하면 되는 것이 아닙니다.

`std::sort`와 `std::ranges::sort`가 요구하는 비교 관계는 **strict weak ordering**을 만족해야 합니다.

예를 들어 다음 비교는 잘못될 수 있습니다.

```cpp
return lhs.score <= rhs.score;
```

같은 객체끼리 비교하면:

```cpp
compare(x, x) == true
```

가 될 수 있기 때문입니다.

정상적인 less-than 비교는 자기 자신보다 작다고 판단해서는 안 됩니다.

```cpp
return lhs.score < rhs.score;
```

일반적인 직관은 다음과 같습니다.

* `x < x`는 false
* `x < y`가 true이면 `y < x`는 false
* 비교 관계가 일관된 순서를 형성해야 함

조건을 깨뜨린 comparator를 정렬 algorithm에 전달하면 결과를 신뢰할 수 없습니다.

## tie-breaker와 결정적인 순서

다음처럼 duration으로 정렬한다고 합시다.

```cpp
return lhs.duration < rhs.duration;
```

두 Task의 duration이 같다면 comparator 입장에서는 둘이 동등한 순위입니다.

```text
A.duration = 10
B.duration = 10
```

이때 일반적인 `std::sort`는 두 원소의 기존 상대 순서를 유지한다고 보장하지 않습니다.

실행할 때마다 반드시 순서가 바뀐다는 뜻은 아닙니다. 단지 `duration`만으로는 어느 쪽이 먼저여야 하는지를 정의하지 않은 것입니다.

출력이나 test에 완전히 결정적인 순서가 필요하다면 tie-breaker를 추가합니다.

```cpp
return std::tie(lhs.duration, lhs.id)
     < std::tie(rhs.duration, rhs.id);
```

그러면:

```text
1차: duration
2차: id
```

순서로 비교합니다.

C++20에서는 `<=>` 또는 tuple 기반 비교도 활용할 수 있습니다.

## 안정 정렬과 tie-breaker는 다른 개념입니다

`std::stable_sort`는 비교 결과가 동등한 원소들의 **기존 상대 순서**를 유지합니다.

예:

```text
원본:
A(duration=10)
B(duration=10)

stable_sort 후:
A
B
```

하지만 원본 입력 자체의 순서가 비결정적이라면 stable sort만으로 최종 결과가 결정적이라고 할 수 없습니다.

예를 들어 `unordered_map` 순회 결과를 가져와 stable sort했는데 정렬 key가 동일하다면 최초 입력 순서가 안정적으로 정의되지 않을 수 있습니다.

따라서 원하는 조건이:

```text
항상 duration이 같으면 id가 작은 순서
```

라면 comparator에 `id`를 tie-breaker로 직접 넣는 것이 정확합니다.

## function template

template은 구체적인 타입을 미리 정하지 않고 타입에 필요한 연산을 기준으로 코드를 작성할 수 있게 합니다.

```cpp
template <typename Iterator, typename Predicate>
Iterator find_match(
    Iterator first,
    Iterator last,
    Predicate predicate
) {
    for (; first != last; ++first) {
        if (predicate(*first)) {
            return first;
        }
    }

    return last;
}
```

이 함수는 임의의 타입을 실제로 아무 조건 없이 허용하는 것이 아닙니다.

구현이 다음 연산을 요구합니다.

```text
Iterator:
- first != last
- ++first
- *first

Predicate:
- predicate(*first)
- 그 결과를 조건으로 사용 가능
```

즉 template 구현에서 사용하는 표현식이 사실상 타입의 요구사항을 만듭니다.

## unconstrained template의 오류

잘못된 타입을 전달하면 compiler가 함수 내부 깊은 곳에서 오류를 낼 수 있습니다.

```cpp
find_match(
    42,
    100,
    some_predicate
);
```

`int`에는 `*first` 같은 iterator 연산이 맞지 않으므로 오류가 발생합니다.

template 코드가 복잡하면 오류 메시지도 매우 길어질 수 있습니다.

이럴 때 concept으로 호출 가능한 타입을 제한하고 요구사항을 명시할 수 있습니다.

## concept

예를 들어 특정 range에 대해 작업하는 함수를 작성할 수 있습니다.

단순히 `Job`을 순회하며 읽기만 필요하다면 실제 요구사항보다 강한 조건을 넣지 않는 것이 중요합니다.

예를 들어 다음 concept:

```cpp
template <typename Range>
concept JobRange =
    std::ranges::input_range<Range> &&
    std::same_as<
        std::remove_cvref_t<
            std::ranges::range_reference_t<Range>
        >,
        Job
    >;
```

는 reference 타입을 정규화한 결과가 정확히 `Job`이어야 한다는 조건입니다.

하지만 이 조건은 생각보다 강할 수 있습니다.

예를 들어 proxy reference를 사용하는 range나 `Job`으로 변환 가능한 다른 reference 형태를 배제할 수 있습니다.

함수가 실제로 필요한 것이 단순히 각 원소에서 다음을 호출하는 것뿐이라면:

```cpp
job.failed()
```

그 연산 자체를 기준으로 제약하는 편이 더 정확할 수 있습니다.

예:

```cpp
template <typename T>
concept FailedReadable =
    requires(const T& value) {
        { value.failed() } -> std::convertible_to<bool>;
    };
```

range까지 결합하면:

```cpp
template <typename Range>
concept FailedJobRange =
    std::ranges::input_range<Range> &&
    FailedReadable<
        std::remove_cvref_t<
            std::ranges::range_reference_t<Range>
        >
    >;
```

핵심은 concept을 "특정 concrete type만 받는 장치"로 생각하지 않고 **함수가 실제로 필요로 하는 기능을 표현하는 장치**로 보는 것입니다.

## concept은 구현 오류를 없애지 않습니다

concept이 있다고 함수 구현이 자동으로 올바르게 되는 것은 아닙니다.

```cpp
template <FailedJobRange Range>
std::size_t count_failed(const Range& jobs) {
    // 여전히 내부 구현 bug는 가능
}
```

concept이 하는 일은 주로 다음과 같습니다.

* 어떤 호출이 overload 후보가 될 수 있는지 제한
* template의 요구사항을 명시
* 잘못된 호출에서 더 가까운 위치에 오류를 제공
* overload 선택에 사용

logic bug, lifetime bug, out-of-bounds 같은 일반적인 구현 오류를 방지하는 기능은 아닙니다.

## concept을 지나치게 강하게 만들지 않습니다

함수가 한 번 순회만 한다면:

```cpp
std::ranges::input_range
```

로 충분할 수 있습니다.

그런데 이유 없이:

```cpp
std::ranges::random_access_range
```

를 요구하면 `std::list`, stream 기반 range 등 불필요하게 많은 타입이 제외됩니다.

마찬가지로 함수가 값을 읽기만 하는데 mutable reference를 요구할 필요도 없습니다.

원칙:

```text
함수 구현에 실제로 필요한 최소 요구사항만 표현
```

이렇게 해야 template을 재사용할 수 있는 범위가 넓어집니다.

## algorithm과 projection

ranges algorithm은 projection을 지원합니다.

예를 들어 `Task`를 `id` 기준으로 정렬하고 싶다면 comparator에 전체 접근 코드를 반복하지 않아도 됩니다.

```cpp
std::ranges::sort(
    tasks,
    std::less{},
    &Task::id
);
```

개념적으로 각 `Task`에 `Task::id`를 적용한 결과를 비교합니다.

이는 다음처럼 comparator를 직접 작성하는 것과 비슷한 의도를 가집니다.

```cpp
std::ranges::sort(
    tasks,
    [](const Task& lhs, const Task& rhs) {
        return lhs.id() < rhs.id();
    }
);
```

projection을 사용하면 "원소를 어떤 값으로 비교하는가"를 분리해서 표현할 수 있습니다.

## algorithm의 반환값도 확인합니다

ranges algorithm은 전통 algorithm과 반환 타입이 다른 경우가 있습니다.

예:

```cpp
auto it = std::ranges::find(tasks, id, &Task::id);
```

결과는 iterator일 수 있지만 전달하는 range의 lifetime에 따라 dangling 방지를 위해 `std::ranges::dangling`이 반환될 수도 있습니다.

예를 들어 임시 container에 대한 iterator를 외부로 반환하면 원본이 즉시 파괴되어 위험합니다.

ranges library는 일부 경우 이를 타입 수준에서 막기 위해 **borrowed range** 개념을 사용합니다.

따라서 range algorithm의 반환 iterator를 오래 저장한다면 원본 lifetime까지 확인해야 합니다.

## 복잡도와 실제 비용

algorithm 이름만 보고 전체 성능을 판단하지 않습니다.

### `vector::push_back`

일반적으로 amortized `O(1)`입니다.

대부분의 삽입은 뒤에 바로 추가할 수 있지만 capacity가 부족하면:

```text
새 메모리 할당
↓
기존 원소 이동/복사
↓
기존 메모리 해제
↓
새 원소 추가
```

가 필요합니다.

따라서 개별 한 번의 `push_back()`은 `O(n)`일 수 있지만 여러 번 수행했을 때 평균적으로 상수 시간에 가까운 비용을 갖습니다.

### `reserve()`

예상 원소 수를 알고 있다면:

```cpp
std::vector<Task> tasks;
tasks.reserve(expected_count);
```

로 미리 capacity를 확보하여 반복적인 재할당을 줄일 수 있습니다.

하지만 필요 이상으로 매우 큰 크기를 reserve하면 메모리를 불필요하게 확보할 수 있으므로 무조건 호출하는 최적화는 아닙니다.

### `map`

`map::find()`는 보통 `O(log n)`입니다.

하지만 node 기반 자료구조이므로 다음 비용도 존재합니다.

* 개별 node allocation
* pointer chasing
* 낮은 cache locality

### view

view는 중간 container를 만들지 않아 allocation과 복사를 줄일 수 있습니다.

반면 lazy evaluation 때문에 동일한 계산을 여러 차례 수행할 수도 있습니다.

예:

```cpp
auto expensive =
    values
    | std::views::filter(expensive_predicate);
```

이 view를 세 번 순회한다면 predicate도 여러 번 평가될 수 있습니다.

결과가 여러 번 필요하다면 한 번 계산해서 container에 저장하는 편이 더 효율적일 수 있습니다.

## 성능 측정

정렬 속도를 측정하고 싶다면 다음 전체 시간을 재면 결과를 해석하기 어렵습니다.

```text
파일 읽기
+
문자열 parsing
+
정렬
+
출력
```

정렬 자체를 비교하려면 측정 범위를 분리합니다.

또한 최소한 다음 정보를 함께 기록합니다.

* 입력 크기
* 데이터 분포
* compiler
* compiler option
* Debug / Release
* 측정 반복 횟수

특히 Debug build의 결과를 실제 최적화 성능으로 해석하지 않습니다.

## 자주 놓치는 문제

* container를 익숙하다는 이유만으로 선택하고 실제 주요 연산을 확인하지 않습니다.
* 이론상 Big-O만 보고 작은 입력에서의 allocation과 cache 비용을 무시합니다.
* 이미 이름이 있는 단순 작업을 복잡한 loop로 다시 작성합니다.
* 반대로 여러 상태를 갱신하는 복잡한 loop를 억지로 여러 algorithm으로 쪼갭니다.
* `end()` iterator를 역참조합니다.
* `vector`가 변경된 뒤 기존 iterator가 항상 유효하다고 생각합니다.
* 반대로 모든 `push_back()`이 무조건 iterator 전체를 무효화한다고 생각합니다.
* view를 새 container라고 생각합니다.
* lazy view가 생성 시 모든 결과를 계산한다고 생각합니다.
* view 또는 `reference_wrapper`를 원본보다 오래 저장합니다.
* stable sort와 완전한 tie-breaker를 같은 개념으로 생각합니다.
* comparator에서 `<=`를 사용해 strict weak ordering을 깨뜨립니다.
* template 오류를 줄이려다 실제로 필요하지 않은 강한 concept을 추가합니다.
* range algorithm이 반환한 iterator의 원본 lifetime을 확인하지 않습니다.
* 성능 측정에서 parsing과 출력 비용까지 한꺼번에 포함하고 algorithm 성능이라고 판단합니다.

## 코드를 읽을 때 확인할 질문

1. 가장 자주 수행하는 container 연산은 무엇입니까?
2. 데이터 크기와 순서 보장 요구사항은 무엇입니까?
3. 이 loop는 이름이 있는 표준 algorithm으로 더 직접적으로 표현할 수 있습니까?
4. 현재 iterator/reference/pointer는 어떤 연산에서 무효화됩니까?
5. view는 무엇을 참조하거나 소유합니까?
6. view를 사용하는 시점에도 원본이 살아 있습니까?
7. 같은 lazy 계산을 반복해서 수행하고 있지는 않습니까?
8. 검색 결과를 원본과 독립적으로 보관해야 합니까?
9. 정렬 결과가 완전히 결정적이어야 합니까?
10. comparator가 strict weak ordering을 만족합니까?
11. template 구현이 실제로 사용하는 연산은 무엇입니까?
12. concept이 그 요구사항보다 지나치게 강하지 않습니까?
13. range algorithm의 반환 iterator가 원본보다 오래 살아남지 않습니까?
14. 실제 성능 문제를 Release build와 현실적인 입력으로 측정했습니까?

## 완료 기준

* 주요 연산과 실제 입력 특성을 기준으로 container를 선택합니다.
* 직접 loop와 표준 algorithm 중 의도가 더 잘 드러나는 표현을 선택합니다.
* 반열린 범위 `[first, last)`의 의미를 설명합니다.
* container별 iterator invalidation 규칙이 다르다는 것을 설명합니다.
* range view가 lazy하게 계산될 수 있음을 설명합니다.
* view와 비소유 결과의 원본 lifetime을 추적합니다.
* stable sort와 tie-breaker의 차이를 설명합니다.
* comparator가 strict weak ordering을 만족하도록 작성합니다.
* template이 실제로 요구하는 연산을 식별합니다.
* concept을 실제 필요한 최소 요구사항으로 제한합니다.
* range algorithm 반환값의 lifetime 조건을 확인합니다.
* 이론상 복잡도뿐 아니라 allocation, locality, 반복 계산 비용을 함께 고려합니다.
