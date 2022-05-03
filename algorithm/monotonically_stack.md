# [LeetCode]一文刷完所有的单调栈技巧题

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