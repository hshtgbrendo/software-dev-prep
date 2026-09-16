There are mainly three steps in the algorithm:

1. ****Choose a Pivot:**** Select an element from the array as the pivot. The choice of pivot can vary (e.g., first element, last element, random element, or median).
2. ****Partition the Array:**** Re arrange the array around the pivot. After partitioning, all elements smaller than the pivot will be on its left, and all elements greater than the pivot will be on its right.
3. ****Recursively Call:**** Recursively apply the same process to the two partitioned sub-arrays.
4. ****Base Case:**** The recursion stops when there is only one element left in the sub-array, as a single element is already sorted.

![[Pasted image 20260916020748.png]]

### Partition Algorithm

The key process in ****quickSort**** is a ****partition().**** There are three common algorithms to partition. All these algorithms have O(n) time complexity.

1. [****Naive Partition****](https://www.geeksforgeeks.org/dsa/naive-partition-algorithm/): Here we create copy of the array. First put all smaller elements and then all greater. Finally we copy the temporary array back to original array. This requires O(n) extra space.
2. [****Lomuto Partition****](https://www.geeksforgeeks.org/dsa/lomuto-partition-algorithm/): We have used this partition in this article. This is a simple algorithm, we keep track of index of smaller elements and keep swapping. We have used it here in this article because of its simplicity.
3. [****Hoare's Partition****](https://www.geeksforgeeks.org/dsa/hoare-s-partition-algorithm/): This is the fastest of all. Here we traverse array from both sides and keep swapping greater element on left with smaller on right while the array is not partitioned. Please refer [Hoare’s vs Lomuto](https://www.geeksforgeeks.org/dsa/hoares-vs-lomuto-partition-scheme-quicksort/) for details.

### \[Approach\] - Hoare's Algorithm for Array Partition - O(n) Time and O(1) Space

> ****Hoare's partitioning algorithm**** is an efficient way to partition an array around a pivot. It’s based on [two pointers](https://www.geeksforgeeks.org/dsa/two-pointers-technique/) that start at opposite ends of the array and move toward each other until they find elements that need to be swapped.

Step-by-step explanation of algorithm:

- Consider the first element as the pivot and initialise two pointers, **`**i**`** at the start and **`**j**`** at the end of the array.
- Move **`**i**`** to the right until an element greater than or equal to the pivot is found, and move **`**j**`** to the left until an element less than or equal to the pivot is found.
- If **`**i**`** points to an element greater than or equal to the pivot and **`**j**`** points to an element less than or equal to the pivot, swap them.
- Repeat the process, moving **`**i**`** and **`**j**`** toward each other until they meet or cross.
- When the pointers cross, the partitioning is complete, with elements less than or equal to the pivot on the left and those greater than or equal to the pivot on the right.


```javascript
/**
 * @param {number[]} nums
 * @return {number[]}
 */
var sortArray = function(nums) {
    function hoares(arr, left, right) {
        let i = left - 1
        let j = right + 1
        let pivot = arr[Math.floor((left + right) / 2)]

        while (true) {
            do {
                i++
            } while (arr[i] < pivot)

            do {
                j--
            } while (arr[j] > pivot)

            if (i >= j) {
                return j
            }

            let temp = arr[j]
            arr[j] = arr[i]
            arr[i] = temp
        }
    }

    function sort(arr, left, right) {
        if (left >= right) {
            return arr
        }

        let partition = hoares(arr, left, right)

        sort(arr, left, partition)
        sort(arr, partition + 1, right)

        return arr
    }

    return sort(nums, 0, nums.length - 1)
};
```