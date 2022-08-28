## LeetCode-Go

### 线性表

### 链表

### 堆

### 树

### 图

### 题目



#### two sum

Given an array of integers, return indices of the two numbers such that they add up to a specific target.

You may assume that each input would have exactly one solution, and you may not use the same element twice.

**Example**:

```
Given nums = [2, 7, 11, 15], target = 9,

Because nums[0] + nums[1] = 2 + 7 = 9,
return [0, 1]
```

最笨的方法，两个for嵌套遍历nums，最坏的结果是最后两位数的才是期望数字。则时间复杂度为O(n*(n-1)/2).

```go
func twoSum(nums []int, target int) []int {
    lenNums := len(nums)
    res := make([]int,2,2)
    if lenNums < 2{
        return res
    }
    for i:= 0; i < lenNums; i++{
        j := i + 1
        for ;j < lenNums; j++{
            if nums[i] + nums[j] == target{
                res[0] = i
                res[1] = j
            }    
        }
    }
    return res
}

```

大佬的方法，时间复杂度O(N)即只需要遍历一次

顺序扫描数组，对每一个元素，在 map 中找能组合给定值的另一半数字，如果找到了，直接返回 2 个数字的下标即可。如果找不到，就把这个数字存入 map 中(用于其他匹配另一半)，等待扫到“另一半”数字的时候，再取出来返回结果。

```go
package leetcode

func twoSum(nums []int, target int) []int {
	m := make(map[int]int)
	for i := 0; i < len(nums); i++ {
		another := target - nums[i]
		if _, ok := m[another]; ok {
			return []int{m[another], i}
		}
		m[nums[i]] = i
	}
	return nil
}

```

end

#### 快速排序

算法实现：

1. 从数列中挑出一个元素,称为 “基准”(pivot)
2. 重新排序数列,所有元素比基准值小的摆放在基准前面,所有元素比基准值大的摆在基准的后面(相同的数可以到任一边).在这个分区退出之后,该基准就处于数列的中间位置.这个称为分区(partition)操作;
3. 递归地(recursive)把小于基准值元素的子数列和大于基准值元素的子数列排序;

递归的最底部情形,是数列的大小是零或一,也就是永远都已经被排序好了. 虽然一直递归下去,但是这个算法总会退出,因为在每次的迭代(iteration)中,它至少会把一个元素摆到它最后的位置去.

```go
func partition(list []int, low, high int) int {
	pivot := list[low] //导致 low 位置值为空
	for low < high {
		//high指针值 >= pivot high指针👈移
		for low < high && pivot <= list[high] {
			high--
		}
		//填补low位置空值
		//high指针值 < pivot high值 移到low位置
		//high 位置值空
		list[low] = list[high]
		//low指针值 <= pivot low指针👉移
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

func TestQuickSort(t *testing.T) {
	list := []int{2,44,4,8,33,1,22,-11,6,34,55,54,9}
	QuickSort(list,0,len(list)-1)
	t.Log(list)
}
```



### 引用

1. https://mojotv.cn/algorithm/golang-quick-sort