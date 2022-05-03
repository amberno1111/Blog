# [LeetCode]一文刷完所有的链表题
LeetCode上链表相关的题目，刷题技巧(俗称刷题套路)都是比较固定的，即使是一些Hard的题，其实也只是多种技巧的结合而已。
本文将会介绍所有的这些技巧，并且提供所有的链表相关的题的题解答，并标注这些题解所使用的刷题技巧。

链表的数据结构：
```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
```

## 一、刷链表题到底有哪些技巧
对于LeetCode上跟数据结构相关的题来说，其刷题技巧都是跟数据结构本身的特性绑定的，本质上就是该数据结构的增删改查的技巧，链表题也不例外。所以，本文会根据对链表结构的增删改查对刷题技巧进行分类。

### 1.1 链表节点的删除操作

删除链表中的某个节点，一般有几种场景：
- 只知道当前节点，删除当前节点
- 知道前一个节点，删除当前节点

#### 1.1.1 只知道当前节点时，删除当前节点

删除某个节点，只需要知道当前节点就够了，删除操作主要就两步：
- 把当前节点的值设置为下一个节点的值
- 把当前节点的next指针，指向下一个节点的next指针的指向节点

这个操作实际上是删除下一个节点，然后把下一个节点的val和next赋值给当前的节点，就达到了删除当前节点的目的。

