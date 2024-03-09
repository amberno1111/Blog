本文主要是对LeetCode上Binary Tree这个Tag的相关题解总结。

# 一、二叉树的遍历方式
在解题的时候，最常见的操作就是需要遍历某种数据结构里存储的所有元素，二叉树也不例外。二叉树有以下几种遍历方式：
- 深度优先遍历：包括了前序遍历、中序遍历、后序遍历
- 莫里斯遍历：只是深度优先遍历的非递归实现，这里单独归类是因为这种方式比较巧妙，性能也很好
- 宽度优先遍历

## 1.1 深度优先遍历
### 1.1.1 递归
递归形式的二叉树深度优先遍历有一个固定的框架，如下：

```python
def preorder(root: Optional[TreeNode]) -> List[int]:  
    if root is None:  
        return []  
    # 前序优先遍历  
    return [root.val] + preorder(root.left) + preorder(root.right)  
  
  
def inorder(root: Optional[TreeNode]) -> List[int]:  
    if root is None:  
        return []  
    # 中序优先遍历  
    return inorder(root.left) + [root.val] + inorder(root.right)  
  
  
def postorder(root: Optional[TreeNode]) -> List[int]:  
    if root is None:  
        return []  
    # 后序优先遍历  
    return postorder(root.left) + postorder(root.right) + [root.val]
```

