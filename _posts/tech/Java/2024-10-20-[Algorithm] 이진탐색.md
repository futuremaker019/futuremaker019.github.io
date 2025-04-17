---
layout: post
categories: [Tech, Algorithm]
tags: [BinarySearch, Algorithm, ETC]
---

이진 트리는 자식 노드가 최대 2개인 트리를 말한다.

또한 이진 트리는 데이터 크기를 따져 크기가 작으면 왼쪽 자식에 위치시키고, 크거나 같으면 오른쪽 자식 위치에 배치하는 독특한 정렬 방식을 갖는다. 이러한 구성으로 찾고자하는 값이 현재 노드의 값보다 작으면 왼쪽, 크면 오른쪽으로 이동하여 찾고자하는 값을 검색한다.

> 이진 탐색의 `시간 복잡도`는 트리 균형에 의존한다. 균형 잡힌 트리의 경우 삽입과 탐색 연산시 이진 탐색 트리에 저장된 노드가 N개라면 시간 복잡도는 `O(logN)`이다. 하지만 균형이 맞지 않은 트리(치우쳐진 형태의 트리)의 경우에는 시간 복잡도가 `O(N)` 이 필요하다.

이진 탐색은 `정렬된` 배열 또는 리스트에서 값을 찾을때 최적화된 방법이다. (반드시 정렬된 배열, 리스트라는것을 기억하자)

방식은 아래와 같다.

- left, right 범위 설정 (left는 맨 좌측 인데스를 right는 맨 오른쪽 인덱스다.)
- mid 인덱스 값을 설정한다. (`((왼쪽 인덱스 / 오른쪽 인덱스) / 2)`)
- mid의 값과 구하고자하는 값을 비교하여 
    - value < array[mid] : left ~ mid - 1 (mid - 1 은 중앙 인덱스를 포함하지 않은 오른쪽 인덱스로 재정의 함)
    - value > array[mid] : mid + 1 ~ right (mid + 1 은 중앙 인덱스를 포함하지 않은 왼쪽 인덱스로 재정의 함)
    - value == arrar[mid] : 값을 찾았으므로 return
- left와 right 의 값이 줄어들거나 늘어나면서 left가 right 보다 커지게 될 때를 종료 조건으로 만들어준다.


## 코드

반복문을 이용한 풀이

```java
public static int solution(int[] arr, int value) {
    Arrays.sort(arr);                           // 배열을 정렬한다.

    int left = 0;                               // left 인덱스 초기화
    int right = arr.length - 1;                 // right 인덱스 초기화

    while (left <= right) {
        int mid = (left + right) / 2;           // 중앙 인덱스 초기화

        if (value == arr[mid]) {                // value를 발견했다면 값에 해당하는 인덱스를 반환
            return mid;                       
        } else if (value > arr[mid]) {          // 중앙값보다 value가 크다면 오른쪽에서 찾기 위해 구간수정
            left = mid + 1;
        } else {                                // 중앙값보다 value가 작다면 왼쪽에서 찾기 위해 구간수정
            right = mid - 1;
        }
    }

    return -1;                                  // 찾고자 하는 값이 배열에 없다면 -1 반환
}
```

재귀를 이용한 풀이

```java
public static int initRecursive(int[] arr, int value) {
    Arrays.sort(arr);                                   // 배열 정렬
    return recursive(arr, value, 0, arr.length - 1);
}

private static int recursive(int[] arr, int value, int left, int right) {
    if (left > right) return -1;                        // 검색이 종료되고도 값을 찾지못한다면 -1을 반환

    int mid = (left + right) / 2;

    if (arr[mid] == value) {
        return mid;
    } else if (arr[mid] < value) {
        return recursive(arr, value, mid + 1, right);
    } else {
        return recursive(arr, value, left, mid - 1);
    }
}
```

---

참고
- 코딩 테스트 합격자 되기 - 서적
- [자바로 배우는 자료구조](https://www.youtube.com/watch?v=waPwUJu0-lo&list=PLlTylS8uB2fCoXCVtfJ0wzotOU8a3ti6M)
