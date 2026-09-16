---
title: "Hoare's Partition Algorithm"
source: "https://www.geeksforgeeks.org/dsa/hoare-s-partition-algorithm/"
author:
  - "[[GeeksforGeeks]]"
published: 2024-11-07
created: 2026-09-16
description: "Your All-in-One Learning Portal: GeeksforGeeks is a comprehensive educational platform that empowers learners across domains-spanning computer science and programming, school education, upskilling, commerce, software tools, competitive exams, and more."
tags:
  - "clippings"
---
Given an array **`**arr[]**`**, the task is to ****partition**** the array by assuming first element as ****pivot**** element.

The partition of an array must satisfy the following two conditions:

- Elements smaller than pivot element must appear at index less than or equal to partition index.
- Elements larger than or equal to pivot element must appear at index greater than partition index.

Partition index is equal to count of elements, strictly smaller than pivot, minus one.

****Note****: There might me more than one possible partition arrays.

****Examples****:

> ****Input:**** arr\[\] = \[5, 3, 8, 4, 2, 7, 1, 10\]  
> ****Output:**** \[1, 3, 2, ****4****, 8, 7, 5, 10\]  
> ****Explanation:**** The partition index is 3 and pivot element is 5, allelements smaller than pivot element \[1, 3, 2, 4\] were arranged before partition index and elements larger than or equal to pivot \[8, 7, 5, 10\] were arranged after partition index.
> 
> ****Input:**** arr\[\] = \[12, 10, 9, 16, 19, 9\]  
> ****Output:**** \[9, 10, ****9****, 16, 19, 12\]  
> ****Explanation:**** The partition index is 2 and pivot element is 12, allelements smaller than pivot element \[9, 10, 9\] were arranged before or at partition index and elements larger than or equal to pivot \[16, 19, 12\] were arranged after partition index.

Table of Content

- [\[Approach\] - Hoare's Algorithm for Array Partition - O(n) Time and O(1) Space](#hoares-algorithm-for-array-partition)
- [Some Interesting Facts](#some-interesting-facts)

### \[Approach\] - Hoare's Algorithm for Array Partition - O(n) Time and O(1) Space

> ****Hoare's partitioning algorithm**** is an efficient way to partition an array around a pivot. It’s based on [two pointers](https://www.geeksforgeeks.org/dsa/two-pointers-technique/) that start at opposite ends of the array and move toward each other until they find elements that need to be swapped.

Step-by-step explanation of algorithm:

- Consider the first element as the pivot and initialise two pointers, **`**i**`** at the start and **`**j**`** at the end of the array.
- Move **`**i**`** to the right until an element greater than or equal to the pivot is found, and move **`**j**`** to the left until an element less than or equal to the pivot is found.
- If **`**i**`** points to an element greater than or equal to the pivot and **`**j**`** points to an element less than or equal to the pivot, swap them.
- Repeat the process, moving **`**i**`** and **`**j**`** toward each other until they meet or cross.
- When the pointers cross, the partitioning is complete, with elements less than or equal to the pivot on the left and those greater than or equal to the pivot on the right.

**Output**
```
1 3 2 4 8 7 5 10
```

### Some Interesting Facts

- Hoare's Partition Algorithm is generally faster than Lomuto's because it performs fewer swaps and makes only one traversal of the array, leading to better time complexity in practice.
- It works in-place and does not require extra space, unlike the naive partitioning method which uses a temporary array.
- It can be used to implement a stable version of Quick Sort with the right adjustments, though it is not inherently stable.
- We can easily modify the algorithm to consider the first element (or any other element) as pivot by swapping first and last elements and then using the same code.