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



### 引用

1. https://mojotv.cn/algorithm/golang-quick-sort