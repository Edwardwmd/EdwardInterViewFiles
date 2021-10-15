## Java算法

#### 一、排序

<img src="E:\资料\个人整理远峰面试资料\Sources Image\排序算法时间和空间复杂度.webp" alt="排序算法时间和空间复杂度" style="zoom:80%;" />

算法时间复杂度：O(1)  <  O(log n)  <  O(n)  <  O(n log n)  <  O(n²)  <  O(n log² n)  <  O(n³)



+ ==**①冒泡排序：**==

![冒泡排序动态图](E:\资料\个人整理远峰面试资料\Sources Image\冒泡排序动态图.gif)

   

```kotlin
fun bubbleSort(arr: Array<Int>): Array<Int> {
    val len = arr.size
    var tempValue = 0
    for (i in 0 until len) {
        for (j in 0 until i) {
            //升序
            if (arr[i]<arr[j]){
                tempValue = arr[i]
                arr[i] = arr[j]
                arr[j] = tempValue
            }
        }
    }
    return arr
}
```



==**时间复杂度：**== 冒泡排序平均时间复杂度为O(n2)，最好时间复杂度为O(n)，最坏时间复杂度为O(n2)。
 **最好情况：**如果待排序元素本来是正序的，那么一趟冒泡排序就可以完成排序工作，比较和移动元素的次数分别是 (n - 1) 和 0，因此最好情况的时间复杂度为O(n)。
 **最坏情况：**如果待排序元素本来是逆序的，需要进行 (n - 1) 趟排序，所需比较和移动次数分别为 n * (n - 1) / 2和 3 * n * (n-1) / 2。因此最坏情况下的时间复杂度为O(n2)。

==**稳定性：**==当 array[j] == array[j+1] 的时候，我们不交换 array[i] 和 array[j]，所以冒泡排序是稳定的


+ ==**② 选择排序：**==

![选择排序动态图](E:\资料\个人整理远峰面试资料\Sources Image\选择排序动态图.gif)

```kotlin
fun selectSort(arr: Array<Int>): Array<Int> {
    val len = arr.size
    if (len == 0) return arr
    for (i in 0 until len) {
        var minIndex = i
        //先找到最小的那个数,获取最小数的index
        for (j in i until len) {
            if (arr[j] < arr[minIndex]) {
                minIndex = j
            }
        }
        val tempValue = arr[minIndex]
        arr[minIndex] = arr[i]
        arr[i] = tempValue
    }

    return arr
}
```

==**算法时间复杂度：**==简单选择排序平均时间复杂度为O(n2)，最好时间复杂度为O(n2)，最坏时间复杂度为O(n2)。
 **最好情况：**如果待排序元素本来是正序的，则移动元素次数为 0，但需要进行 n * (n - 1) / 2 次比较。
 **最坏情况：**如果待排序元素中第一个元素最大，其余元素从小到大排列，则仍然需要进行 n * (n - 1) / 2 次比较，且每趟排序都需要移动 3 次元素，即移动元素的次数为3 * (n - 1)次。需要注意的是，简单选择排序过程中需要进行的比较次数与初始状态下待排序元素的排列情况无关。

==**稳定性：**==简单选择排序不稳定，比如序列 2、4、2、1，我们知道第一趟排序第 1 个元素 2 会和 1 交换，那么原序列中 2 个 2 的相对前后顺序就被破坏了，所以简单选择排序不是一个稳定的排序算法。



==**③**==





④

⑤

⑥

⑦

⑧

⑨