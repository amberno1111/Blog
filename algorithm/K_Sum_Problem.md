# K-Sum问题

在 LeetCode 上有 [Two Sum](https://leetcode.com/problems/two-sum/)，[Three Sum](https://leetcode.com/problems/3sum/)，[Three Sum Closest](https://leetcode.com/problems/3sum-closest/) 等等问题，这些问题实际上有比较通用的解法，所以这篇文章记录了一下对于这种K-Sum问题的解题思路。

## Two-Sum

> Given an array of integers, return indices of the two numbers such that they add up to a specific target.
You may assume that each input would have exactly one solution, and you may not use the same element twice.
Example:
Given nums = [2, 7, 11, 15], target = 9
Because nums[0] + nums[1] = 2 + 7 = 9
return [0, 1]

### 暴力解法

对于这种题，第一反应是暴力解法，直接两个 for 循环遍历，时间复杂度 O(N*N)，这种解法基本上是效率最低的:

```python
class Solution(object):
    @staticmethod
    def twoSum0(nums: [int], target: int) -> [int]:
        """
        两层循环暴力解法
        :param nums:
        :param target:
        :return:
        """
        l: int = len(nums)
        for i in range(l):
            for j in range(i + 1, l):
                if nums[i] + nums[j] == target:
                    return [i, j]
        return []
```

### 以空间换时间

可以看到，暴力解法的第二层循环实际上已经遍历过所有的元素了，如果，我们能把遍历到的元素保存下来，然后以 O(1) 的时间来做查找，那效率就会大大的提高，时间复杂度可以提升到 O(N) 。

```python
class Solution(object):
    @staticmethod
    def twoSum1(nums: [int], target: int) -> [int]:
        """
        思路相对来说是比较简单的，就是遍历数组，然后用 target - nums[i]
        并且以 target - nums[i] 为 key，i 为 value 保存到一个临时的 map 里
        时间复杂度：O(n)
        空间复杂度：O(n)
        :param nums:
        :param target:
        :return:
        """
        lookup = {}
        for index, value in enumerate(nums):
            if value in lookup:
                return [lookup[value], index]
            else:
                lookup[target - value] = index
        return []
```

### 如果数组有序的

Two Sum 问题有一个变种，如果数组已经按照从小大排序，能否实现 O(N) 的时间和 O(1) 空间复杂度的解法？原题如下：
> Given an array of integers that is already sorted in ascending order, find two numbers such that they add up to a specific target number.
The function twoSum should return indices of the two numbers such that they add up to the target, where index1 must be less than index2.
Node: 
> 1. You returned answers (both index1 and index2) are not zero-based
> 2. You may assume the each input would exactly one solution and you may not use the same element twice.
>
> Example:
Input: numbers = [2, 7, 11, 15], target = 9
Output: [1, 2]
Explanation: The sum of 2 and 7 is 9, Therefore index1 = 1, index2 = 2


```python
class Solution(object):
    @staticmethod
    def twoSum(numbers: [int], target: int) -> [int]:
        """
        这个题目看起来和 two-sum 差不多，基本上也是可以用相同的方法来解的
        但是这里既然题目里多了一个数组已排序的条件，就可以把这个条件用起来
        使用双指针的方法，可以达到 O(N) 的时间复杂度和 O(1) 的空间复杂度
        左右两个指针，分别从列表的左右两端开始遍历，如果：
        1. numbers[left] + number[right] = target, 说明找到了结果，直接返回
        2. numbers[left] + number[right] < target, 因为数组是从小达到排序的，所以这时候只能找一个更大的值相加才行，所以 left + 1
        3. number[left] + number[right] > target, 条件 2 的反面，right - 1
        :param numbers:
        :param target:
        :return:
        """
        length = len(numbers)
        left, right = 0, length - 1
        for i in range(length):
            if numbers[left] + numbers[right] == target:
                return [left + 1, right + 1]
            if numbers[left] + numbers[right] < target:
                left += 1
            else:
                right -= 1
        return []
```

## Three Sum

再来看 Three Sum 问题：

> Given an array nums of n integers, are there elements a, b, c in nums such that a + b + c = 0?
Find all unique triplets in the array which gives the sume of zero.
Note: The solution set must not contain duplicate triplets.
Example:
Given array nums = [-1, 0, 1, 2, -1, -4],
A solution set is:
[ [-1, 0, 1], [-1, -1, 2] ]

这题的难度稍微大一点，出现了三个变量，并且还需要去重。暴力解法这里就不继续写了，因为没啥技术含量。

### 借用 Two Sum 的思路

稍微好点的解法是，双重 `for` 循环，然后第二重循环调用 two-sum 的解法，就相当于把题目变成了 `a + b = -c` ，然后 `a, b, c` 都是来自于这个数组，这种解法的时间复杂度是 O(N * N) ，空间复杂度是 O(1) 。

```python
class Solution(object):
    @staticmethod
    def threeSum(nums: [int]) -> [[int]]:
        """
        主要的思路是把 three sum 转成 two sum，遍历当前的数组，用 0 减去当前的数作为 sum，再找两个数使得和为 sum
        我们首先对数组进行排序，这样的话去重就可以变得非常容易
        :param nums:
        :return:
        """
        nums.sort()
        result = []
        length = len(nums)
        # len(nums) - 2 是因为至少需要有三个数
        for i in range(length - 2):
            # 因为已经排过序了，后面的数肯定比当前的数大，所以后面肯定找不到两个数加起来是负数(0 - nums[i] < 0)的解了
            if nums[i] > 0:
                break
            # 去重
            if i > 0 and nums[i] == nums[i - 1]:
                continue
            left, right = i + 1, length - 1
            while left < right:
                total = nums[i] + nums[left] + nums[right]
                if total < 0:
                    left += 1
                elif total > 0:
                    right -= 1
                else:
                    result.append([nums[i], nums[left], nums[right]])
                    # 对 left 和 right 去重
                    while left < right and nums[left] == nums[left + 1]:
                        left += 1
                    while left < right and nums[right] == nums[right - 1]:
                        right -= 1
                    left += 1
                    right -= 1
        return result
```

### Three Sum Cloest

Three Sum 的变种 [Three Sum Cloest](https://leetcode.com/problems/3sum-closest/) ，实际上并没有太多的变化。

> Given an array nums of n integers and an integer target,
 find three integers in nums such that the sum is closest to target.
 You may assume that each input would have exactly one solution.
 Example:
 Given array nums = [-1, 2, 1, -4], target = 1
 The sum that is closest to the target is 2 (-1 + 2 + 1).

```python
class Solution(object):
    @staticmethod
    def threeSumClosest(nums: [int], target: int) -> int:
        """
        这个是 three sum 的变种，但是感觉起来稍微简单一点，因为不需要去重了
        题目给定了只会有一个解
        思路还是一样，three sum 变成 two sum 求解，时间复杂度 O(N*N)
        :param nums:
        :param target:
        :return:
        """
        nums.sort()
        length, result = len(nums), nums[0] + nums[1] + nums[2]
        for i in range(length - 2):
            left, right = i + 1, length - 1
            while left < right:
                s = nums[i] + nums[left] + nums[right]
                if s == target:
                    return s
                if abs(target - s) < abs(target - result):
                    result = s
                if s < target:
                    left += 1
                else:
                    right -= 1
        return result
```


## Four Sum

先来看一下题目，这其实上和 Three Sum 没有太大区别：

> Given an array nums of integers and an integer target,
 are there elements a, b, c, d in nums such that a + b + c + d = target?
 Find all unique quadruplets in the array which gives the sum of target.
 Note: The Solution set must not contain duplicate quadruplets.
 Example:
 Given array nums = [1, 0, -1, 0, -2, 2], target = 0
 A solution is:
 [ [-1,  0, 0, 1], [-2, -1, 1, 2], [-2,  0, 0, 2] ]

 
解题思路也差不多，就是降级到 Three Sum，再降级到 Two Sum 然后用双指针解决，时间复杂度 O(N\*N\*N)，这个解题思路适用于所有的 K-Sum 问题。

先来看一个不那么通用的解法：

```python
class Solution(object):
    @staticmethod
    def fourSum(nums: [int], target: int) -> [[int]]:
        """
        解题思路就是写一个双指针实现的 three sum，然后递归的解决 four sum 问题
        这个思路基本上可以适用于所有的 k sum 问题
        当然这里的写法没有太多的通用性，想要更通用的解法可以看下面的 kSum 方法
        :param nums:
        :param target:
        :return:
        """
        nums.sort()
        length, result = len(nums), []
        for i in range(length - 3):
            # 先去重
            if i == 0 or nums[i] != nums[i - 1]:
                temp_result = Solution.threeSum(nums[i + 1:], target - nums[i])
                for item in temp_result:
                    result.append([nums[i]] + item)
        return result

    @staticmethod
    def threeSum(nums: [int], target: int) -> [[int]]:
        # 这里求解数组长度实际上是不必要的，完全可以上层传进来
        length, result = len(nums), []
        for i in range(length - 2):
            # 去重
            if i == 0 or nums[i] != nums[i - 1]:
                left, right = i + 1, length - 1
                while left < right:
                    s = nums[i] + nums[left] + nums[right]
                    if s == target:
                        result.append([nums[i], nums[left], nums[right]])
                        # 去重
                        while left < right and nums[right] == nums[right - 1]:
                            right -= 1
                        while left < right and nums[left] == nums[left + 1]:
                            left += 1
                        left += 1
                        right -= 1
                    elif s < target:
                        left += 1
                    else:
                        right -= 1
        return result
```


## K Sum

我们可以继续把问题扩展到 K Sum，然后使用递归来解决问题。使用递归的话有一个固定的套路，先找公共抽象。



```python
class Solution(object):
    @staticmethod
    def kSum(nums: [int], target: int, k: int) -> [[int]]:
        """
        k sum 递归解法
        :param nums: 给定的列表，假设列表里的元素已经从小到大排序
        :param start: 从列表的 start index 开始搜索
        :param end: 搜索到列表的 end index 即可停止
        :param target: 目标数字
        :param k: k sum 问题的参数 k
        :return:
        """
        # 首先我们基本确定公共问题的最小解是 two sum 问题
        # 所以这里先写一个高效的 two sum 解题
        length = len(nums)
        if k == 2:
            temp_result = []
            left, right = 0, length - 1
            while left < right:
                s = nums[left] + nums[right]
                if s == target:
                    temp_result.append([nums[left], nums[right]])
                    while left < right and nums[right] == nums[right - 1]:
                        right -= 1
                    while left < right and nums[left] == nums[left + 1]:
                        left += 1
                    left += 1
                    right -= 1
                elif s < target:
                    left += 1
                else:
                    right -= 1
            return temp_result
        else:
            # 然后我们来缩小公共问题的范围，递归求解
            result = []
            for i in range(0, length - k + 1):
                # 去重
                if i == 0 or nums[i] != nums[i - 1]:
                    temp_res = Solution.kSum(nums[i+1:], target - nums[i], k - 1)
                    for item in temp_res:
                        result.append([nums[i]] + item)
            return result

```
