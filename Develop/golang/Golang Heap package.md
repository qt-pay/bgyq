## Golang Heap package

### What is heap？

堆是一个完全二叉树。

若设二叉树的深度为h，除第 h 层外，其它各层 (1～h-1) 的结点数都达到最大个数，第 h 层所有的结点都连续集中在最左边，这就是完全二叉树。完全二叉树的要求：

* 任何一个节点不能只有右子树没有左子树
* 叶子节点出现在最后一层或者倒数第二层，不能再往上

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/heap-1.webp)

* 1,2为 大顶堆，大顶堆每个节点值都大于等于其左右节点的值
* 3 小顶堆
* 4 不是堆

堆的最大值和最小值，很好取，但是第二大和第三大就得变量了。如下：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/max-haep-sort.png)

### proof left node is 2i+1：

Lets look at an example of a small complete binary tree

```
Nodes                                   Level   Number of nodes in this level
1                                       1     | 2^0  
2 3                                     2     | 2^1 
4 5 6 7                                 3     | 2^2
8 9 10 11 12 13 14 15                   4     | 2^3
```

I'm going to use 3 variables to make things shorter: L the level in which a node exists, g the global index of a node, s the subindex of a node in a level (the index of that node in that level). More explanation of these variables will come ahead

**Notice that for this whole proof, i'm not starting my indexes at zero but at one to make things easier.**

Notice that at each level the number of nodes in that level is 2^(L-1). Also since each level has 2^(L-1) nodes then the index at that level starts at 2^(L-1). Based on the previous info we get that

```
g = 2^(L-1) + s - 1
```

so for example the global index of the second node in the third level is（第三层的第二个节点）

```
g = 2^(3-1) + 2 - 1 = 4 + 2 + 1 = 5
```

now lets say we want to get the left child of the node (6), here is what we need to do, we need to walk 1 more step to the end of the current level and then for each node that came before node (6) we need to walk two steps (since each node has 2 children) and then one more step to get to the first (left child).

> leftchildindex由三部分组成：parentIndex 、父节点所在Tree_Level_Max_Num和subindex of childr

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/heap-subindex-left-child-node-proof.png)

transforming this into an equation we get

```
leftChildIndex = parentIndex + (2^(L-1)-s) + 2(s-1) + 1
```

where (2^(L-1)-s) is the steps that is left to reach the end of current level, and 2(s-1) is 2 steps we need to take for each node that came before the current node.

As an example say we want to get the left child of node (6), this means first we need to move 1 step to reach end of Level 3, then we need to take 4 steps (because of nodes 4,5 having two children's each) and one step.

so for first child of node (6) we get

```
leftChildIndex = 6 + (2^(3-1) - 3) + 2(3-1) + 1 = 6 + 1 + 4 + 1= 12
```

so now that we saw the equation works lets get back to it

```
leftChildIndex = parentIndex + (2^(L-1)-s) + 2(s-1) + 1 = parentIndex + 2^(L-1) - s + 2s  -2 + 1 = parentIndex + 2^(L-1) + s - 1 = parentIndex + g = g + g = 2g
```

we can replace parentIndex with g so we get leftChildIndex = 2g as expected

I got 2g instead of 2g + 1 because i started my indexes at 1 instead of 0, starting them at zero will get you 2g + 1.

#### another simple proof

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/array-and-heap.jpg)

如下所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/heap-tree-and-array.png)

当下标从1开始时，满足任意节点i的左子节点为2i，右子节点为2i+1，但是转换成计算机中的二进制数字，则下标是从0开始的，相当于数组整体减一，因此可以得出：

对于任意节点i，左子节点为2i+1，右子节点为2i+2.

**假设第 n/2+1 的节点（下标为n/2）不是叶子结点，则其左子节点下标为 (n/2)  \* 2 = n ，超过了数组元素 下班最大值`n-1`，显然不符合逻辑，由此可以证明 n / 2 (下标则为n/2-1) 开始的元素一定是叶子节点**。

eg：堆数组N长度为5，则第三个元素一定是叶子节点。

> 完全二叉树总共有N个节点，父节点的取值范围必然是小于等于N/2的，因为每层个数都是`2^n`，n从0开始。

