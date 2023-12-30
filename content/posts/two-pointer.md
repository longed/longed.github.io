---
title: "Two pointer"
date: 2023-12-29T23:01:59+08:00
draft: true
---
etc
```c
#include <stdio.h>
#include <stdlib.h>

double findMedianSortedArrays(int *nums1, int nums1Size, int *nums2,
                              int nums2Size) {
  int toalSize = nums1Size + nums2Size;
  int median = toalSize / 2;
  int i = 0, j = 0;
  int cnt = 0;
  int pre = 0, cur = 0;
  while (i < nums1Size && j < nums2Size && cnt <= median) {
    pre = cur;
    if (*(nums1 + i) <= *(nums2 + j)) {
      cur = *(nums1 + i);
      ++i;
    } else {
      cur = *(nums2 + j);
      ++j;
    }
    ++cnt;
  }
  while (i < nums1Size && cnt <= median) {
    pre = cur;
    cur = *(nums1 + i);
    ++cnt;
    ++i;
  }
  while (j < nums2Size && cnt <= median) {
    pre = cur;
    cur = *(nums2 + j);
    ++cnt;
    ++j;
  }

  if (toalSize % 2 == 0) {
    return ((pre + cur) * 1.0) / 2;
  } else {
    return cur * 1.0;
  }
}

int main() {
  int num1[1] = {2};
  int num2[2] = {1, 3};

  double ret = findMedianSortedArrays(num1, 1, num2, 2);
  printf("%f", ret);
}

```
