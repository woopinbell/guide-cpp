# algorithm·range·template·concept

## 목표

컨테이너를 직접 반복하는 코드만 작성하지 않고, 표준 algorithm과 range를 사용해 선택·변환·정렬 의도를 드러냅니다. template이 받아들일 타입과 반환 결과의 수명도 명시합니다.

## 컨테이너부터 고릅니다

자료구조 이름보다 필요한 연산을 먼저 적습니다.

- 순서대로 저장하고 index 접근: `std::vector`
- key로 찾고 정렬된 순회: `std::map`
- 평균 상수 시간 key 조회, 순서 불필요: `std::unordered_map`
- 뒤에서 넣고 빼기: `std::vector` 또는 `std::stack`
- 양끝 삽입·삭제: `std::deque`

원소 수가 작다면 이론상 복잡도보다 연속 메모리와 단순한 코드가 더 나을 수 있습니다. 실제 입력과 주요 연산을 기준으로 선택합니다.

## algorithm으로 의도를 적습니다

```cpp
const auto found = std::ranges::find_if(tasks, [](const Task& task) {
    return task.ready();
});
```

직접 loop가 잘못된 것은 아닙니다. 여러 조건에서 일찍 빠져나오거나 상태를 함께 갱신한다면 loop가 더 명확할 수 있습니다. 다만 검색, 정렬, 누적처럼 이미 이름이 있는 작업은 표준 algorithm을 먼저 확인합니다.

## iterator와 반열린 범위

표준 algorithm은 보통 `[first, last)` 범위를 사용합니다. `last`는 마지막 원소가 아니라 마지막 다음 위치입니다.

```cpp
std::sort(values.begin(), values.end());
```

빈 범위에서는 `begin() == end()`입니다. iterator가 가리키는 원소를 읽기 전에 `it != end()`를 확인합니다.

컨테이너 변경 뒤 iterator 유효 여부를 확인해야 합니다. `vector` 재할당은 기존 pointer, reference, iterator를 모두 무효화합니다.

## range와 view

view는 보통 원본을 소유하지 않고 계산을 지연합니다.

```cpp
auto ready = tasks | std::views::filter([](const Task& task) {
    return task.ready();
});
```

`ready`를 사용하는 동안 `tasks`가 살아 있어야 합니다. 원본을 파괴하거나 재할당하면 view와 그 iterator가 더 이상 유효하지 않을 수 있습니다.

결과를 오래 저장해야 한다면 값을 새 container에 복사하거나, 원본 수명을 함께 관리하는 비소유 참조 목록을 만듭니다.

## 원본을 바꾸지 않는 조회

조회 결과만 정렬해야 한다면 원본 container를 직접 정렬하지 않습니다.

```cpp
std::vector<std::reference_wrapper<const Task>> result;
for (const Task& task : tasks) {
    if (matches(task, query))
        result.emplace_back(std::cref(task));
}
std::ranges::sort(result, compare_by_id);
```

이 결과는 `Task`를 복사하지 않지만 원본보다 오래 사용할 수 없습니다. 반환 타입과 문서에 수명 조건을 적습니다.

## 결정적인 순서

주 정렬 값이 같을 때 tie-breaker를 정합니다.

```cpp
return std::tie(lhs.duration, lhs.id)
     < std::tie(rhs.duration, rhs.id);
```

tie-breaker가 없으면 실행마다 결과가 달라진다고 단정할 수는 없지만, 어떤 순서도 보장하지 못합니다. 출력, test, cache key에 순서가 중요하다면 완전한 비교 기준을 둡니다.

## function template과 class template

```cpp
template <typename Iterator, typename Predicate>
Iterator find_match(Iterator first, Iterator last, Predicate predicate) {
    for (; first != last; ++first) {
        if (predicate(*first))
            return first;
    }
    return last;
}
```

구현에서 실제로 사용하는 연산이 template의 요구사항입니다. 문서나 concept 없이 임의의 타입을 받으면 긴 compiler 오류가 발생할 수 있습니다.

## concept으로 요구 연산을 제한합니다

```cpp
template <typename Range>
concept JobRange = std::ranges::input_range<Range> &&
    std::same_as<std::remove_cvref_t<
        std::ranges::range_reference_t<Range>>, Job>;

template <JobRange Range>
std::size_t count_failed(const Range& jobs);
```

concept은 함수 내부 오류를 없애는 기능이 아니라, 어떤 타입이 호출 후보가 되는지 제한합니다. 필요 이상으로 강한 조건을 요구하면 사용할 수 있는 타입이 줄어듭니다.

## 복잡도와 할당

algorithm 이름만 보고 성능을 판단하지 않습니다.

- `vector::push_back`은 평균 상수 시간이지만 재할당 시 전체 이동이 필요합니다.
- `map::find`는 로그 시간이지만 노드 할당과 pointer 추적 비용이 있습니다.
- view는 중간 container 할당을 줄일 수 있지만 같은 계산을 반복할 수 있습니다.
- `reserve()`는 예상 크기를 알 때 재할당을 줄입니다.

측정할 때는 parsing과 출력 시간을 정렬 시간에 섞지 않고, Release build와 입력 크기를 기록합니다.

## 자주 놓치는 문제

- 정렬 결과만 필요하지만 원본을 직접 정렬합니다.
- view나 `reference_wrapper`를 원본보다 오래 저장합니다.
- 비교 함수가 strict weak ordering을 지키지 않습니다.
- 같은 key의 순서를 정하지 않아 test가 구현 세부사항에 의존합니다.
- template 오류를 줄이려다 실제로 필요하지 않은 concept을 추가합니다.
- container를 습관으로 선택하고 주요 연산의 복잡도를 확인하지 않습니다.

## 완료 기준

- 필요한 연산을 기준으로 container를 선택합니다.
- 직접 loop와 표준 algorithm 중 더 구체적인 표현을 고릅니다.
- iterator와 view의 무효화 조건을 설명합니다.
- 비소유 결과의 원본 수명을 명시합니다.
- tie-breaker를 포함한 결정적인 정렬을 구현합니다.
- template이 요구하는 연산을 concept 또는 문서로 제한합니다.