- [237. 删除链表中的节点](https://leetcode-cn.com/problems/delete-node-in-a-linked-list/)
- [面试题 02.03. 删除中间节点](https://leetcode-cn.com/problems/delete-middle-node-lcci/submissions/)
```python
# 这两个题都是一样的，知道当前节点，删除当前节点
def deleteNode(self, node):
    node.val, node.next = node.next.val, node.next.next
```

#### 1.1.2 知道前一个节点，删除当前节点
删除操作如下：
```python
def deleteNode(self, pre):
    ## pre表示前一个节点，直接设置pre的next指针往后跳一个节点即可
    pre.next = pre.next.next
```

- [83. 删除排序链表中的重复元素](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-list/)
```python
def deleteDuplicates(self, head: ListNode) -> ListNode:
    # 边界条件
    if not head or not head.next:
        return head
    cur = head
    while cur.next:
        if cur.val == cur.next.val:
            # 删除cur.next节点，可以用pre.next=pre.next.next的技巧
            cur.next = cur.next.next
        else:
            # 不用删除就直接往后移动指针
            cur = cur.next
    return head
```
- [82. 删除排序链表中的重复元素 II](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-list-ii/)
```python
# 这个本83题不同的在于，要把重复的数字全删了，一个都不留
def deleteDuplicates(self, head: ListNode) -> ListNode:
    # 边界条件
    if not head or not head.next:
        return head
    cur = dummy = ListNode(-1, head)
    while cur.next and cur.next.next:
        if cur.next.val == cur.next.next.val:
            x = cur.next.val
            # 持续的把后面相同的值都删掉，这一小段其实跟83的相同
            while cur.next and cur.next.val == x:
                cur.next = cur.next.next
        else:
            cur = cur.next
    return dummy.next
```

- [203. 移除链表元素](https://leetcode-cn.com/problems/remove-linked-list-elements/)
```python
# 可以看到，这里就是一个简单的遍历找到节点然后运用删除链表的即可
def removeElements(self, head: ListNode, val: int) -> ListNode:
    # 去掉边界条件
    if not head:
        return head
    # 哑节点，这也是一个技巧，考虑这题的场景，当你想要删除的节点
    # 是head节点怎么办？？这时候用哑节点可以很轻易的解决问题
    # 一般链表题，使用哑节点的场景是：当头结点会动的时候就需要用哑节点，比如换位置，删除
    pre = dummy = ListNode(-1, head)
    while head:
        if head.val == val:
            # 利用上文所述的删除节点的方法
            pre.next = pre.next.next
        else:
            # 如果不删除，则pre指针往后移动一步
            pre = pre.next
        head = head.next
    # 使用哑节点的好处，是这里可以轻易的找到结果链表的第一个节点
    return dummy.next
```
- [剑指 Offer 18. 删除链表的节点](https://leetcode-cn.com/problems/shan-chu-lian-biao-de-jie-dian-lcof/) 这题就是想办法找到待删除节点的前一个节点，然后删掉目标节点。
```python
# 这题其实和上一题是一样的，不过就是这个题目本身的条件，
# 保证了链表中只有一个节点满足node.val == val，所以只需要删除一个节点即可
# 所以可以看到删除了一个节点以后，就可以直接break跳出循环
def deleteNode(self, head: ListNode, val: int) -> ListNode:
    if head is None:
        return head
    pre = dummy = ListNode(-1, head)
    while head:
        if head.val == val:
            pre.next = pre.next.next
            # 当删除了要删除的这一个节点以后，就可以直接跳出循环了
            break
        else:
            pre = pre.next
        head = head.next
    return dummy.next
```

- [面试题 02.01. 移除重复节点](https://leetcode-cn.com/problems/remove-duplicate-node-lcci/)
```python
# 一个简单空间换时间，加上删除节点的技巧
def removeDuplicateNodes(self, head: ListNode) -> ListNode:
    # 边界条件
    if not head or not head.next:
        return head
    occurred, pre, cur = {head.val}, head, head.next
    while cur:
        if cur.val in occurred:
            # 如果出现过，使用删除节点的技巧删除当前cur节点
            pre.next = cur.next
        else:
            # 没出现过，则保留cur节点，指针往后移
            pre = pre.next
            occurred.add(cur.val)
        cur = cur.next
    return head
```


### 1.2 链表节点的查询操作
在给定的链表中，查询某个节点，其实就是对链表的遍历操作。最简单的当然是从前往后一个节点一个节点的遍历，但在刷LeetCode的时候，用的最多的技巧是**双指针遍历**。

ps: 普通的一个一个节点遍历的技巧这里就不提了，太简单了。

双指针一半用来查找节点，也可以扩展到多指针，比如三指针、四指针等等，其原理都是一样的。

#### 1.2.1 查找中间节点

- [876. 链表的中间结点](https://leetcode-cn.com/problems/middle-of-the-linked-list/)
```python
def middleNode(head: ListNode) -> ListNode:
    # 边界条件最先排除
    # head 是 None，或者整个链表只有 head 一个节点
    if not head or not head.next:
        return head
    # 快慢指针
    slow, fast = head, head.next
    # 快慢指针需要注意的边界条件，一定是 fast and fast.next
    # 如果是一次性走三步，就是 fast and fast.next and fast.next.next
    # 确定 while 循环体里面，不会有空指针异常即可
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
    return slow
```
链表有可能是奇数个节点，也有可能是偶数个节点，对于奇数个节点的链表来说，其中间节点是确定的，对于偶数个节点的链表来说，其中间节点就不一定了。
比如：1->2->3->4，有些题要求的中间节点是2，有些题要求的中间节点是3，对于这种问题来说，其实变换一下双指针的起始节点的位置就可以了。

```python
# 还是上面那个题，我们结合哑节点的技巧，找中间节点的pre节点，然后再返回pre.next就是题目要求的结果
# 以1->2->3->4为例，上一种解法是直接找到slow=3
# 这里的解法是，找到pre=2，然后返回pre.next(3)
def middleNode(self, head: ListNode) -> ListNode:
    if not head or not head.next:
        return head
    # 关键在这里，前移了一个节点
    pre, fast = ListNode(-1, head), head
    while fast and fast.next:
        pre, fast = pre.next, fast.next.next
    # 如果题目要求的是返回1->2->3->4中的2节点，直接返回pre即可
    # return pre
    return pre.next

```

- [109. 有序链表转换二叉搜索树
](https://leetcode-cn.com/problems/convert-sorted-list-to-binary-search-tree/)
```python
def sortedListToBST(self, head: Optional[ListNode]) -> Optional[TreeNode]:
    # 这里的主要思路是一个树的递归思想，就不解释了
    # 跟链表相关的是，利用了快慢指针找链表中点，因为树的root节点是链表的中间节点
    if head is None:
        return head
    if head.next is None:
        return TreeNode(head.val)
    preMiddle = self.middlePreNode(head)
    middle = preMiddle.next
    preMiddle.next = None
    root = TreeNode(middle.val)
    root.left = self.sortedListToBST(head)
    root.right = self.sortedListToBST(middle.next)
    return root

 def middlePreNode(self, head: ListNode) -> ListNode:
    # 快慢指针找中点，slow初始值指向head的前一个节点
    # 所有最终的结果，slow也等于中间节点的前一个节点
    slow, fast = ListNode(-1, head), head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
    return slow
```
#### 1.2.2 查找链表中的第K个节点
双指针再扩展一下，两个指针之间，不一定要一个走一步一个走两步的，也不定一开始的时候，两个指针之间只相隔一个节点的。

来看下面这两个题：
- [剑指 Offer 22. 链表中倒数第k个节点](https://leetcode-cn.com/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof/)
- [面试题 02.02. 返回倒数第 k 个节点](https://leetcode-cn.com/problems/kth-node-from-end-of-list-lcci/)

```python
# 剑指 Offer 22. 链表中倒数第k个节点
def getKthFromEnd(self, head: ListNode, k: int) -> ListNode:
    if head is None:
        return head
    slow, fast = head, head
    # 先在slow和fast之间拉开k的距离
    # 然后双指针一起向链表尾部移动，知道fast为None的时候，slow就是倒数第K个节点
    for _ in range(k):
        fast = fast.next
    while fast:
        # slow, fast往后走的速度是一样的，都是一个节点
        slow, fast = slow.next, fast.next
    return slow
# 面试题 02.02. 返回倒数第 k 个节点
def kthToLast(self, head: ListNode, k: int) -> int:
    slow, fast = head, head
    for _ in range(k):
        fast = fast.next
    while fast:
        slow, fast = slow.next, fast.next
    # 跟上面那题基本一模一样，只是返回的是节点值
    return slow.val
```

- [1721. 交换链表中的节点](https://leetcode-cn.com/problems/swapping-nodes-in-a-linked-list/)
```python
# 其实就是一个双指针找正数第K个节点和倒数第K个节点的问题
def swapNodes(self, head: Optional[ListNode], k: int) -> Optional[ListNode]:
    # 去掉边界条件
    if head is None or head.next is None:
        return head
    # 只交换节点的值就很简单了，只要双指针找到两个节点即可
    slow, fast = head, head
    # 注意这里用的是 k-1，后面用的是fast.next不为空
    # 跟上面两题(面试题 02.02. 返回倒数第 k 个节点)是有区别的
    # 主要是为了找到正数第K个节点，因为正数都是从1开始而非0开始
    # 所以这里需要用k-1
    for _ in range(k - 1):
        fast = fast.next
    # 找到正数第k个节点
    p1 = fast
    while fast.next:
        slow, fast = slow.next, fast.next
    # slow就是倒数第k个节点，直接交换值即可
    p1.val, slow.val = slow.val, p1.val
    return head
```



#### 1.2.3 双指针判断链表状态(查找链表相交点、判断链表成环等)
双指针另一个比较巧妙的应用，是用于判断链表的状态：
- 链表成环：两个指针一快一慢的走，如果是链表是个环，那么快慢两个指针必然会相遇
- 判断链表是否相交：同样是两个指针，走的速度一样，但是从两条链表的两端出发，如果两条链表有交点，那么两个指针必然会相遇在交点的节点上

先来看链表成环的题：
- [141. 环形链表](https://leetcode-cn.com/problems/linked-list-cycle/)
- [面试题 02.08. 环路检测](https://leetcode-cn.com/problems/linked-list-cycle-lcci)
- [142. 环形链表 II](https://leetcode-cn.com/problems/linked-list-cycle-ii/)
- [剑指 Offer II 022. 链表中环的入口节点](https://leetcode-cn.com/problems/c32eOV/)
```python
# 141. 环形链表
def hasCycle(self, head: Optional[ListNode]) -> bool:
    # 慢指针每次走一步，快指针每次走两步
    # 如果链表有环，那么他们必定会相遇在某个节点
    # (想象一下操场跑圈，只要一直在跑，跑的快的人，必定能够超跑的慢的人一圈，也就是他们必定会在某个点相遇)
    slow, fast = head, head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow == fast:
            return True
    return False

# 142. 环形链表 II 
# 面试题 02.08. 环路检测
# 剑指 Offer II 022. 链表中环的入口节点
# 这几个题是一样的，寻找成环的那个点
def detectCycle(self, head: ListNode) -> ListNode:
    slow, fast = head, head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow == fast:
            # 假设成环的那个点为A，相遇点为B，head到A的距离为x，A到B的距离为y
            # 由于有环存在，那么B是可以到A的，距离为z
            # 那么，第一次相遇的时候，slow走过的距离为 x + y，fast走过的距离为 x + y + z + y
            # slow和fast速度不同，时间相同，所以可以得到等式 2(x+y) = x+y+z+y，则求出z=x
            # 所以再来一次双指针，从head节点和B点开始走，他们相遇的地方就是A点
            s, f = head, slow
            while s != f:
                s, f = s.next, f.next
            return s
    # 如果不成环，slow和fast不回相遇，直接返回None
    return None
```


然后看链表相交的题：
- [剑指 Offer II 023. 两个链表的第一个重合节点](https://leetcode-cn.com/problems/3u1WK4/)
- [面试题 02.07. 链表相交](https://leetcode-cn.com/problems/intersection-of-two-linked-lists-lcci/)
- [160. 相交链表](https://leetcode-cn.com/problems/intersection-of-two-linked-lists/)
- [剑指 Offer 52. 两个链表的第一个公共节点](https://leetcode-cn.com/problems/liang-ge-lian-biao-de-di-yi-ge-gong-gong-jie-dian-lcof/)
```python
# 这些题都是一样的
def getIntersectionNode(self, headA: ListNode, headB: ListNode) -> ListNode:
    # 边界条件
    if not headA or not headB:
        return None
    # 双指针
    p1, p2 = headA, headB
    # 核心逻辑是：假设链表headA长度为a，链表headB长度为b，那么
    # 当p1指针从headA跑到结尾时，就从headB继续跑
    # 当p2指针从headB跑到结尾时，就从headA继续跑
    # 如果headA和headB有交点，那么p1和p2就肯定会相遇在交点
    # 如果headA和headB没有交点，那么p1和p2就肯定相遇在None，就是刚好都跑到终点，
    #   且他们跑过的路径总长就是a+b
    while p1 != p2:
        p1 = p1.next if p1 else headB
        p2 = p2.next if p2 else headA
    return p1
```












### 1.3 链表节点的插入操作
链表节点总共就只有两个字段，一个是值字段val，一个是指向下一个节点的指针字段next。一般来说，改next字段的题比较多。

#### 1.3.1 反转链表节点

反转一整条链表的节点，其实只需要了解两个链表节点之间怎么反转就行，然后只需要循环的对链表节点两两交换即可。

```python
# 这种反转链表的方式是需要不断的移动 pre 和 cur 指针的
# 本质上就是不断的往pre节点后插入新的节点
def reverse(pre: ListNode, cur: ListNode):
    # 保留下一个节点，避免丢失
    tmp = cur.next
    # 当前节点的next指向前一个节点
    cur.next = pre
    # pre和cur的指针都向后移动，开始准备下一轮的反转链表节点
    pre, cur = cur, tmp

# 头插法反转链表，可以不用显式的去移动 pre 和 cur 指针
# 本质上就是不断的往pre节点插入插入新的节点，但也可以看作是不断的往链表头部插入节点
def reverse(pre: ListNode, cur: ListNode):
    tmp = cur.next
    cur.next = tmp.next
    tmp.next = pre.next
    pre.next = tmp
    # 这种做法的好处是，不需要显式的移动pre和cur指针了
    # 其中，cur指针回隐式的随着反转两个节点之后被移动，
    # pre指针则一直处于头部，并不会移动
```

- [206. 反转链表](https://leetcode-cn.com/problems/reverse-linked-list/)
- [剑指 Offer 24. 反转链表](https://leetcode-cn.com/problems/fan-zhuan-lian-biao-lcof/)
- [剑指 Offer II 024. 反转链表](https://leetcode-cn.com/problems/UHnkqh/)
```python
# 三个题完全一样的
# 206. 反转链表 剑指 Offer 24. 反转链表  剑指 Offer II 024. 反转链表
def reverseList(self, head: ListNode) -> ListNode:
    # 去除边界条件，链表为空或者只有单个节点时直接返回，不需要反转
    if not head or not head.next:
        return head
    # 考虑到第一个节点，反转后，next 的指向是 None，所以这里把 pre 设置成 None
    pre, cur = None, head
    while cur:
        # 当前节点的next指针要指向前一个
        tmp = cur.next
        cur.next = pre
        # pre和cur的指针都向后移动，开始准备下一轮的反转链表节点
        pre, cur = cur, tmp
    return pre
# 头插法也可以解决这几个题
def reverseList(self, head: ListNode) -> ListNode:
    if not head or not head.next:
        return head
    pre = ListNode(-1, head)
    cur = head
    while cur.next:
        tmp = cur.next
        cur.next = tmp.next
        # 这里注意赋值一定要是pre.next，因为是插入到链表的头部
        # 要注意走了几步以后，pre.next不是等于cur的
        tmp.next = pre.next
        pre.next = tmp
    return pre.next
```

- [92. 反转链表 II](https://leetcode-cn.com/problems/reverse-linked-list-ii/)
```python
# 使用头插法，相对来说比较简单
def reverseBetween(self, head: ListNode, left: int, right: int) -> ListNode:
    dummy = ListNode(-1, head)
    # 先找left位置的前一个节点
    pre = dummy
    for _ in range(left - 1):
        pre = pre.next
    # 然后开始反转
    cur = pre.next
    # 头插法反转链表
    for _ in range(right - left):
        tmp = cur.next
        cur.next = tmp.next
        tmp.next = pre.next
        pre.next = tmp
    return dummy.next
```

#### 1.3.2 交换节点

交换节点的技巧：
```python
# 非常的类似于头插法
# 需要知道至少三个节点 pre, first, second
# 首先保存下最后一个节点 post，这样我们就知道有 pre first second post 四个节点了
# 然后只需要把 first second 换个位置就行
post = second.next
# 交换位置
pre.next = second
second.next = first
first.next = post
```
- [24. 两两交换链表中的节点](https://leetcode-cn.com/problems/swap-nodes-in-pairs/)
```python
def swapPairs(self, head: ListNode) -> ListNode:
    # 先去除边界条件
    if not head or not head.next:
        return head
    dummy = ListNode(-1, head)
    # 这里是类似于双指针的思路，四指针并行跑
    pre, first, second, post = dummy, head, head.next, head.next.next
    while True:
        # 这里直接使用交换节点的技巧
        pre.next = second
        second.next = first
        first.next = post
        # 先判断是否已经到链表最后，如果是则直接跳出循环
        # 如果没到最后则继续移动指针
        if not post or not post.next:
            break
        # 移动指针
        pre = pre.next.next
        first, second = post, post.next
        post = post.next.next
    return dummy.next
```




#### 1.3.3 纯插入操作
- [剑指 Offer II 028. 展平多级双向链表](https://leetcode-cn.com/problems/Qv1Da2/)
- [430. 扁平化多级双向链表](https://leetcode-cn.com/problems/flatten-a-multilevel-doubly-linked-list/)
```python
def flatten(self, head: 'Node') -> 'Node':
    # 思路比较简单，就是从左到右遍历，发现某个节点有child，就先处理child节点
    # 把下一层的节点都扫描完然后接到当前的节点的后面
    # 处理完后，指针再往后移动一格，重复之前的操作
    cur = head
    while cur:
        if cur.child:
            # 其实这里就是一个插入操作的技巧
            # 先保存一下某些关键节点的指针
            tmp, child = cur.next, cur.child
            # 然后开始修改cur和child节点的next以及prev指针，相当于在cur后面插入child节点
            # 同时把cur.child置为None，因为这个节点的child已经处理完了
            cur.next, child.prev, cur.child = child, cur, None

             if tmp:
                # 如果原来的cur.next是存在的话，要把下一层的链表的最后的节点和这个节点连接上
                while child.next:
                    child = child.next
                tmp.prev, child.next = child, tmp
        # 处理完以后，指针向后走一格
        cur = cur.next
    return head
```## 二、多种技巧结合的链表题

### 2.1 双指针与删除链表节点结合
- [剑指 Offer II 021. 删除链表的倒数第 n 个结点](https://leetcode-cn.com/problems/SLwz0R/)
- [19. 删除链表的倒数第 N 个结点](https://leetcode-cn.com/problems/remove-nth-node-from-end-of-list/)
```python
def removeNthFromEnd(self, head: ListNode, n: int) -> ListNode:
    if not head:
        return head
    dummy = ListNode(-1, head)
    # 双指针，找到倒数第N个节点的pre节点
    slow, fast = dummy, head
    # 先让fast走n步
    for _ in range(n):
        fast = fast.next
    # 然后一直走到底，这时候slow就是倒数第N个节点的pre节点
    while fast:
        slow, fast = slow.next, fast.next
    # 知道当前节点的前驱pre节点，删除当前节点的技巧
    slow.next = slow.next.next
    return dummy.next
```

- [2095. 删除链表的中间节点](https://leetcode-cn.com/problems/delete-the-middle-node-of-a-linked-list/)
```python
def deleteMiddle(self, head: Optional[ListNode]) -> Optional[ListNode]:
    # 双指针找中间节点的前一个节点
    dummy = ListNode(-1, head)
    # 这边稍微变通一下，本来找中间节点时，是slow,fast=head,head
    # 这边把slow设置为head的前一个节点，所以最后就能找到中间节点的前一个节点
    slow, fast = dummy, head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
    # 知道pre节点，删除节点的技巧
    slow.next = slow.next.next
    return dummy.next
```

### 2.2 双指针与反转链表技巧相结合
- [面试题 02.06. 回文链表](https://leetcode-cn.com/problems/palindrome-linked-list-lcci/)
- [剑指 Offer II 027. 回文链表](https://leetcode-cn.com/problems/aMhZSa/)
- [234. 回文链表](https://leetcode-cn.com/problems/palindrome-linked-list/)
```python
def isPalindrome(self, head: ListNode) -> bool:
    if not head or not head.next:
        return True
    # 先用双指针找中点
    slow, fast = head, head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
    # 再反转后半部分链表
    pre = None
    # 链表节点数是奇数时，fast肯定不是None，且slz进一个，如果是偶数，则不需要
    # 1 -> 2 -> 3 -> 2 -> 1，此时中点slow是3，需要从下一个2开始反转
    # 1 -> 2 -> 2 -> 1，此时中点slow是2，直接开始反转即可
    if fast:
        slow = slow.next
    while slow:
        tmp = slow.next
        slow.next = pre
        pre = slow
        slow = tmp
    # 最后循环对比
    while pre:
        if pre.val != head.val:
            return False
        pre, head = pre.next, head.next
    return True
``` 

- [2130. 链表最大孪生和](https://leetcode.cn/problems/maximum-twin-sum-of-a-linked-list/)
```python
def pairSum(self, head: Optional[ListNode]) -> int:
    # 思路比较简单，就是先反转后半部分的链表，然后从两头开始遍历，比较相加的值即可
    # 技巧是双指针+反转链表
    if head is None:
        return head
    # 先双指针找中间节点，注意这里对起始位置的设置
    # 可以做到循环结束以后，slow是中间节点的前一个节点
    # 比如 1->2->3->4，循环结束时slow指向2
    slow, fast = head, head.next
    while fast.next:
        slow, fast = slow.next, fast.next.next
    
    # 从中间断开链表，然后反转链表，最后pre是后半段链表的头指针
    pre, cur = None, slow.next
    slow.next = None
    while cur:
        tmp = cur.next
        cur.next = pre
        pre = cur
        cur = tmp
    
    # 然后开始对比即可
    ans = 0
    p1, p2 = head, pre
    while p1:
        ans = max(ans, p1.val + p2.val)
        p1, p2 = p1.next, p2.next
    return ans
```

- [25. K 个一组翻转链表](https://leetcode.cn/problems/reverse-nodes-in-k-group/)
```python
def reverseKGroup(self, head: Optional[ListNode], k: int) -> Optional[ListNode]:
    if not head or not head.next:
        return head
    pre = dummy = ListNode(-1, head)
    while head:
        cur = pre
        # 先检查是否大于等于K
        for _ in range(k):
            cur = cur.next
            if not cur:
                return dummy.next
        # 此时cur是需要反转的链表的tail节点
        # pre.next是需要反转的链表的head节点
        tmp = cur.next
        h, t = self.reverse(pre.next, cur)
        # 把链表接上
        pre.next, t.next = h, tmp
        # 重新调整指针位置
        pre, head = t, t.next
    return dummy.next

def reverse(self, head: ListNode, tail: ListNode):
    pre, cur = None, head
    while pre != tail:
        tmp = cur.next
        cur.next = pre
        pre = cur
        cur = tmp
    return tail, head
```

- [143. 重排链表](https://leetcode.cn/problems/reorder-list/)
- [剑指 Offer II 026. 重排链表](https://leetcode.cn/problems/LGjMqU/)
```python
def reorderList(self, head: ListNode) -> None:
    """
    Do not return anything, modify head in-place instead.
    """
    # 边界条件
    if not head or not head.next:
        return
    # 基本思路是，双指针找中点，然后反转后半部分链表，最后合并链表
    slow, fast = ListNode(-1, head), head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
    # 此时，slow是中点的前一个节点
    tmp = slow.next
    slow.next = None

    p1, p2 = head, self.reverse(tmp)

    self.merge(p1, p2)

 def merge(self, p1: ListNode, p2: ListNode) -> ListNode:
    head = dummy = ListNode(-1)
    # 这里因为题目能确定，前半部分链表长度肯定小于等于后半部分
    # 所以有一些边界条件就不考虑了
    while p1:
        head.next = p1
        p1 = p1.next
        head = head.next

         head.next = p2
        p2 = p2.next
        head = head.next
    if p2:
        head.next = p2
    return dummy.next


  def reverse(self, head: ListNode) -> ListNode:
    if not head or not head.next:
        return head
    pre, cur = None, head
    while cur:
        tmp = cur.next
        cur.next = pre
        pre = cur
        cur = tmp
    return pre
```


### 2.3 双指针与插入链表技巧结合
- [86. 分隔链表](https://leetcode-cn.com/problems/partition-list/)
- [面试题 02.04. 分割链表](https://leetcode-cn.com/problems/partition-list-lcci/)
```python
def partition(self, head: ListNode, x: int) -> ListNode:
    # 边界条件
    if not head or not head.next:
        return head
    # 类似于双指针技巧吧，其实是搞了两个链表，
    # small表示小于x的节点，按照原链表顺序排列
    # large表示大于等于x的节点，按照原链表顺序排列
    # 最后把large接到samll的尾节点之后即可
    small = smallHead = ListNode(0)
    large = largeHead = ListNode(0)
    while head:
        # 插入链表的技巧
        if head.val < x:
            small.next = head
            small = small.next
        else:
            large.next = head
            large = large.next
        head = head.next
    # 这里注意一下把最后那个节点设置为next=None，插入链表时常需要注意的点
    # 因为这个节点可能是原链表中的某一个中间节点
    large.next = None
    # 最后把large接到samll的尾节点之后
    small.next = largeHead.next
    return smallHead.next
```

- [138. 复制带随机指针的链表](https://leetcode.cn/problems/copy-list-with-random-pointer/)
- [剑指 Offer 35. 复杂链表的复制](https://leetcode.cn/problems/fu-za-lian-biao-de-fu-zhi-lcof/)
```python
def copyRandomList(self, head: 'Node') -> 'Node':
    # 边界条件
    if head is None:
        return None
    # 先遍历一边，把每个节点都copy一下并插入到当前节点后面
    cur = head
    while cur:
        tmp = Node(cur.val)
        tmp.next = cur.next
        cur.next = tmp
        cur = tmp.next
    # 然后把random拷贝一下
    cur = head
    while cur:
        if cur.random:
            cur.next.random = cur.random.next
        cur = cur.next.next
    # 最后开始拆链表，把之前copy出来的节点从链表里拆出来
    slow, fast = head, head.next
    dummy = fast
    while fast and fast.next:
        slow.next, fast.next = slow.next.next, fast.next.next
        slow, fast = slow.next, fast.next
    slow.next = None
    return dummy
```

- [147. 对链表进行插入排序](https://leetcode.cn/problems/insertion-sort-list/)
```python
def insertionSortList(self, head: ListNode) -> ListNode:
    # 边界条件
    if not head or not head.next:
        return head
    dummy = ListNode(float("-inf"), head)
    # 就是一个双指针模拟一下就可以
    slow, fast = dummy, dummy.next
    while fast:
        # 如果已经按照顺序排列好了，就都往后走一步
        if fast.val >= slow.val:
            slow, fast = slow.next, fast.next
        else:
            # 如果需要把fast节点插入到前面的节点
            # 就从头开始遍历，找到可插入的位置
            pre = dummy
            while pre.next.val < fast.val:
                pre = pre.next
            # 需要处理两个节点，一个是把fast从原来的地方删掉，所以用删除节点的技巧
            slow.next = slow.next.next
            # 这里要把fast节点插入到对应的位置，用插入节点的技巧
            tmp = pre.next
            pre.next = fast
            fast.next = tmp
        # 最后为了能继续循环后面的，需要把fast节点重新指向slow节点的后续
        fast = slow.next
    return dummy.next

``

- [328. 奇偶链表](https://leetcode.cn/problems/odd-even-linked-list/)
```python
def oddEvenList(self, head: Optional[ListNode]) -> Optional[ListNode]:
    # 边界条件
    if not head or not head.next:
        return head
    # 思路比较简单，双指针遍历一遍，把奇数节点和偶数节点分成两个链表
    # 然后拼接上去就行
    slow, fast = head, head.next
    tmp = head.next

     while fast and fast.next:
        slow.next, fast.next = slow.next.next, fast.next.next
        slow, fast = slow.next, fast.next
    slow.next = tmp
    return head
```
### 2.4 双指针与排序算法相结合
- [148. 排序链表](https://leetcode.cn/problems/sort-list/)
- [剑指 Offer II 077. 链表排序](https://leetcode-cn.com/problems/7WHec2/)
```python
def sortList(self, head: Optional[ListNode]) -> Optional[ListNode]:
    # 使用递归实现归并排序
    # 思路就是，先用双指针找中点，从中间断开链表，合并左右两条链表，不断的递归即可
    # 时间复杂度是O(N*logN)空间复杂度是O(logN)
    if not head or not head.next:
        # 边界条件
        return head
    # 双指针找中点
    slow, fast = ListNode(-1, head), head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
    # slow是中点的前一个节点，然后从中间截断链表
    mid, slow.next = slow.next, None
    # 递归的对两遍的链表排序
    left, right = self.sortList(head), self.sortList(mid)
    # 合并两条链表
    cur = dummy = ListNode(-1)
    while left and right:
        if left.val <= right.val:
            cur.next = left
            left = left.next
        else:
            cur.next = right
            right = right.next
        cur = cur.next
    cur.next = left if left else right
    return dummy.next

# 但是这个题，还有个更好的解法，可以把空间复杂度优化到O(1)


```

- [剑指 Offer II 078. 合并排序链表](https://leetcode-cn.com/problems/vvXgSW/)
- []()
```python
def mergeKLists(self, lists: List[ListNode]) -> ListNode:
    # 边界条件
    if not lists:
        return None
    length = len(lists)
    if length == 0:
        return None
    # 其实是对数组做了一个归并的操作
    return self.mergeAndSort(lists, 0, length - 1)

def mergeAndSort(self, lists: List[ListNode], left: int, right: int) -> ListNode:
    if left == right:
        return lists[left]
    middle = (left + right) // 2
    l = self.mergeAndSort(lists, left, middle)
    r = self.mergeAndSort(lists, middle + 1, right)
    return self.merge(l, r)

def merge(self, p1: ListNode, p2: ListNode) -> ListNode:
    if not p1:
        return p2
    if not p2:
        return p1
    pre = dummy = ListNode(-1)
    while p1 and p2:
        if p1.val <= p2.val:
            pre.next, p1 = p1, p1.next
        else:
            pre.next, p2 = p2, p2.next
        pre = pre.next
    pre.next = p1 if p1 else p2
    return dummy.next
```

### 2.5 链表和其他数据结构结合
这种就是利用链表数据结构的特性来做一些事情，链表数据结构有如下的特性：
- 插入、删除操作O(1)
- 查询操作O(n)
- 有顺序

#### 2.5.1 双向链表跟哈希表结合
哈希表的数据结构特性：
- 插入、删除、查询操作O(1)
- 没有顺序

哈希表跟链表数据结构结合起来使用，可以取长补短，做到：
- 插入、删除、查询操作O(1)
- 有顺序

- [剑指 Offer II 031. 最近最少使用缓存](https://leetcode-cn.com/problems/OrIXps/)
- [面试题 16.25. LRU 缓存](https://leetcode-cn.com/problems/lru-cache-lcci/)
- [146. LRU 缓存](https://leetcode-cn.com/problems/lru-cache/)
```python
# 主要的思路是：
# 1. 利用哈希表的查找操作为O(1)，可以直接找到key对应的链表节点
# 2. 利用双向链表的插入、删除节点操作为O(1)，可以很方便的新增、删除数据
# 3. 利用双向链表保持顺序，头部的节点是新的，尾部的节点是旧的
class Node:
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:

    def __init__(self, capacity: int):
        self.cache = dict()
        self.head = Node(-1)
        self.tail = Node(-1)
        # 这里用哑节点，这样后面插入/删除链表节点操作就不用考虑刚好
        # 要操作的节点在两端了
        self.head.next, self.tail.prev = self.tail, self.head
        self.capacity = capacity
        self.size = 0

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        # 如果key存在，则需要把这个节点移动到头部
        node = self.cache[key]
        self.moveToHead(node)
        return node.value

    def put(self, key: int, value: int) -> None:
        
        if key in self.cache:
            node = self.cache[key]
            # 如果存在，就修改value并把节点移动到头部
            node.value = value
            self.moveToHead(node)
        else:
            # 如果不存在，则在头部插入一个新节点
            node = Node(key, value)
            self.cache[key] = node
            self.addToHead(node)
            # 检查size
            if self.size >= self.capacity:
                # 超容量了先删除尾部节点
                removed = self.removeTail()
                self.cache.pop(removed.key)
            else:
                self.size += 1

    def moveToHead(self, node: Node):
        self.removeNode(node)
        self.addToHead(node)
    
    def removeNode(self, node: Node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def addToHead(self, node: Node):
        self.head.next.prev = node
        node.next = self.head.next
        self.head.next = node
        node.prev = self.head
                
    def removeTail(self) -> Node:
        node = self.tail.prev
        self.removeNode(node)
        return node
```




## 三、剩下的一些其实跟链表关系不大的题
剩下的还有一部分虽然打了链表的tag，但是实际上跟链表关系不是特别大的题目。这些题里可能只是使用了链表这个数据结构而已，并不涉及到以上提到的一些链表特有的答题技巧。

### 3.1 一些简单的只需要遍历链表的题
- [剑指 Offer 25. 合并两个排序的链表](https://leetcode-cn.com/problems/he-bing-liang-ge-pai-xu-de-lian-biao-lcof/)
- [21. 合并两个有序链表](https://leetcode-cn.com/problems/merge-two-sorted-lists/)
```python
# 这两题也是一个简单的遍历，一遍遍历一遍选择节点就行
def mergeTwoLists(self, l1: ListNode, l2: ListNode) -> ListNode:
    cur = dum = ListNode(0)
    while l1 and l2:
        if l1.val < l2.val:
            cur.next, l1 = l1, l1.next
        else:
            cur.next, l2 = l2, l2.next
        cur = cur.next
    # 需要注意的是这里，l1 或者 l2 其中某一个遍历完了以后，
    # 剩下的那个链表的所有节点，还需要接到结果的链表上去
    cur.next = l1 if l1 else l2
    return dum.next
```

- [1290. 二进制链表转整数](https://leetcode-cn.com/problems/convert-binary-number-in-a-linked-list-to-integer/)
```python
# 这个就是简单的遍历，没有什么特别的技巧
def getDecimalValue(self, head: ListNode) -> int:
    cur, ans = head, 0
    while cur:
        ans = ans * 2 + cur.val
        cur = cur.next
    return ans
```

- [剑指 Offer 06. 从尾到头打印链表](https://leetcode-cn.com/problems/cong-wei-dao-tou-da-yin-lian-biao-lcof/)
```python
# 遍历一遍就行，实在不行可以用反转链表
def reversePrint(self, head: ListNode) -> List[int]:
    res = []
    while head:
        res.append(head.val)
        head = head.next
    # 这里其实用了python的语法糖
    return res[::-1]
```

- [2. 两数相加](https://leetcode-cn.com/problems/add-two-numbers/)
- [面试题 02.05. 链表求和](https://leetcode-cn.com/problems/sum-lists-lcci/)
```python
# 也就是一个简单的遍历
def addTwoNumbers(self, l1: ListNode, l2: ListNode) -> ListNode:
    pre = dummy = ListNode(-1)
    carry = 0
    while l1 or l2:
        val = carry
        if l1:
            val += l1.val
            l1 = l1.next
        if l2:
            val += l2.val
            l2 = l2.next
        pre.next = ListNode(val % 10)
        carry = val // 10
        pre = pre.next
    # 需要注意的是这边，最后如果还有进位的话，要多加一个节点
    if carry != 0:
        pre.next = ListNode(carry)
    return dummy.next
```

- [445. 两数相加 II](https://leetcode-cn.com/problems/add-two-numbers-ii/)
- [剑指 Offer II 025. 链表中的两数相加](https://leetcode-cn.com/problems/lMSNwu/)
```python
# 445. 两数相加 II
# 剑指 Offer II 025. 链表中的两数相加
# 题目要求不能反转链表，就只能用栈解决了，空间复杂度O(m+n)
# 反而是使用反转链表的解法可以把空间压缩到 O(1)
def addTwoNumbers(self, l1: ListNode, l2: ListNode) -> ListNode:
    s1, s2 = [], []
    while l1:
        s1.append(l1.val)
        l1 = l1.next
    while l2:
        s2.append(l2.val)
        l2 = l2.next
    ans = None
    carry = 0
    while s1 or s2 or carry != 0:
        a = 0 if not s1 else s1.pop()
        b = 0 if not s2 else s2.pop()
        cur = a + b + carry
        carry = cur // 10
        cur %= 10
        curnode = ListNode(cur)
        curnode.next = ans
        ans = curnode
    return ans
```

- [1669. 合并两个链表](https://leetcode-cn.com/problems/merge-in-between-linked-lists/)
```python
# 这个就是简单遍历，然后把链表接上就行
def mergeInBetween(self, list1: ListNode, a: int, b: int, list2: ListNode) -> ListNode:
    preA = pB = dummy = ListNode(-1, list1)
    # 先找到a位置节点的pre节点(preA)
    for _ in range(a):
        preA = preA.next
    # 再找b位置的节点(pB)
    for _ in range(b+1):
        pB = pB.next
    # 然后把list2接到a,b之间
    preA.next = list2
    while preA.next:
        preA = preA.next
    preA.next = pB.next
    return dummy.next
```

- [61. 旋转链表](https://leetcode-cn.com/problems/rotate-list/)
```python
def rotateRight(self, head: Optional[ListNode], k: int) -> Optional[ListNode]:
    # 思路：现将头尾连起来，然后找合适的地方断开即可
    # 这题的难点其实是个数学计算，要把断开的点算对，画个图就会比较清晰
    if head is None or head.next is None or k == 0:
        return head
    # 先找最后一个节点
    tail, count = head, 1
    while tail.next:
        count += 1
        tail = tail.next
    # 可以提早结束
    step = k % count
    if step == 0:
        return head
    # 头尾相连
    tail.next = head
    for _ in range(count - step):
        tail = tail.next
    # 找到合适的点，然后断开
    res = tail.next
    tail.next = None
    return res
```

- [1472. 设计浏览器历史记录](https://leetcode-cn.com/problems/design-browser-history/)
```python
# 这题就是一个简单的双向链表模拟一下
class Node:
    def __init__(self, val: str):
        self.val = val
        self.pre = None
        self.next = None

class BrowserHistory:

    def __init__(self, homepage: str):
        self.cur = Node(homepage)

    def visit(self, url: str) -> None:
        node = Node(url)
        node.pre = self.cur
        self.cur.next = node
        self.cur = self.cur.next

    def back(self, steps: int) -> str:
        while steps > 0 and self.cur.pre:
            self.cur = self.cur.pre
            steps -= 1
        return self.cur.val

    def forward(self, steps: int) -> str:
        while steps > 0 and self.cur.next:
            self.cur = self.cur.next
            steps -= 1
        return self.cur.val
```

- [2181. 合并零之间的节点](https://leetcode.cn/problems/merge-nodes-in-between-zeros/)
```python
def mergeNodes(self, head: Optional[ListNode]) -> Optional[ListNode]:
    # 排除边界条件
    if head is None:
        return None
    # 这题的思路其实很简单，从左到右一遍循环，
    # 不断的累加值，只要遇到0就停下来，新建节点，重置累加的值，
    # 然后继续循环即可
    dummy = pre = ListNode(-1)
    cur = head.next
    acc = 0

     while cur:
        if cur.val == 0:
            node = ListNode(acc)
            pre.next = node
            pre = pre.next
            acc = 0
        else:
            acc += cur.val
        cur = cur.next
    return dummy.next
```

- [725. 分隔链表](https://leetcode-cn.com/problems/split-linked-list-in-parts/)
```python
def splitListToParts(self, head: ListNode, k: int) -> List[ListNode]:
    # 思路比较简单，就是先计算平均长度，比如链表总长为5， k=3时
    # 平均长度5//3=1，如果三个链表，每个链表都分配1个节点的话，就剩下5%3=2个节点未分配
    # 这时候就把剩下的2个节点都分配给前面的链表，所以答案就是[2, 2, 1]
    res, length = [], self.calLength(head)
    quotient, remainder = length // k, length % k

     res = [None for _ in range(k)]
    i, cur = 0, head
    while i < k and cur:
        res[i] = cur
        size = quotient + (1 if i < remainder else 0)
        for _ in range(size - 1):
            cur = cur.next
        tmp = cur.next
        cur.next = None
        cur = tmp
        i += 1
    return res

 def calLength(self, head: ListNode) -> int:
    cur, length = head, 0
    while cur:
        cur, length = cur.next, length + 1
    return length
```

- [2058. 找出临界点之间的最小和最大距离](https://leetcode-cn.com/problems/find-the-minimum-and-maximum-number-of-nodes-between-critical-points/)
```python
def nodesBetweenCriticalPoints(self, head: Optional[ListNode]) -> List[int]:
    # 思路比较简单，最小距离就是两个相邻的极值点之间的距离，最大距离就是第一个和最后一个极值点的距离
    # 0. 用 first 和 last 保存第一个和最后一个极值点的位置
    # 1. 先从左往右遍历，cur,cur.next, cur.next.next 三个点之间判断cur.next是不是极值点
    # 2. 如果不是极值点，更新cur指针，直接往后遍历即可
    # 3. 如果是极值点：
    #   3.1 当first没有被初始化时，需要初始化first
    #   3.2 last没有被初始化时，初始化last
    #   3.3 当last已经被初始化时，说明可以开始更新minDist和maxDist的值了，最后也要更新last的值
    minDist = maxDist = -1
    first = last = -1
    cur, pos = head, 0

     while cur and cur.next and cur.next.next:
        a, b, c = cur.val, cur.next.val, cur.next.next.val
        if b > max(a, c) or b < min(a, c):
            if last != -1:
                maxDist = max(pos - first, maxDist)
                minDist = (pos - last if minDist == -1 else min(minDist, pos - last))
            if first == -1:
                first = pos
            # last的值在每次发现新的极值时都需要更新
            last = pos
        cur, pos = cur.next, pos + 1

     return [minDist, maxDist]
```

- [剑指 Offer II 029. 排序的循环链表](https://leetcode-cn.com/problems/4ueAj6/)
```python
# 这题其实跟链表关系不大，只要遍历一遍找到能插入的点就行
# 思路比较简单，分为三种情况：
# 1. 如果 head 是 None，那就创建一个节点，让这个节点的next指针指向自己即可
# 2. 开始循环，寻找边界，边界会出现p.next.val < p.val的情况，这时候p.next就是最小值节点，p就是最大值节点
#   2.1 如果要插入的值是大于最大值，小于最小值，就是要在边界点插入 
#   2.2 如果要插入的值是大于等于p.val小于等于p.next.val，就说明找到了插入的点
def insert(self, head: 'Node', insertVal: int) -> 'Node':
    if not head:
        tmp = Node(insertVal)
        tmp.next = tmp
        return tmp
    
    p = head
    while p.next != head:
        if p.val > p.next.val:
            # 说明到达了边界点，p.next就是这个链表的起点
            if p.val < insertVal or p.next.val > insertVal:
                break
        if p.val <= insertVal and insertVal <= p.next.val:
            break
        p = p.next
    p.next = Node(insertVal, p.next)
    return head
```

- [817. 链表组件](https://leetcode-cn.com/problems/linked-list-components/)
```python
def numComponents(self, head: Optional[ListNode], nums: List[int]) -> int:
    # 用一个set保存nums的值，然后遍历的时候检查下即可，需要注意ans加一的条件：
    # 1. 当前节点的值在numSet里，而下一个节点的值不在numSet里
    # 2. 当前节点的值在numSet里，而下一个节点为None
    numSet, cur, ans = set(nums), head, 0
    while cur:
        if cur.val in numSet:
            if cur.next:
                if cur.next.val not in numSet:
                    ans += 1
            else:
                ans += 1
        cur = cur.next
    return ans
```
### 3.2 通用刷题技巧：前缀和

- [1171. 从链表中删去总和值为零的连续节点
](https://leetcode-cn.com/problems/remove-zero-sum-consecutive-nodes-from-linked-list/)
```python
# 核心思路就是，如果两个节点的前缀和相等，则意味着两个节点之间的节点和为0
# 前缀和也是一种常见的刷题技巧，可以用在任意的线性数据结构之上，比如数组
def removeZeroSumSublists(self, head: ListNode) -> ListNode:
    lookup = dict()
    cur = dummy = ListNode(0, head)
    preSum = 0
    # 先建立前缀和跟节点的映射，注意后面的节点如果前缀和跟前面的节点一样
    # 在map里就会覆盖掉前一个节点
    # 然后下一次遍历的时候就能找到前后两个前缀和一样的节点，只要删除这两个节点中间的全部节点即可
    while cur:
        preSum += cur.val
        lookup[preSum] = cur
        cur = cur.next
    # 开始一轮新的遍历
    cur, preSum = dummy, 0
    while cur:
        preSum += cur.val
        # lookup[preSum] 这时候要么找到cur节点自己，要么就是找到跟他一样前缀和的那个节点
        cur.next = lookup[preSum].next
        cur = cur.next
    return dummy.next
    
```


### 3.3 随机算法

- [382. 链表随机节点](https://leetcode-cn.com/problems/linked-list-random-node/)
```python
class Solution:
    def __init__(self, head: Optional[ListNode]):
        self.head = head

    def getRandom(self) -> int:
        # 蓄水池抽样算法可以以解决一个经典的面试算法题：当内存无法加载全部数据时，
        # 如何从包含未知大小的数据流中随机选取k个数据，
        # 并且要保证每个数据被抽取到的概率相等
        # 其实跟这个题就是一样的
        cur, i, ans = self.head, i, None
        while cur:
            # 每一轮都有 i/1 机会被选中
            if randrange(i) == 0:
                ans = cur.val
            i = i + 1
            cur = cur.next
        return ans
```

其实在[设计跳表](https://leetcode-cn.com/problems/design-skiplist/)的时候，也有一个随机算法，用于选择层数：
```python
# 理论来讲，一级索引中元素个数应该占原始数据的 50%，二级索引中元素个数占 25%，三级索引12.5% ，一直到最顶层
# 因为这里每一层的晋升概率是 50%。对于每一个新插入的节点，都需要调用 randomLevel 生成一个合理的层数。
# 该 randomLevel 方法会随机生成 1~MAX_LEVEL 之间的数，且 ：
#        50%的概率返回 1
#        25%的概率返回 2
#      12.5%的概率返回 3 ...
def randomLevel():
    # p 表示每一层被选到的概率
    level, p, maxLevel = 1, 0.5, 16
    # pytyon的random.random()用于从[0, 1)生成一个随机的浮点数
    while random.random() < 0.5 and level < 16:
        level += 1
    return level

```


### 3.4 通用刷题技巧：单调栈

[单调栈](https://www.cnblogs.com/mk-oi/p/13541195.html) 是一种比较通用的刷题技巧，它是有模版的：
```python
def template(self, nums: List[int]) -> List[int]:
    stack, res = [], [0] * len(nums)
    # 反向遍历数组
    for i in range(len(nums)-1, -1, -1):
        # 这一步是把所有的比当前元素小的数都弹出栈
        while len(stack) > 0 and stack[-1] <= nums[i]:
            stack.pop()
        # 更新结果，要么是栈顶元素，要么找不到就赋值0
        res[i] = stack[-1] if len(stack) > 0 else 0
        # 把当前的元素入栈
        stack.append(nums[i])
    return res
```

- [1019. 链表中的下一个更大节点](https://leetcode-cn.com/problems/next-greater-node-in-linked-list/)
```python
# 这个题比较简单的解法是，直接转成数组，然后套用单调栈模版
def nextLargerNodes(self, head: Optional[ListNode]) -> List[int]:
    # 先转成数组
    array = []
    while head:
        array.append(head.val)
        head = head.next
    # 套用单调栈模版
    length = len(array)
    stack, ans = [], [0] * length
    # 从后往前逆序遍历
    for i in range(length - 1, -1, -1):
        while len(stack) > 0 and stack[-1] <= array[i]:
            stack.pop()
        ans[i] = stack[-1] if len(stack) > 0 else 0
        stack.append(array[i])
    return ans
```