举例：

当数组下标为0时，父节点就是a[0]，左孩子是a[1],右孩子是a[2]
当数组下标为1时，父节点就是a[0]，左孩子是a[3],右孩子是a[4]
当数组下标为2时，父节点就是a[0]，左孩子是a[5],右孩子是a[6]

### 建堆: heapify

完全二叉树可以用一个数组表示，不需要指针，所以效率更高。

一个N给节点的堆，一定有`N/2`个根节点。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/heap-in-array.png)

#### 定义heap

```go
package main

import "fmt"

type heap struct {
	m   []int
	len int //堆中有多少元素
}

func main() {
	m := []int{1, 9, 3, 2, 7, 66,27,8,6,5} //第0个下标不放目标元素
	h := buildHeap(m)               //建堆，返回一个heap结构
	//h.Push(50)
	//h.Pop()
	fmt.Println(h.m)
}

/**
建堆，就是在原切片上操作，形成堆结构
只要按照顺序，把切片下标为n/2到1的节点依次堆化，最后就会把整个切片堆化
*/
func buildHeap(m []int) *heap {
	n := len(m)
	for i := n/2 -1;i>=0; i-- {
		heapf(m, n, i)
	}
	return &heap{m, n}
}
//对下标为i的节点进行堆化， n表示堆的最后一个节点下标
//2i,2i+1
// 自上往下堆化
func heapf(m []int, n, i int) {
	for {
		maxPos := i
		left := 2 *i + 1
		right := left +1
		if left >= n || left < 0{ // j1 < 0 after int overflow
			break
		}
        // ①
		if  m[left] < m[i] {
			maxPos = left
		}
        // ②
		if  right < n && m[left] > m[right] {
			maxPos = right
		}
        // ③
		if maxPos == i || m[i] < m[maxPos]{ //如果i节点位置正确，则退出
			break
		}
		m[i], m[maxPos] = m[maxPos], m[i]
		i = maxPos
	}
}
//修改①②③，切换大顶堆和小顶堆
// output
[1 2 3 6 5 66 27 8 9 7]
```

确实不优雅

#### heap package

需要实现`pop`和`push`接口

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int

func main() {
	h := &IntHeap{1, 9, 3, 2, 7, 66,27,8,6,5}
	heap.Init(h)
	fmt.Println("heap:",*h)

}

func (h IntHeap) Len() int { return len(h) }

func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] } // 这里决定 大小顶堆 现在是小顶堆

func (h IntHeap) Swap(i, j int) {
	h[i], h[j] = h[j], h[i]
}

func (h *IntHeap) Pop() interface{} {
	return "pop"
}

func (h *IntHeap) Push(x interface{}) { // 绑定push方法，插入新元素
	return
}

// 
heap: [1 2 3 6 5 66 27 8 9 7]
```



### Methods provided by container/heap

After understanding what a heap is, let’s take a look at the `container/heap` package.

The heap package provides heap methods for types that implement `heap.Interface`: Init/Push/Pop/Remove/Fix. `container/heap` is a minimum-valued heap, i.e., the value of each node is less than the value of all elements of its subtree (A heap is a tree with the property that each node is the minimum-valued node in its subtree).

```go
// The Interface type describes the requirements
// for a type using the routines in this package.
// Any type that implements it may be used as a
// min-heap with the following invariants (established after
// Init has been called or if the data is empty or sorted):
//
//	!h.Less(j, i) for 0 <= i < h.Len() and 2*i+1 <= j <= 2*i+2 and j < h.Len()
//
// Note that Push and Pop in this interface are for package heap's
// implementation to call. To add and remove things from the heap,
// use heap.Push and heap.Pop.
type Interface interface {
	sort.Interface
	Push(x interface{}) // add x as element Len()
	Pop() interface{}   // remove and return element Len() - 1.
}

