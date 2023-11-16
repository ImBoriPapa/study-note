# Linear Search(선형탐색)

- 배열이나 리스트와 같은 데이터 구조에서 원하는 항목을 찾기 위해 처음부터 끝까지 하나씩 순차적으로 탐색하는 검색 알고리즘
- 단순하고 직관적이지만 배열이나 리스트가 큰 경우 성능이 저하될 수 있다.
- 정렬이 되지 않은 상태의 배열이나 리스트에서 값을 찾기 위해 사용

## 선형 탐색의 원리
1. 순차적 탐색:
- 배열이나 리스트의 처음부터 시작하여 원하는 항목을 찾을 때까지 순차적으로 탐색
2. 탐색과정:
- 배열이나 리스트의 첫 번째 요소부터 시작하여 각 요소를 순차적으로 검사
- 현재 요소가 찾고자 하는 항목과 일치하면 검색을 종료하고 해당 인덱스를 반환
- 모든 요소를 검사한 후에도 찾고자 하는 항목을 찾이 못하면 검색을 종료하고 -1을 반환
         
## 선형 탐색의 시간 복잡도:
- 최선의 경우: O(1) - 찾고자 하는 항목이 배열 또는 리스트에 첫 번째 요소에 있을 때.
- 회악의 경우: O(n) - 찾고자 하는 항목이 배열 또는 리스트에 마지막 요소에 있거나 없을때.
- 평균 경우: O(n) - 찾고자 하는 항목이 배열 또는 리스트의 어디에 있던 간에 모든 요소를 평균적으로 절반까지만 확인해야하기 때문에 선형적으로 증가

## 장단점
#### 장점 
- 데이터의 크기가 작을 경우나 정렬되어 있지 않은 경우 binary search(이진탐색 - 데이터가 정렬되어 있어야함) 보다 빠를 수 있다.
#### 단점
- 데이터의 크기가 커질수록 검색 속도가 급격히 느려진다.
  

## 구현
```java
public class LinearSearch {

    public static void main(String[] args) {
        final int ARRAY_SIZE = 100;
        final int target = 33;
        final int[] intArray = IntStream.rangeClosed(1, ARRAY_SIZE).toArray();

        final int result = linearSearch(intArray, target);

        printResult(result);
    }

    private static void printResult(int result) {
        final String resultIndex = result == -1 ? "There is no target value" : String.valueOf(result);

        System.out.println("Index of Target: " + resultIndex);
    }

    /**
     * Simple int 배열 선형 탐색
     * @param array: int 배열
     * @param target: 찾을 int 값
     * @return target 이 있을 경우 target 이 배열에서 가지고 있는 인덱스 target 이 없을 경우 -1 반환
     */
    public static int linearSearch(final int[] array, final int target) {
        // 매개변수의 배열을 처음 부터 배열의 마지막(배열의 크기)까지 순회하면서 매개변수 target 과 일치하면 해당 인덱스를 반환
        for (int index = 0; index < array.length; index++) {
            if (array[index] == target) {
                return index;
            }
        }
        // 없을 경우 -1 반환
        return -1;
    }
}
```
  
