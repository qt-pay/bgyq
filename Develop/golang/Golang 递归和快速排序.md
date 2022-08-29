## Golang 递归和快速排序

### 快速排序

基本操作如图所示，这里还需要依靠递归来实现全体排序。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-quickly-sort-array-1.jpg)

算法实现：

1. 从数列中挑出一个元素,称为 “基准”(pivot)
2. 重新排序数列,所有元素比基准值小的摆放在基准前面,所有元素比基准值大的摆在基准的后面(相同的数可以到任一边).在这个分区退出之后,该基准就处于数列的中间位置.这个称为分区(partition)操作;
3. 递归地(recursive)把小于基准值元素的子数列和大于基准值元素的子数列排序;

递归的最底部情形,是数列的大小是零或一,也就是永远都已经被排序好了. 虽然一直递归下去,但是这个算法总会退出,因为在每次的迭代(iteration)中,它至少会把一个元素摆到它最后的位置去.

```go
// 将list分成两段，返回分割点
func partition(list []int, low, high int) int {
	pivot := list[low] //报错list[low]的值，即low 位置值为可以被覆盖
	for low < high {
		//high指针值 >= pivot high指针左移
		for low < high && pivot <= list[high] {
			high--
		}
		//填补low位置空值
		//high指针值 < pivot high值 移到low位置
		//high 位置值空
		list[low] = list[high]
		//low指针值 <= pivot low指针右移
		for low < high && pivot >= list[low] {
			low++
		}
		//填补high位置空值
		//low指针值 > pivot low值 移到high位置
		//low位置值空
		list[high] = list[low]
	}
	//pivot 填补 low位置的空值
	list[low] = pivot
    // 这里low和high是重合的，是此次排序的分割点
	return low
}

func QuickSort(list []int,low,high int)  {
	if high > low{
		//位置划分
		pivot := partition(list,low,high)
		//左边部分排序
		QuickSort(list,low,pivot-1)
		//右边排序
		QuickSort(list,pivot+1,high)
	}
}


func main()  {
	s := []int{2,44,4,8,33,1,0,22,-11,6,34,55,54,9,-4,-9}
	QuickSort(s,0,len(s)-1)
	fmt.Println(s)
}

// output
[-11 -9 -4 0 1 2 4 6 8 9 22 33 34 44 54 55]
```

end

### 递归条件

如果一个函数在内部调用自身本身，这个函数就是递归函数。

编写递归函数时，必须告诉它何时停止递归。正因为如此，每个递归函数都有两部分：基线条件 （base case）和递归条件 （recursive case）。递归条件指的是函数调用自己，而基线条件则指的是函数不再 调用自己，从而避免形成无限循环。



### 引用

1. https://www.jianshu.com/p/c529341a02e6
2. 【大量golang笔记】https://mojotv.cn/algorithm/golang-quick-sort