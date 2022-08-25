## LeetCode-Go

### 线性表

### 链表

### 堆

https://driverzhang.github.io/post/go%E6%A0%87%E5%87%86%E5%BA%93%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E7%B3%BB%E5%88%97%E4%B9%8B%E5%A0%86heap/

#### 完全二叉树

堆是一个完全二叉树。若设二叉树的深度为h，除第 h 层外，其它各层 (1～h-1) 的结点数都达到最大个数，第 h 层所有的结点都连续集中在最左边，这就是完全二叉树。

完全二叉树可以用一个数组表示，不需要指针，所以效率更高。当用数组表示时，数组中任一位置i上的元素，其左儿子在位置2i上，右儿子在位置(2i+ 1)上，其父节点在位置(i/2)上。



#### 建堆

建堆，就是在原切片上操作，形成堆结构。
只要按照顺序，把切片下标为n/2到1的节点依次堆化，最后就会把整个切片堆化。

堆中每一个节点的值都必须大于等于（或小于等于）其子树中每个节点的值。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/heap-1.webp)

1,2为 大顶堆

3 小顶堆

4 不是堆

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

### 引用

1. https://gopherzhang-1992.gitbook.io/golang-server-side/shu-ju-jie-gou-yu-suan-fa/dui-heap