```

Since `heap.Interface` contains `sort.Interface`, the target type needs to contain the following methods: Len/Less/Swap, Push/Pop.

### How heap is done

The example given above is amazing to say the least. How does `container/heap` do it?

Go 在它的标准库 `container/heap` 也提供了堆的定义，它匿名嵌套的 `sort.Interface` 的定义如下：

```go
type Interface interface {
 // Len is the number of elements in the collection.
 Len() int
 // Less reports whether the element with
 // index i should sort before the element with index j.
 Less(i, j int) bool
 // Swap swaps the elements with indexes i and j.
 Swap(i, j int)
}
```

通过自定义实现`Less()`和`Swap()`可以控制生成大顶堆还是小顶堆。

#### heap.Init

Let’s first look at the `heap.Init` function.

```go
// A heap must be initialized before any of the heap operations
// can be used. Init is idempotent with respect to the heap invariants
// and may be called whenever the heap invariants may have been invalidated.
// Its complexity is O(n) where n = h.Len().
//
func Init(h Interface) {
    // heapify
    n := h.Len()
    for i := n/2 - 1; i >= 0; i-- {
        down(h, i, n)
    }
}
```

The key point is the down function.

```go
func down(h Interface, i0, n int) bool {
    i := i0
    for {
        j1 := 2*i + 1
        if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
            break
        }
        j := j1 // left child
        if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
            j = j2 // = 2*i + 2  // right child
        }
        if !h.Less(j, i) {
            break
        }
        h.Swap(i, j)
        i = j
    }
    return i > i0
}
```

The function of the down function is very simple: given the type, the index of the element in the array that needs to be sunk, the length of the heap, the element is sunk to the appropriate position in the subtree corresponding to that element, thus satisfying the requirement that the subtree be the smallest heap.

Remember the previous diagram of the sequential array representation of the heap? Combine this with the implementation of the down function: pick any element i, compare it with its children 2*i+1 and 2*i+2, and if element i is smaller than its children, swap element i with the smaller of the two children (j), thus ensuring that the requirement of the smallest tree is satisfied (first down); child j may also have its children, continue comparing and swapping until the end of the array or element i is smaller than both of its children, jumping out of the loop.

Why is it that element i, which is smaller than both of its children, can jump out of the loop and not continue? This is because, in the Init function, the first element to start down (sink) is the n/2 - 1th one, and it is guaranteed to always start down from the last subtree (as in the previous figure, n=8 or n=9, n/2-1 is always 4), so it is guaranteed that when Init->down, if element i is smaller than both of its children, then the subtree corresponding to that element, is the minimum heap.

Init, after traversal, guarantees that the array to be Init is a minimal heap.

####  heap.Push

Let’s see again how `heap.Push` guarantees that the sequential array remains a minimal heap when new elements are inserted.

```go
// Push pushes the element x onto the heap. The complexity is
// O(log(n)) where n = h.Len().
func Push(h Interface, x interface{}) {
    h.Push(x)
    up(h, h.Len()-1)
}
```

First call `h.Push` to push the elements into the user-defined type, the aforementioned `PriorityQueue`. The array append, there is nothing to say. Since the element is inserted at the end of the array, the up function needs to be called to “float” it.

Let’s see how up floats.

```go
func up(h Interface, j int) {
    for {
        i := (j - 1) / 2 // parent
        if i == j || !h.Less(j, i) {
            break
        }
        h.Swap(i, j)
        j = i
    }
}
```

If element j is smaller than parent i, the two nodes are swapped and the comparison continues to the next higher parent until the root node, or element j, is larger than parent i.

In this way, it is ensured that the sequential array in which the new elements are inserted remains a minimal heap after up.

#### heap.Pop

```go
// Pop removes the minimum element (according to Less) from the heap
// and returns it. The complexity is O(log(n)) where n = h.Len().
// It is equivalent to Remove(h, 0).
func Pop(h Interface) interface{} {
    n := h.Len() - 1
    h.Swap(0, n)
    down(h, 0, n)
    return h.Pop()
}
```

The previous Pop function of `PriorityQueue` actually takes the :n-1 sub-array of the sequential array, so the purpose of heap.Pop is to swap the root node (0) with the element of the last node, and sink the element of the new root node to the right position to satisfy the requirement of the minimum heap; finally, call the Pop function of `PriorityQueue` again to get the last element.

#### heap.Fix

The update function of `PriorityQueue` actually relies on heap.Fix when modifying the priority of the elements.

```go
// Fix re-establishes the heap ordering after the element at index i has changed its value.
// Changing the value of the element at index i and then calling Fix is equivalent to,
// but less expensive than, calling Remove(h, i) followed by a Push of the new value.
// The complexity is O(log(n)) where n = h.Len().
func Fix(h Interface, i int) {
    if !down(h, i, h.Len()) {
        up(h, i)
    }
}
```

The code is relatively clear: if it can sink, it sinks, otherwise it floats. the return value of down can express whether there has been a sink (i.e. whether there has been a swap).

#### heap.Remove

The Remove function is not used in the priority queue example, so let’s look at the code directly.

```go
// Remove removes the element at index i from the heap.
// The complexity is O(log(n)) where n = h.Len().
//
func Remove(h Interface, i int) interface{} {
    n := h.Len() - 1
    if n != i {
        h.Swap(i, n)
        if !down(h, i, n) {
            up(h, i)
        }
    }
    return h.Pop()
}
```

First swap the node i to be deleted with the last node n, and then sink or float the new node i to the appropriate position. This piece of logic is similar to Fix, but note that you cannot call `heap.Fix` directly, the last element is to be deleted and cannot participate in Fix.

### 堆的应用

堆排序虽然不常用，但堆在生产中的应用还是很多的，这里我们详细来看堆在生产中的几个重要应用

1、 优先级队列

我们知道队列都是先进先出的，而在优先级队列中，元素被赋予了权重的概念，权重高的元素优先执行，执行完之后下次再执行权重第二高的元素...，显然用堆来实现优先级队列再合适不过了，只要用一个大顶堆来实现优先级队列即可，当权重最高的队列执行完毕，将其移除（相当于删除堆顶），再选出优先级第二高的元素（堆化让其符合大顶堆 的条件），很方便，实际上我们查看源码就知道， Java 中优先级队列  PriorityQueue 就是用堆来实现的。

2、 求 TopK 问题

怎样求出 n 个元素中前 K 个最大/最小的元素呢。假设我们要求前 K 个最大的元素，我们可以按如下步骤来做

1. 取 n 个元素的前 K 个元素构建一个小顶堆
2. 遍历第 K + 1 到  n 之间的元素，每一个元素都与小顶堆的堆顶元素进行比较，如果小于堆顶元素，不做任何操作，如果大于堆顶元素，则将堆顶元素替换成当前遍历的元素，再堆化以让其满足小顶的要求，这样遍历完成后此小顶堆的所有元素就是我们要求的 TopK。

每个元素堆化的时间复杂度是 O(logK)，n 个元素时间复杂度是 O(nlogK)，还是相当给力的！

3、 TP99 是生产中的一个非常重要的指标，如何快速计算

先来解释下什么是 TP99，它指的是在一个时间段内（如5分钟），统计某个接口（或方法）每次调用所消耗的时间，并将这些时间按从小到大的顺序进行排序，取第99%的那个值作为 TP99 值，举个例子， 假设这个方法在 5 分钟内调用消耗时间为从 1 s 到 100 s 共 100 个数，则其 TP99 为 99，这个值为啥重要呢，对于某个接口来说，这个值越低，代表 99% 的请求都是非常快的，说明这个接口性能很好，反之，就说明这个接口需要改进，那怎么去求这个值呢？

思路如下：

1. 创建一个大顶堆和一个小顶堆，大顶堆的堆顶元素比小顶堆的堆顶元素更小，大顶堆维护 99% 的请求时间，小顶堆维护 1% 的请求时间
2. 每产生一个元素（请求时间），如果它比大顶堆的堆顶元素小，则将其放入到大顶堆中，如果它比小顶堆的堆顶元素大，则将其插入到小顶堆中，插入后当然要堆化以让其符合大小顶堆的要求。
3. 上一步在插入的过程中需要注意一下，可能会导致大顶堆和小顶堆中元素的比例不为 99:1，此时就要做相应的调整，如果在将元素插入大顶堆之后，发现比例大于 99：1，将需将大顶堆的堆顶元素移到小顶堆中，再对两个堆堆化以让其符合大小顶堆的要求，同理，如果发现比例小于 99: 1，则需要将小顶堆的堆顶元素移到大顶堆来，再对两者进行堆化。

以上的大小顶堆调整后，则大顶堆的堆顶元素值就是所要求的 TP99 值。

有人可能会说以上的这些应用貌似用快排或其他排序也能实现，没错，确实能实现，但是我们需要注意到，在**静态数据**下用快排确实没问题，但在动态数据上，如果每插入/删除一个元素对所有的元素进行快排，其实效率不是很高，由于要快排要全量排序，时间复杂度是 O(nlog n)，而堆排序就非常适合这种**对于动态数据的排序**,对于每个新添加的动态数据，将其插入到堆中，然后进行堆化，时间复杂度只有 O(logK)

#### Priority Queue Demo

The container/heap package can be used to construct a priority queue.

Take the priority queue of go src as an example.

Define the `PriorityQueue` type.

```go
// An Item is something we manage in a priority queue.
type Item struct {
    value    string // The value of the item; arbitrary.
    priority int    // The priority of the item in the queue.
    // The index is needed by update and is maintained by the heap.Interface methods.
    index int // The index of the item in the heap.
}

