---
title: "线性表的顺序存储算法pugm"
date: 2023-07-18T20:39:32+08:00
---
#### 线性表顺序存储的类型定义

以下代码是顺序表的定义， 可以保存在 seqlist.h

数据类型定义

```c
const int Maxsize = 100; // 定义一个足够大的常数
// 定义一个数据类型
typedef struct 
{
    int num;
    char name[8];
} DataType;

// 定义一个存放数据的数组
typedef struct 
{
    DataType data[Maxsize];
    int length;
} SeqList;

```

插入算法即将元素插入到顺序表L的第i个元素之前

```c
int InsertSeqList(SeqList L, DataType x, int i)
{  // 将元素插入到顺序表L的第i个元素之前
    if (L.length == Maxsize)
    {
        printf("表已满\n");
        return -1;
    }

    if (i < 1 || i > L.length + 1)
    {
        printf("位置错误\n");
        return -1;
    }

    // 依次后移
    for (j = L.length; j >= i; j--)
    {
        L.data[j] = L.data[j - 1];
    }
    L.data[i - 1] = x;
    L.length++;
    return 0;
}
```
