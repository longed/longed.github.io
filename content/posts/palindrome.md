---
title: "Palindrome"
date: 2023-12-17T18:01:59+08:00
draft: true
---

在终端上多次输入字符，程序把多次输入的字符拼接到一起打印到终端上，学习编程语言时经常会写这样的程序。
```C
#include <stdio.h>
#include <stdlib.h>

void input_output() {
  int nums[3] = {0};
  for (int i = 0; i < 3; ++i) {
    printf("input integer:");
    scanf("%d", &nums[i]);
  }
  for (int i = 0; i < 3; ++i) {
    printf("%d", nums[i]);
  }
}

int main() { input_output(); }

```
这样的小程序和回文结合起来时，我觉得更有意思。假设在终端里一次输入一个字符，输入 Y 次，在结束最后一次输入时，判断输入的字符串是不是回文。

回文的定义是"reads the same backwards as forwards"，即从前开始读和从后开始读得到的字符串一样。如果字符串是回文串，它的第一个字符是'x'，那么它的最后一个字符肯定也是'x'。

我最先想到的是和上面的终端打印字符程序的方法一样，用一个数组保存每次输入的字符，在读取到最后一个字符时，逆序将数组里的字符打印出来，如果逆序得到的字符串和输入的字符串相等，那么输入的字符串就是回文串。
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void input_output() {
  int nums[3] = {0};
  for (int i = 0; i < 3; ++i) {
    printf("input integer:");
    scanf("%d", &nums[i]);
  }
  for (int i = 0; i < 3; ++i) {
    printf("%d", nums[i]);
  }
}

void palindrome_v1() {
  char input_string[4] = {'\0'};
  for (int i = 0; i < 3; ++i) {
    printf("input character:");
    input_string[i] = getchar();
    getchar();
  }

  char palindrome_string[4] = {'\0'};
  for (int i = 2; i >= 0; --i) {
    palindrome_string[2 - i] = input_string[i];
  }

  if (strcmp(input_string, palindrome_string) == 0) {
    printf("It's palindrome!\n");
  } else {
    printf("No, it's not.\n");
  }
}

int main() { palindrome_v1(); }
```
这么写可以判断字符串是不是回文字符串，但是不够高效，因为输入、得到逆序串、判等在一起时间复杂度为 O(n) + O(n) + O(log(n))。（假设排序算法时间复杂度是 O(log(n)) )  

至少还有一个优化点，那就是"得到逆序串"。我们可以**实时**地从控制台读取字符并计算出已经输入的字符们的逆序，不用等待拿到字符串所有字符后再计算，这样可以用一个迭代做完；当输入字符串的最后一个字符后，此字符串的逆序也就能拿到。
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void input_output() {
  int nums[3] = {0};
  for (int i = 0; i < 3; ++i) {
    printf("input integer:");
    scanf("%d", &nums[i]);
  }
  for (int i = 0; i < 3; ++i) {
    printf("%d", nums[i]);
  }
}

void palindrome_v1() {
  char input_string[4] = {'\0'};
  for (int i = 0; i < 3; ++i) {
    printf("input character:");
    input_string[i] = getchar();
    getchar();
  }

  char reverse_string[4] = {'\0'};
  for (int i = 2; i >= 0; --i) {
    reverse_string[2 - i] = input_string[i];
  }

  if (strcmp(input_string, reverse_string) == 0) {
    printf("It's palindrome!\n");
  } else {
    printf("No, it's not.\n");
  }
}

void palindrome_v2() {
  char input_string[4] = {'\0'};
  char reverse_string[4] = {'\0'};
  for (int i = 0; i < 3; ++i) {
    printf("input character:");
    input_string[i] = getchar();
    getchar();
    reverse_string[2 - i] = input_string[i];
  }

  if (strcmp(input_string, reverse_string) == 0) {
    printf("It's palindrome!\n");
  } else {
    printf("No, it's not.\n");
  }
}

int main() { palindrome_v2(); }
```