// A PriorityQueue implements heap.Interface and holds Items.
type PriorityQueue []*Item

func (pq PriorityQueue) Len() int { return len(pq) }

func (pq PriorityQueue) Less(i, j int) bool {
    // We want Pop to give us the highest, not lowest, priority so we use greater than here.
    return pq[i].priority > pq[j].priority
}

func (pq PriorityQueue) Swap(i, j int) {
    pq[i], pq[j] = pq[j], pq[i]
    pq[i].index = i
    pq[j].index = j
}

func (pq *PriorityQueue) Push(x interface{}) {
    n := len(*pq)
    item := x.(*Item)
    item.index = n
    *pq = append(*pq, item)
}

func (pq *PriorityQueue) Pop() interface{} {
    old := *pq
    n := len(old)
    item := old[n-1]
    item.index = -1 // for safety
    *pq = old[0 : n-1]
    return item
}

// update modifies the priority and value of an Item in the queue.
func (pq *PriorityQueue) update(item *Item, value string, priority int) {
    item.value = value
    item.priority = priority
    heap.Fix(pq, item.index)
}
```

`PriorityQueue` is essentially an *Item array, whose Len/Less/Swap are more common arrays used to sort the functions that need to be defined, while Push, Pop are methods that use arrays to insert,. `PriorityQueue` also provides the update method. Note that since you usually want the priority queue to pop out the highest priority elements, the Less method is written in reverse.

After defining the above methods, `PriorityQueue` is ready to use the `container/heap` package.

In the following code, a pq array is defined from the items map, the length is the size of the hash, and `heap.Init` is called to initialize the pq array; after that, an element with priority 1 is added to the queue and the queue is updated; finally, the elements are popped from the queue accordingly, and it can be seen that the elements are sorted according to the priority when they are popped.

```go
// This example creates a PriorityQueue with some items, adds and manipulates an item,
// and then removes the items in priority order.
func Example_priorityQueue() {
    // Some items and their priorities.
    items := map[string]int{
        "banana": 3, "apple": 2, "pear": 4,
    }

    // Create a priority queue, put the items in it, and
    // establish the priority queue (heap) invariants.
    pq := make(PriorityQueue, len(items))
    i := 0
    for value, priority := range items {
        pq[i] = &Item{
            value:    value,
            priority: priority,
            index:    i,
        }
        i++
    }
    heap.Init(&pq)

    // Insert a new item and then modify its priority.
    item := &Item{
        value:    "orange",
        priority: 1,
    }
    heap.Push(&pq, item)
    pq.update(item, item.value, 5)

    // Take the items out; they arrive in decreasing priority order.
    for pq.Len() > 0 {
        item := heap.Pop(&pq).(*Item)
        fmt.Printf("%.2d:%s ", item.priority, item.value)
    }
    // Output:
    // 05:orange 04:pear 03:banana 02:apple
}
```



### 引用

1. https://medium.com/@verdi/binary-heap-basics-40a0f3f41c8f
2. https://www.sobyte.net/post/2022-07/go-heap/#4-how-heap-is-done
3. https://juejin.cn/post/6854573217269415950
4. https://blog.csdn.net/EDDYCJY/article/details/124642389