从这个框架来看，其实这三种遍历方式基本上都一样，唯一的区别在于什么时候处理当前这个节点的值：
- 先处理当前节点，再处理左子树，最后处理右子树，就是前序遍历。[LeetCode原题：144. 二叉树的前序遍历](https://leetcode.cn/problems/binary-tree-preorder-traversal/)
- 先处理左子树，再处理当前节点，最后处理右子树，就是中序遍历[LeetCode原题：94. 二叉树的中序遍历](https://leetcode.cn/problems/binary-tree-inorder-traversal/)
- 先处理左子树，再处理右子树，最后处理当前节点，就是后序遍历[LeetCode原题：145. 二叉树的后序遍历](https://leetcode.cn/problems/binary-tree-postorder-traversal/)

所有的二叉树的题，都是在这几种遍历方式的基础上，再增加一些额外的操作而已。

#### 1.1.1.1 直接遍历的变种题
有一些题，实际上直接用遍历就可以解决，但题目又不是直接让你写前/中/后序遍历，其实是直接遍历的变种题，就是写前/中/后序遍历，然后对root节点做一些额外的操作(不再是简单的直接把val加到结果列表)而已。

[100. 相同的树](https://leetcode.cn/problems/same-tree/)
```python
def is_same_tree(p: TreeNode, q: TreeNode) -> bool:  
    # 这是一些边界条件，可以提前做掉的  
    if p is None and q is None:  
        return True  
    if not p or not q:  
        return False  
    # 比较两个树是否一样，实际上就是都用前序遍历，然后比较相等即可  
    if p.val != q.val:  
        # 前序遍历先比较当前节点的值，不相等就直接返回了  
        return False  
    # 前序遍历，再比较左子树，可以直接递归了  
    if not is_same_tree(p.left, q.left):  
        return False  
    # 前序遍历，最后比较左子树，可以直接递归了  
    if not is_same_tree(p.right, q.right):  
        return False  
    return True


# 上面是为了表示思路，写的比较长，可以优化一下，如下  
def is_same_tree(p: TreeNode, q: TreeNode) -> bool:  
    if p is None and q is None:  
        return True  
    if not p or not q:  
        return False  
    return p.val == q.val and is_same_tree(p.left, q.left) and is_same_tree(p.right, q.right)
```

[226. 翻转二叉树](https://leetcode.cn/problems/invert-binary-tree/)
```python
def invert_tree(root: Optional[TreeNode]) -> Optional[TreeNode]:  
    if root is None:  
        return root  
    if root.left is None and root.right is None:  
        return root  
    # 前序遍历，先处理当前节点，如果要翻转，实际上对于当前节点来说，我只需要翻转左右子树即可  
    root.left, root.right = root.right, root.left  
    # 前序遍历，处理左右节点  
    invert_tree(root.left)  
    invert_tree(root.right)  
    return root
```

[617. 合并二叉树](https://leetcode.cn/problems/merge-two-binary-trees/)
```python
def merge_trees(root1: Optional[TreeNode], root2: Optional[TreeNode]) -> Optional[TreeNode]:  
    # 合并两个树，实际上只需要同时对两棵树来一遍前序遍历+处理当前节点  
    # 前序遍历处理当前节点，有一个为None则直接挂另一棵树的节点，都为None时挂的也是None  
    if root1 is None:  
        return root2  
    if root2 is None:  
        return root1  
    # 前序遍历处理当前节点，都不为None，则叠加值  
    merged = TreeNode()  
    merged.val = root1.val + root2.val  
    # 前序遍历依次处理左右子树  
    merged.left = merge_trees(root1.left, root2.left)  
    merged.right = merge_trees(root1.right, root2.right)  
    return merged
```

[872. 叶子相似的树](https://leetcode.cn/problems/leaf-similar-trees/)
```python
def leaf_similar(self, root1: Optional[TreeNode], root2: Optional[TreeNode]) -> bool:  
    # 只需要先用先序遍历把所有的叶子值都取出来，然后对比下结果就行  
    def preorder(root: TreeNode) -> List[int]:  
        res = []  
        if root is None:  
            return res  
        # 先序遍历，处理当前节点时，只有是叶子类型的节点，才会把值放到结果列表里  
        if root.left is None and root.right is None:  
            res.append(root.val)  
            return res  
        # 先序遍历，处理左子树和右子树  
        res += preorder(root.left)  
        res += preorder(root.right)  
        return res  
    root1_leaves = preorder(root1)  
    root2_leaves = preorder(root2)  
    return root1_leaves == root2_leaves
```

[1325. 删除给定值的叶子节点](https://leetcode.cn/problems/delete-leaves-with-a-given-value/)
```python
def remove_leaf_nodes(root: Optional[TreeNode], target: int) -> Optional[TreeNode]:  
    # 题目要求是，先要知道左子节点和右子节点的状态，才能知道怎么处理当前节点  
    # 所以这是个后序遍历的题  
    if root is None:  
        return root  
    # 后序遍历先处理左右子树  
    root.left = remove_leaf_nodes(root.left, target)  
    root.right = remove_leaf_nodes(root.right, target)  
    # 后序遍历处理当前节点，因为左右子树已经处理完了  
    # 所以这里只考虑当前节点是否满足被删除的条件即可  
    if root.left is None and root.right is None and root.val == target:  
        return None  
    return root
```

[112. 路径总和](https://leetcode.cn/problems/path-sum/)
```python
def has_path_sum(root: Optional[TreeNode], target: int) -> bool:
    # 先序遍历，先处理当前节点
    # 满足题目的要求的操作是，当前节点是叶子节点时，从root节点到当前节点的值的和加起来等于target
    # 这里使用了一个取巧的做法，每次都用target减去当前节点的值在递归传入，这样最后到叶子节点只需要判断是否值等于传入的target即可
    if root is None:
        return False
    if root.left is None and root.right is None and root.val == target:
        return True
    # 非叶子节点就直接开始处理左子树和右子树
    return has_path_sum(root.left, target - root.val) or has_path_sum(root.right, target - root.val)
```

[113. 路径总和 II](https://leetcode.cn/problems/path-sum-ii/)
```python
def path_sum(root: Optional[TreeNode], target_sum: int) -> List[List[int]]:  
    # 跟 129. 求根节点到叶节点数字之和 那题基本上一样  
    # 前序遍历  
    def dfs(node: Optional[TreeNode], t: int, path: List[int], res: List[List[int]]):  
        if node is None:  
            return  
        if node.left is None and node.right is None and t == node.val:  
            # 满足要求的路径，保存下来  
            res.append(path[:] + [node.val])  
        dfs(node.left, t - node.val, path + [node.val], res)  
        dfs(node.right, t - node.val, path + [node.val], res)
```


[965. 单值二叉树](https://leetcode.cn/problems/univalued-binary-tree/)
```python
def is_unival_tree(root: Optional[TreeNode]) -> bool:  
    # 这就是个直接遍历的题  
    if root is None:  
        return True  
  
    def dfs(node: Optional[TreeNode], target: int) -> bool:
	    # 先处理node节点，实际上是个前序遍历  
        if node is None:  
            return True  
        if node.val != target:  
            return False  
        return dfs(node.left, target) and dfs(node.right, target)  
  
    return dfs(root, root.val)
```

[257. 二叉树的所有路径](https://leetcode.cn/problems/binary-tree-paths/)
```python
def binary_tree_paths(root: Optional[TreeNode]) -> List[str]:
    # 本质上就是个深度优先遍历，当遇到叶子节点的时候，特殊处理下，把路径加到结果里即可
    # 这个题非常的适合用递归，因为可以保存中间节点的path数据
    if root is None:
        return []

    def dfs(node: Optional[TreeNode], path: str, res: List[str]) -> None:
        if node:
            path += str(node.val)
            if node.left is None and node.right is None:
                res.append(path)
                return
            if node.left:
                dfs(node.left, path + "->", res)
            if node.right:
                dfs(node.right, path + "->", res)

    paths = []
    dfs(root, '', paths)
    return paths
```

[129. 求根节点到叶节点数字之和](https://leetcode.cn/problems/sum-root-to-leaf-numbers/)
```python
def sum_numbers(root: Optional[TreeNode]) -> int:
    # 这题其实跟 257. 二叉树的所有路径 那题一模一样
    # 所以这里为了模仿那一题的解题思路，res 换成了数组，实际上可以用一个全局变量来替换的
    if root is None:
        return 0

    def dfs(node: Optional[TreeNode], s: int, res: List[int]) -> None:
        if node is None:
            return
        s = s * 10 + node.val
        if node.left is None and node.right is None:
            # 叶子节点，累加结果
            res.append(s)
        else:
            dfs(node.left, s, res)
            dfs(node.right, s, res)

    path_sums = []
    dfs(root, 0, path_sums)
    return sum(path_sums)
```

[1022. 从根到叶的二进制数之和](https://leetcode.cn/problems/sum-of-root-to-leaf-binary-numbers/)
```python
def sum_root_to_leaf(root: Optional[TreeNode]) -> int:  
    # 跟 129. 求根节点到叶节点数字之和 那题基本上一样  
    def dfs(node: Optional[TreeNode], acc: int, res: List[int]) -> None:  
        if node is None:  
            return  
        acc = acc * 2 + node.val  
        if node.left is None and node.right is None:  
            res.append(acc)  
        dfs(node.left, acc, res)  
        dfs(node.right, acc, res)  
    ans = []  
    dfs(root, 0, ans)  
    return sum(ans)
```


[1315. 祖父节点值为偶数的节点和](https://leetcode.cn/problems/sum-of-nodes-with-even-valued-grandparent/)
```python
def sum_even_grand_parent(root: Optional[TreeNode]) -> int:  
    # 转变一下视角，从找祖先变成找孙子节点  
    # 然后就变成了实际上只是一个深度遍历的题，找到符合条件的节点，把节点值累加一下即可  
    ans = 0  
    if root is None:  
        return ans  
    if root.val % 2 == 0:  
        if root.left and root.left.left:  
            ans += root.left.left.val  
        if root.left and root.left.right:  
            ans += root.left.right.val  
        if root.right and root.right.left:  
            ans += root.right.left.val  
        if root.right and root.right.right:  
            ans += root.right.right.val  
    ans += sum_even_grand_parent(root.left)  
    ans += sum_even_grand_parent(root.right)  
    return ans
```

[1448. 统计二叉树中好节点的数目](https://leetcode.cn/problems/count-good-nodes-in-binary-tree/)
```python
def good_nodes(root: Optional[TreeNode]) -> int:
    # 这题其实跟 257. 二叉树的所有路径 那题一模一样，只是路径变成了路径上的最大值
    # 所以这里为了模仿那一题的解题思路，res 换成了数组，实际上可以用一个全局变量来替换的
    if root is None:
        return 0

    def dfs(node: Optional[TreeNode], max_value: int, res: List[TreeNode]) -> None:
        if node is None:
            return
        if node.val >= max_value:
            res.append(node)
            max_value = node.val
        dfs(node.left, max_value, res)
        dfs(node.right, max_value, res)

    path_nodes = []
    dfs(root, root.val, path_nodes)
    return len(path_nodes)
```


[2265. 统计值等于子树平均值的节点数](https://leetcode.cn/problems/count-nodes-equal-to-average-of-subtree/)
```python
def average_of_subtree(root: Optional[TreeNode]) -> int:  
    # 先找子节点，再找父节点的，就用后序遍历  
    if root is None:  
        return 0  
    result = []  
  
    def dfs(node: Optional[TreeNode]) -> Tuple[int, int]:  
        if node is None:  
            return 0, 0  
        left_num, left_sum = dfs(node.left)  
        right_num, right_sum = dfs(node.right)  
        mid = (left_sum + right_sum + node.val) / (left_num + right_num + 1)  
        if node.val == mid:  
            result.append(node)  
        return left_num + right_num + 1, left_sum + right_sum + node.val  
  
    dfs(root)  
    return len(result)
```

[814. 二叉树剪枝](https://leetcode.cn/problems/binary-tree-pruning/)
```python
def prune_tree(root: Optional[TreeNode]) -> Optional[TreeNode]:
	# 先找子节点，再找父节点的，就用后序遍历 
	if root is None:
		return root
	root.left = prune_tree(root.left)
	root.right = prune_tree(root.right)
	# 处理父节点，如果左右子树都是None，且root.val=0，则当前节点需要被删除，返回None
	if root.left is None and root.right is None and root.val == 0:
		return None
	return root
```

[1110. 删点成林](https://leetcode.cn/problems/delete-nodes-and-return-forest/)
```python
def del_nodes(root: Optional[TreeNode], to_delete: List[int]) -> List[TreeNode]:
	# 本质上还是一个后序遍历，从子节点开始删，一路往上遍历
	# 如果子节点是需要被删除的，就返回None，否则返回这个子节点
	def dfs(node: Optional[TreeNode], target: List[int], collector: List[TreeNode]) -> Optional[TreeNode]:
		if node is None:
			return None
		node.left = dfs(node.left, target, collector)
		node.right = dfs(node.right, target, collector)
		if node.val in target:
		    # 这里可以删掉列表里的元素，因为题目要求说了树节点的值不会重复
			target.remove(node.val)
			if node.left is not None:
				collector.append(node.left)
			if node.right is not None:
				collector.append(node.right)
			return None
		return node
	res = []
	n = dfs(root, to_delete, res)
	# 最后要处理下root节点
	if n is not None:
		res.append(n)
	return res
```

[101. 对称二叉树](https://leetcode.cn/problems/symmetric-tree/)
```python
def is_symmetric(root: Optional[TreeNode]) -> bool:
	# 这实际上是个前序遍历的题加上需要满足对称的条件，而轴对称实际上又可以转化为
	# 满足对称的条件分析：
	#  1. root is None，直接返回True 
	#  2. 左子树和右子树镜像对称
	if root is None:
	    return True
	
	# 左右子树镜像对称，通过前序遍历递归求解
	# 两棵树满足镜像对称的条件：
	# 1. p，q都是None，直接返回True
	# 2. p，q一棵是None，另一棵不是，直接返回False
	# 3. p，q都不是None的情况下，其值相等且满足p左子树和q右子树镜像对称且p右子树和q左子树镜像对称，才返回True，否则返回False
	def is_symmetric_inner(p: Optional[TreeNode], q: Optional[TreeNode]) -> bool:
		if not p and not q:
			return True
		if p and q:
			return p.val == q.val and is_symmetric_inner(p.left, q.right) and is_symmetric_inner(p.right, q.right)
		return False

	return is_symmetric_inner(root.left, root.right)

```

[110. 平衡二叉树](https://leetcode.cn/problems/balanced-binary-tree/)
```python
def is_balanced(root: Optional[TreeNode]) -> bool:
    # 自底向上的后序遍历题，
    def height(node: Optional[TreeNode]) -> int:
        if node is None:
            return 0
        left_height = height(node.left)
        if left_height == -1:
            return -1
        right_height = height(node.right)
        if right_height == -1:
            return -1
        if abs(left_height - right_height) > 1:
            return -1
        return max(left_height, right_height) + 1

    return height(root) >= 0
```

[589. N 叉树的前序遍历](https://leetcode.cn/problems/n-ary-tree-preorder-traversal/)
```python
def preorder(root: 'Node') -> List[int]:
	# N叉树的前序遍历，实际上跟二叉树的前序遍历是一样的模版
	def dfs(node: 'Node', collector: List[int]):
		if node is None:
			return
		collector.append(node.val)
		if not node.children:
			return
		for c in node.children:
			dfs(c)
	res = []
	dfs(root, res)
	return res
```


#### 1.1.1.2 **从遍历结果还原**
前面三种遍历方式，我们把数据都存储在数组里，现在我们也可以从数组里还原二叉树。先来看一下不同遍历方式所得到的数组的结构：
- 前序遍历的数组结构是[root节点，左子树，右子树]，继续往下拆分成[root节点，左子树root节点，左子树的左子树，左子树的右子树，右子树root节点，右子树的左子树，右子树的右子树]，还可以继续往下拆直到每个节点无法继续拆分为止
- 中序遍历的数据结构是[左子树，root节点，右子树]，继续往下拆分成[左子树的左子树，左子树root节点，左子树的右子树，root节点，右子树的左子树，右子树root节点，右子树的右子树]，还可以继续往下拆直到每个节点无法继续拆分为止
- 后序遍历的数组结构是[左子树，右子树，root节点]，继续往下拆分成[左子树的左子树，左子树的右子树，左子树root节点，右子树的左子树，右子树的右子树，右子树的root节点，root节点]，还可以继续往下拆直到每个节点无法继续拆分为止

了解这三个拆分规则，就可以尝试从从遍历结果里还原二叉树了。

[105. 从前序与中序遍历序列构造二叉树](https://leetcode.cn/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)
```python
def build_tree(preorder: List[int], inorder: List[int]) -> Optional[TreeNode]:  
    if preorder is None or inorder is None:  
        return None  
    if len(preorder) == 0 or len(inorder) == 0:  
        return None  
    root = TreeNode()  
    # 前序遍历结构[root节点的值,左子树,右子树]  
    root.val = preorder[0]  
    # 中序遍历结构[左子树,root节点的值,右子树]  
    index = inorder.index(root.val)
    # 这里需要注意计算下左子树的节点个数  
    root.left = build_tree(preorder[1: index + 1], inorder[0:index])  
    root.right = build_tree(preorder[index + 1:], inorder[index + 1:])  
    return root
```

[106. 从中序与后序遍历序列构造二叉树](https://leetcode.cn/problems/construct-binary-tree-from-inorder-and-postorder-traversal/)
```python
# 几乎是跟105一模一样的套路，只需要计算对左右子树在数组中的分界位置就可以
def build_tree(inorder: List[int], postorder: List[int]) -> Optional[TreeNode]:  
    if inorder is None or postorder is None:  
        return None  
    if len(inorder) == 0 or len(postorder) == 0:  
        return None  
    root = TreeNode()  
    # 后序遍历结构[左子树,右子树,root节点的值]  
    root.val = postorder[len(postorder) - 1]  
    # 中序遍历结构[左子树,root节点的值,右子树]  
    index = inorder.index(root.val)  
    root.left = build_tree(inorder[0: index], postorder[0: index])  
    root.right = build_tree(inorder[index + 1:], postorder[index: len(postorder) - 1])  
    return root
```

[889. 根据前序和后序遍历构造二叉树](https://leetcode.cn/problems/construct-binary-tree-from-preorder-and-postorder-traversal/)
```python
def build_tree(preorder: List[int], postorder: List[int]) -> Optional[TreeNode]:  
    if preorder is None or postorder is None:  
        return None  
    if len(preorder) == 0 or len(postorder) == 0:  
        return None  
    root = TreeNode()  
    # 前序遍历结构[root节点的值,左子树,右子树]  
    root.val = preorder[0]  
    # 后序遍历结构[左子树,右子树,root节点的值]  
    # 前序遍历的左子树根节点,在后序遍历序列里面分割了左右子树  
    # 即前序遍历在root节点之后遍历到的左子树第一个节点，是后序遍历左子树最后一个遍历到的节点  
    if len(preorder) == 1:  
        return root  
    index = postorder.index(preorder[1])  
    root.left = build_tree(preorder[1:index + 2], postorder[0: index + 1])  
    root.right = build_tree(preorder[index + 2:], postorder[index + 1: len(postorder) - 1])  
    return root
```


上面几题都是需要两种遍历方式的结果才能还原二叉树，但有一些变种的题，会用其他条件取代其中一种遍历结果，需要通过一种遍历结果+其他条件的方式来解题。

[1028. 从先序遍历还原二叉树](https://leetcode.cn/problems/recover-a-tree-from-preorder-traversal/)
```python

```



#### 1.1.2 莫里斯遍历


## 1.2 宽度优先遍历
宽度优先遍历只有一种，所以会更简单一点。

[102. 二叉树的层序遍历](https://leetcode.cn/problems/binary-tree-level-order-traversal/)
```python
def level_order(root: Optional[TreeNode]) -> List[List[int]]:  
    if root is None:  
        return []  
    level_nodes = [root]  
    res = []  
    while level_nodes:  
        next_level_nodes = []  
        current_level_values = []  
        for node in level_nodes:  
            current_level_values.append(node.val)  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        res.append(current_level_values)  
        level_nodes = next_level_nodes  
    return res
```

[LCR 149. 彩灯装饰记录 I](https://leetcode.cn/problems/cong-shang-dao-xia-da-yin-er-cha-shu-lcof/)
```python
def decorate_record(root: Optional[TreeNode]) -> List[int]:  
    # 直接套宽度优先遍历的模版即可  
    if root is None:  
        return []  
    res, cur_level = [], [root]  
    while cur_level:  
        next_level = []  
        for node in cur_level:  
            res.append(node.val)  
            if node.left:  
                next_level.append(node.left)  
            if node.right:  
                next_level.append(node.right)  
        cur_level = next_level  
    return res
```

[LCR 150. 彩灯装饰记录 II](https://leetcode.cn/problems/cong-shang-dao-xia-da-yin-er-cha-shu-ii-lcof/)
```python
def decorate_record(root: Optional[TreeNode]) -> List[int]:  
    # 直接套宽度优先遍历的模版即可  
    if root is None:  
        return []  
    res, cur_level = [], [root]  
    while cur_level:  
        next_level, cur_values = [], []  
        for node in cur_level:  
            cur_values.append(node.val)  
            if node.left:  
                next_level.append(node.left)  
            if node.right:  
                next_level.append(node.right)  
        cur_level = next_level
        res.append(cur_values) 
    return res
```

[LCR 151. 彩灯装饰记录 III](https://leetcode.cn/problems/cong-shang-dao-xia-da-yin-er-cha-shu-iii-lcof/)
```python
def decorate_record(root: Optional[TreeNode]) -> List[int]:  
    # 跟锯齿形层序遍历是一样题
    if root is None:  
        return []  
    res, cur_level, flag = [], [root], True  
    while cur_level:  
        next_level, cur_values = [], []  
        for node in cur_level:  
            cur_values.append(node.val)  
            if node.left:  
                next_level.append(node.left)  
            if node.right:  
                next_level.append(node.right)  
        cur_level = next_level
        if flag:
	        res.append(cur_values)
	    else:
		    res.append(cur_values[:-1])
        flag = not flag
    return res
```

[103. 二叉树的锯齿形层序遍历](https://leetcode.cn/problems/binary-tree-zigzag-level-order-traversal/)
```python
def zigzag_level_order(root: Optional[TreeNode]) -> List[List[int]]:  
    if root is None:  
        return []  
    level_nodes = [root]  
    res = []  
    # 完全就是套模版，只需要加一个标记，用来控制当前是从尾部加入新元素还是从头部加入新元素  
    flag = False  
    while level_nodes:  
        current_level_values = []  
        next_level_nodes = []  
        for node in level_nodes:  
            if flag:  
                current_level_values.insert(0, node.val)  
            else:  
                current_level_values.append(node.val)  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        level_nodes = next_level_nodes  
        flag = not flag  
        res.append(current_level_values)  
    return res
```

[107. 二叉树的层序遍历 II](https://leetcode.cn/problems/binary-tree-level-order-traversal-ii/)
```python
def level_order(root: Optional[TreeNode]) -> List[List[int]]:  
    if root is None:  
        return []  
    level_nodes = [root]  
    res = []  
    while level_nodes:  
        next_level_nodes = []  
        current_level_values = []  
        for node in level_nodes:  
            current_level_values.append(node.val)  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        res.append(current_level_values)  
        level_nodes = next_level_nodes
    # 这个就是直接reverse就行，实在不行就res.append改成res.insert从头部插入新元素  
    return res[::-1]
```

[199. 二叉树的右视图](https://leetcode.cn/problems/binary-tree-right-side-view/)```python
```python
def right_side_view(root: Optional[TreeNode]) -> List[int]:  
    # 右视图，实际上看到的就是层序遍历结果的所有层次的最后一个节点的值  
    if root is None:  
        return []  
    level_nodes = [root]  
    res = []  
    while level_nodes:  
        next_level_nodes = []  
        # 直接取最后一个节点的值放进来就行  
        res.append(level_nodes[-1].val)  
        for node in level_nodes:  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        level_nodes = next_level_nodes  
    return res
```

[515. 在每个树行中找最大值](https://leetcode.cn/problems/find-largest-value-in-each-tree-row/)
```python
def largest_values(root: Optional[TreeNode]) -> List[int]:  
	# 套模版，求个最大值即可
    if root is None:  
        return []  
    level_nodes = [root]  
    res = []  
    while level_nodes:  
        next_level_nodes = []  
        largest = float("-inf")  
        for node in level_nodes:  
            largest = max(largest, node.val)  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        res.append(largest)  
        level_nodes = next_level_nodes  
    return res
```

[637. 二叉树的层平均值](https://leetcode.cn/problems/average-of-levels-in-binary-tree/)
```python
def average_of_levels(root: Optional[TreeNode]) -> List[float]:  
    # 套模版，简单的求个平均数  
    if root is None:  
        return []  
    level_nodes = [root]  
    res = []  
    while level_nodes:  
        next_level_nodes = []  
        node_sum = 0  
        for node in level_nodes:  
            node_sum += node.val  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        res.append(node_sum / len(level_nodes))  
        level_nodes = next_level_nodes  
    return res
```

[623. 在二叉树中增加一行](https://leetcode.cn/problems/add-one-row-to-tree/)
```python
def add_one_row(root: Optional[TreeNode], val: int, depth: int) -> Optional[TreeNode]:
    # 像这种操作一行的题，直接BFS准没错
    if root is None:  
        return root  
    if depth == 1:  
        res = TreeNode()  
        res.val, res.left, res.right = val, root, None  
        return res  
    level_nodes = [root]  
    dep = 1  
    while level_nodes:  
        next_level_nodes = []  
        if dep == depth - 1:  
            for node in level_nodes:  
                new_left = TreeNode()  
                new_left.val, new_left.left, new_left.right = val, node.left, None  
                new_right = TreeNode()  
                new_right.val, new_right.left, new_right.right = val, None, node.right  
                node.left, node.right = new_left, new_right  
            return root  
  
        for node in level_nodes:  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        level_nodes = next_level_nodes  
        dep += 1  
    return None
```

[513. 找树左下角的值](https://leetcode.cn/problems/find-bottom-left-tree-value/)
```python
def find_bottom_left_value(root: Optional[TreeNode]) -> int:  
    #  找最底层、最左边节点的值，直接BFS拿到最后一层的值就行  
    level_nodes = [root]  
    while level_nodes:  
        next_level_nodes = []  
        for node in level_nodes:  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        if not next_level_nodes:  
            return level_nodes[0].val  
        level_nodes = next_level_nodes  
    return -1
```

[104.二叉树的最大深度](https://leetcode.cn/problems/maximum-depth-of-binary-tree/)
```python
def max_depth(root: Optional[TreeNode]) -> int:  
    # 这种就是DFS和BFS都可以解的题，实际上就是考一个遍历而已  
    if root is None:  
        return 0  
    level_nodes = [root]  
    depth = 0  
    while level_nodes:  
        next_level_nodes = []  
        depth += 1  
        for node in level_nodes:  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        level_nodes = next_level_nodes  
    return depth
```

[111. 二叉树的最小深度](https://leetcode.cn/problems/minimum-depth-of-binary-tree/)
```python
def min_depth(root: Optional[TreeNode]) -> int:  
    # 这种就是DFS和BFS都可以解的题，实际上就是考一个遍历而已  
    if root is None:  
        return 0  
    level_nodes = [root]  
    depth = 0  
    while level_nodes:  
        depth += 1  
        next_level_nodes = []  
        for node in level_nodes:  
            if node.left is None and node.right is None:  
                # 第一个遍历到的叶子节点，一定是深度最小的节点  
                return depth  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        level_nodes = next_level_nodes  
    return depth
```

[404. 左叶子之和](https://leetcode.cn/problems/sum-of-left-leaves/)
```python
def sum_of_left_leaves(root: Optional[TreeNode]) -> int:  
    # BFS找左叶子即可  
    if root is None:  
        return 0  
    if root.left is None and root.right is None:  
        return 0  
    level_nodes = [root]  
    res = 0  
    while level_nodes:  
        next_level_nodes = []  
        for node in level_nodes:  
            if node.left:  
                next_level_nodes.append(node.left)
                # 注意这里找左叶子
                if node.left.left is None and node.left.right is None:  
                    res += node.left.val  
            if node.right:  
                next_level_nodes.append(node.right)  
        level_nodes = next_level_nodes  
    return res
```

[1161. 最大层内元素和](https://leetcode.cn/problems/maximum-level-sum-of-a-binary-tree/)
```python
def max_level_sum(root: Optional[TreeNode]) -> int:  
    # BFS求每一行的和，比较一下即可  
    if root is None:  
        return 1  
    depth, res, index, level_nodes = 1, float("-INF"), 1, [root]  
    while level_nodes:  
        next_level_nodes, level_sum = [], 0  
        for node in level_nodes:  
            level_sum += node.val  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        if level_sum > res:  
            depth, res = index, level_sum  
        level_nodes = next_level_nodes  
        index += 1  
    return depth
```

[1302. 层数最深叶子节点的和](https://leetcode.cn/problems/deepest-leaves-sum/)
```python
def deepest_leaves_sum(root: Optional[TreeNode]) -> int:  
    # 其实就是BFS最后一层的节点和  
    if root is None:  
        return 0  
    level_nodes, res = [root], 0  
    while level_nodes:  
        next_level_nodes, level_sum = [], 0  
        for node in level_nodes:  
            level_sum += node.val  
            if node.left:  
                next_level_nodes.append(node.left)  
            if node.right:  
                next_level_nodes.append(node.right)  
        level_nodes = next_level_nodes  
        res = level_sum  
    return res
```

[662. 二叉树最大宽度](https://leetcode.cn/problems/maximum-width-of-binary-tree/)
```python
def width_of_binary_tree(root: Optional[TreeNode]) -> int:  
    # 实际上是个宽度有限遍历，但是要结合一个给节点编号的技巧  
    # 假设父节点的索引是index，那么左子树索引是2 * index，右子树是 2 * index + 1    # root节点索引从1开始  
    if root is None:  
        return 1  
    res = 1  
    level_nodes = [[root, 1]]  
    while level_nodes:  
        next_level_nodes = []  
        for node, index in level_nodes:  
            if node.left:  
                next_level_nodes.append([node.left, 2 * index])  
            if node.right:  
                next_level_nodes.append([node.right, 2 * index + 1])  
        res = max(res, level_nodes[-1][1] - level_nodes[0][1] + 1)  
        level_nodes = next_level_nodes  
    return res
```