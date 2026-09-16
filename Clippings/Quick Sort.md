---
title: "Quick Sort"
source: "https://www.geeksforgeeks.org/dsa/quick-sort-algorithm/"
author:
  - "[[GeeksforGeeks]]"
published: 2014-01-07
created: 2026-09-16
description: "Your All-in-One Learning Portal: GeeksforGeeks is a comprehensive educational platform that empowers learners across domains-spanning computer science and programming, school education, upskilling, commerce, software tools, competitive exams, and more."
tags:
  - "clippings"
---
QuickSort is a sorting algorithm based on the Divide and Conquer that picks an element as a pivot and partitions the given array around the picked pivot by placing the pivot in its correct position in the sorted array..

==There are mainly three steps in the algorithm:==

1. ****Choose a Pivot:**** Select an element from the array as the pivot. The choice of pivot can vary (e.g., first element, last element, random element, or median).
2. ****Partition the Array:**** Re arrange the array around the pivot. After partitioning, all elements smaller than the pivot will be on its left, and all elements greater than the pivot will be on its right.
3. ****Recursively Call:**** Recursively apply the same process to the two partitioned sub-arrays.
4. ****Base Case:**** The recursion stops when there is only one element left in the sub-array, as a single element is already sorted.
![Heap-Sort-Recursive-Illustration](https://media.geeksforgeeks.org/wp-content/uploads/20240926172924/Heap-Sort-Recursive-Illustration.webp)

### Choice of Pivot

There are many different choices for picking pivots.

- [Always pick the first (or last) element as a pivot](https://www.geeksforgeeks.org/dsa/implement-quicksort-with-first-element-as-pivot/). The below implementation picks the last element as pivot. The problem with this approach is it ends up in the worst case when array is already sorted.
- [Pick a random element as a pivot](https://www.geeksforgeeks.org/dsa/quicksort-using-random-pivoting/). This is a preferred approach because it does not have a pattern for which the worst case happens.
- Pick the median element is pivot. This is an ideal approach in terms of time complexity as [we can find median in linear time](https://www.geeksforgeeks.org/dsa/median-of-an-unsorted-array-in-liner-time-on/) and the partition function will always divide the input array into two halves. But it takes more time on average as median finding has high constants.

### Partition Algorithm

The key process in ****quickSort**** is a ****partition().**** There are three common algorithms to partition. All these algorithms have O(n) time complexity.

1. [****Naive Partition****](https://www.geeksforgeeks.org/dsa/naive-partition-algorithm/): Here we create copy of the array. First put all smaller elements and then all greater. Finally we copy the temporary array back to original array. This requires O(n) extra space.
2. [****Lomuto Partition****](https://www.geeksforgeeks.org/dsa/lomuto-partition-algorithm/): We have used this partition in this article. This is a simple algorithm, we keep track of index of smaller elements and keep swapping. We have used it here in this article because of its simplicity.
3. [****Hoare's Partition****](https://www.geeksforgeeks.org/dsa/hoare-s-partition-algorithm/): This is the fastest of all. Here we traverse array from both sides and keep swapping greater element on left with smaller on right while the array is not partitioned. Please refer [Hoare’s vs Lomuto](https://www.geeksforgeeks.org/dsa/hoares-vs-lomuto-partition-scheme-quicksort/) for details.

### Working of Lomuto Partition Algorithm with Illustration

> The logic is simple, we start from the leftmost element and keep track of the index of smaller (or equal) elements as ****i**** . While traversing, if we find a smaller element, we swap the current element with ****arr\[i\]****. Otherwise, we ignore the current element.

Let us understand the working of partition algorithm with the help of the following example:

### Illustration of QuickSort Algorithm

In the previous step, we looked at how thepartitioningprocess rearranges the array based on the chosen ****pivot****. Next, we apply the same method recursively to the smaller sub-arrays on the ****left**** and ****right**** of the pivot. Each time, we select new pivots and partition the arrays again. This process continues until only one element is left, which is always sorted. Once every element is in its correct position, the entire array is sorted.

Below image illustrates, how the recursive method calls for the smaller sub-arrays on the ****left**** and ****right**** of the ****pivot****:

![quick-sort--images](https://media.geeksforgeeks.org/wp-content/uploads/20240925173636/quick-sort--images.webp)

**Output**
```
1 5 7 8 9 10
```

### Complexity Analysis of Quick Sort

****Time Complexity:****

- ****Best Case:**** (Ω(n log n)), Occurs when the pivot element divides the array into two equal halves.
- ****Average Case**** (θ(n log n)), On average, the pivot divides the array into two parts, but not necessarily equal.
- ****Worst Case:**** (O(n²)), Occurs when the smallest or largest element is always chosen as the pivot (e.g., sorted arrays).

****Auxiliary Space:****

- ****Worst-case scenario:**** ****O(n)**** due to unbalanced partitioning leading to a skewed recursion tree requiring a call stack of size O(n).
- ****Best-case scenario: O(log n)**** as a result of balanced partitioning leading to a balanced recursion tree with a call stack of size O(log n).

Please refer [Time and Space Complexity Analysis of Quick Sort](https://www.geeksforgeeks.org/dsa/time-and-space-complexity-analysis-of-quick-sort/) for more details.

### Advantages of Quick Sort

- It is a divide-and-conquer algorithm that makes it easier to solve problems.
- It is efficient on large data sets.
- It has a low overhead, as it only requires a small amount of memory to function.
- It is Cache Friendly as we work on the same array to sort and do not copy data to any auxiliary array.
- Fastest general purpose algorithm for large data when stability is not required.
- It is [****tail recursive****](https://www.geeksforgeeks.org/dsa/tail-recursion/) and hence all the [tail call optimization](https://www.geeksforgeeks.org/dsa/quicksort-tail-call-optimization-reducing-worst-case-space-log-n/) can be done.

### Disadvantages of Quick Sort

- It has a worst-case time complexity of O(n <sup>2</sup>), which occurs when the pivot is chosen poorly.
- It is not a good choice for small data sets.
- It is not a stable sort, meaning that if two elements have the same key, their relative order will not be preserved in the sorted output in case of quick sort, because here we are swapping elements according to the pivot's position (without considering their original positions).

### Applications of Quick Sort

- Sorting large datasets efficiently in memory.
- Used in library sort functions (like C++ std::sort and Java Arrays.sort for primitives).
- Arranging records in databases for faster searching.
- Preprocessing step in algorithms requiring sorted input (e.g., binary search, two-pointer techniques).
- Finding the kth smallest/largest element using Quickselect (a variant of quicksort).
- Sorting arrays of objects based on multiple keys (custom comparators).
- Data compression algorithms (like Huffman coding preprocessing).
- Graphics and computational geometry (e.g., convex hull algorithms).

Please refer [Application of Quicksort](https://www.geeksforgeeks.org/dsa/application-and-uses-of-quicksort/) for more details.