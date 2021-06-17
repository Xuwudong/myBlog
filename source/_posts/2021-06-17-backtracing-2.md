---
title: 算法的奥秘-回溯（2）
tags:
  - 回溯
categories:
  - 数据结构与算法
date: 2021-06-17 20:02:30
---

&ensp;&ensp;今天分享一下最近学习回溯算法的一些经验与心得！根据百度百科的解释：回溯算法实际上一个类似枚举的搜索尝试过程，主要是在搜索尝试过程中寻找问题的解，当发现已不满足求解条件时，就“回溯”返回，尝试别的路径。回溯法是一种选优搜素法，按选优条件向前搜索，以达到目标。但当探索到某一步时，发现原先选择并不优或达不到目标，就退回一步重新选择，这种走不通就退回再走的技术为回溯法，而满足回溯条件的某个状态的点称为“回溯点”。许多复杂的，规模较大的问题都可以使用回溯法，有“通用解题方法”的美称。直接看题，毁灭吧，赶紧的！

&ensp;&ensp;我们知道二叉树的前序遍历dfs的代码是这样写的：

```bash
public void treeDFS(TreeNode root) {
    if (root == null)
        return;
    System.out.println(root.val);
    treeDFS(root.left);
    treeDFS(root.right);
}
```
&ensp;&ensp;这里可以通过仿照一下二叉树，写一下9叉树前序遍历的代码：

```bash

public void treeDFS(TreeNode node) {
    // 递归终止条件
    if(node == null) {
        return;
    }
    System.out.println(node.val);
    for(int i = 0;i < 9;i++) {
        // 一些操作，可有可无，视情况而定
        treeDfs("第i个节点");
        // 回退操作，可有可无，视情况而定
    }
}
```

&ensp;&ensp;这就是回溯算法的基本框架，几乎所有回溯算法都离不开这个框架，接下来我们看几个实例。

1、全排列

给定一个 没有重复 数字的序列，返回其所有可能的全排列。


示例:

输入: [1,2,3]

输出:

[
  [1,2,3],

  [1,3,2],

  [2,1,3],

  [2,3,1],

  [3,1,2],

  [3,2,1]

]
这其实是一颗三叉树，根据上面的框架，写出代码：

```bash
public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        Deque<Integer> deque = new ArrayDeque<>();
        int n = nums.length;
        boolean[] used = new boolean[n];
        dfs(res, deque, n, 0, used, nums);
        return res;
    }

    /**
     * dfs
     * @param res res 
     * @param deque deque
     * @param n  n
     * @param first 层数
     * @param used 是否有使用到数组标记
     * @param nums nums
     */
    private void dfs(List<List<Integer>> res, Deque<Integer> deque, int n, int first, boolean[] used, int[] nums) {
        if (first == n) {
            res.add(new ArrayList<>(deque));
            return;
        }
        // 注意i= 0
        for (int i = 0; i < n; i++) {
            if (!used[i]) {
                deque.addLast(nums[i]);
                used[i] = true;
                // 这里是first +1 ,first代表层数
                dfs(res, deque, n, first + 1, used, nums);
                deque.removeLast();
                used[i] = false;
            }
        }
    }
```

注意点：
* for 循环 i 值是从0开始的，这样才能出现【3,2，1】这样的集合。
* dfs 递归传的是first + 1，而不是i+1,因为first这里表示的是遍历层数，而i表示的是遍历数组的索引。也可以去掉first参数。直接将deque.size() == n 作为递归终止条件。

2、全排列 || 

给定一个可包含重复数字的序列 nums ，按任意顺序 返回所有不重复的全排列。

示例 1：

输入：nums = [1,1,2]

输出：

[[1,1,2],

 [1,2,1],

 [2,1,1]]

示例 2：

输入：nums = [1,2,3]

输出：[[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]

直接看代码

```bash
    public List<List<Integer>> permuteUnique(int[] nums) {
        List<List<Integer>> list = new ArrayList<>();
        Deque<Integer> deque = new ArrayDeque<>();
        boolean[] used = new boolean[nums.length];
        // 有重复的队列，先排序
        Arrays.sort(nums);
        dfs(list, deque, nums, 0, used);
        return list;
    }

    private void dfs(List<List<Integer>> list, Deque<Integer> deque, int[] nums, int first, boolean[] used) {
        if (first == nums.length) {
            list.add(new ArrayList<>(deque));
            return;
        }
        for (int i = 0; i < nums.length; i++) {
            // 去重
            if (used[i] || (i > 0 && nums[i] == nums[i - 1] && !used[i - 1])) {
                continue;
            }
            deque.addLast(nums[i]);
            used[i] = true;
            dfs(list, deque, nums, first + 1, used);
            used[i] = false;
            deque.removeLast();
        }
    }
```

注意点：
    
* 这里需要去重，最简单的做法就是先将nums排序，这样相同的元素就在一起了，然后在第一题的基础上加上这句就行了

```bash
if (used[i] || (i > 0 && nums[i] == nums[i - 1] && !used[i - 1])) {
    continue;
}
```

3、子集

给你一个整数数组 nums ，数组中的元素 互不相同 。返回该数组所有可能的子集（幂集）。
解集 不能 包含重复的子集。你可以按 任意顺序 返回解集。

示例 1：

输入：nums = [1,2,3]

输出：[[],[1],[2],[1,2],[3],[1,3],[2,3],[1,2,3]]

示例 2：

输入：nums = [0]

输出：[[],[0]]

```bash
    public List<List<Integer>> subsets(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        Deque<Integer> deque = new ArrayDeque<>();
        dfs(res, deque, nums, 0);
        return res;
    }

    private void dfs(List<List<Integer>> res, Deque<Integer> deque, int[] nums, int index) {
        res.add(new ArrayList<>(deque));
        for (int i = index; i < nums.length; i++) {
            deque.addLast(nums[i]);
            dfs(res, deque, nums, i + 1);
            deque.removeLast();
        }
    }
```

注意点:

* for 循环 i的初始值应该是index,如果i = 0的话，nums[i]就重复了
* dfs递归时传的index= i+ 1，而不是index+ 1，如果传的是index + 1,那么for循环那么多次递归进去的值都是一样的。

4、子集||

给定一个可能包含重复元素的整数数组 nums，返回该数组所有可能的子集（幂集）。

说明：解集不能包含重复的子集。

示例:

输入: [1,2,2]

输出:

[

  [2],

  [1],

  [1,2,2],

  [2,2],

  [1,2],

  []

]

```bash

    public List<List<Integer>> subsetsWithDup(int[] nums) {
        Arrays.sort(nums);
        List<List<Integer>> res = new ArrayList<>();
        Deque<Integer> deque = new ArrayDeque<>();
        dfs(res, deque, nums, 0);
        return res;
    }

    private void dfs(List<List<Integer>> res, Deque<Integer> deque, int[] nums, int index) {
        res.add(new ArrayList<Integer>(deque));
        for (int i = index; i < nums.length; i++) {
            if (i > index && nums[i] == nums[i - 1]) {
                continue;
            }
            deque.addLast(nums[i]);
            dfs(res, deque, nums, i + 1);
            deque.removeLast();
        }
    }
```

注意点：

* 这里跟第二题求全排列一样的思路，要去重就先排序，将重复的元素放在一起，然后通过这行代码过滤掉：
```bash
if (i > index && nums[i] == nums[i - 1]) {
                continue;
}
```

5、组合求和

给定一个无重复元素的数组 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合。

candidates 中的数字可以无限制重复被选取。

说明：

所有数字（包括 target）都是正整数。

解集不能包含重复的组合。 

示例 1：

输入：candidates = [2,3,6,7], target = 7,

所求解集为：

[

  [7],

  [2,2,3]

]

示例 2：

输入：candidates = [2,3,5], target = 8,

所求解集为：

[

  [2,2,2,2],

  [2,3,3],

  [3,5]

]

```bash
    public List<List<Integer>> combinationSum(int[] candidates, int target) {
        List<List<Integer>> res = new ArrayList<>();
        dfs(res, new ArrayList<>(), 0, candidates, target);
        return res;
    }

    private void dfs(List<List<Integer>> res, List<Integer> list, int index, int[] arr, int left) {
        if (left == 0) {
            res.add(new ArrayList<>(list));
            return;
        }
        for (int i = index; i < arr.length; i++) {
            if (arr[i] <= left) {
                list.add(arr[i]);
                // 注意这里递归的index取 i,而不是index，取index的话会遍历回去
                dfs(res, list, i, arr, left - arr[i]);
                list.remove(list.size() - 1);
            }
        }
    }
```
注意点：

* 这里的递归终止条件是left == 0

* dfs递归时的index应该取值i,这样才会遍历到同一个元素。

6、组合

给定两个整数 n 和 k，返回 1 ... n 中所有可能的 k 个数的组合。

示例:

输入: n = 4, k = 2

输出:

[

  [2,4],

  [3,4],

  [2,3],

  [1,2],

  [1,3],

  [1,4],

]

```bash
     public List<List<Integer>> combine(int n, int k) {
        List<List<Integer>> res = new ArrayList<>();
        Deque<Integer> deque = new ArrayDeque<>();
        dfs(res, deque, n, k, 0);
        return res;
    }

    private void dfs(List<List<Integer>> res, Deque<Integer> deque, int n, int k, int index) {
        if (deque.size() == k) {
            res.add(new ArrayList<>(deque));
            return;
        }
        for (int i = index; i < n; i++) {
            deque.addLast(i + 1);
            dfs(res, deque, n, k, i + 1);
            deque.removeLast();
        }
    }
```

注意点：
* 这里需要注意递归终止条件是deque.size() == k

通过上面的分析，我们发现其实递归算法都离不开那个框架，只不过需要根据具体的题目改变一些递归终止条件，递归前要做的事，递归的传值。接下来看一题综合的，起码用到了上面的三道题的套路。

7、活字印刷

你有一套活字字模 tiles，其中每个字模上都刻有一个字母 tiles[i]。返回你可以印出的非空字母序列的数目。

注意：本题中，每个活字字模只能使用一次。

示例 1：

输入："AAB"

输出：8

解释：可能的序列为 "A", "B", "AA", "AB", "BA", "AAB", "ABA", "BAA"。

示例 2：

输入："AAABBC"

输出：188

```bash
    private int count = 0;

    public int numTilePossibilities(String tiles) {
        char[] arr = tiles.toCharArray();
        Arrays.sort(arr);
        boolean[] used = new boolean[arr.length];
        dfs(arr, used, 0);
        return count;
    }

    private void dfs(char[] arr, boolean[] used, int index) {
        if (index != 0) {
            count++;
        }
        for (int i = 0; i < arr.length; i++) {
            if (used[i] || (i > 0 && arr[i] == arr[i - 1] && !used[i - 1])) {
                continue;
            }
            used[i] = true;
            dfs(arr, used, index + 1);
            used[i] = false;
        }
    }
```

注意点：

* 这题其实就是求不重复的，能反过来的（相当于全排列）子集，所以我们用到了三个套路：

* 全排列：used标记数组，并且i 从0开始遍历。

* 不重复：先排序，再相邻元素相同并且上一个元素已经遍历过的剪枝判断

* 子集：只要index > 0 ,每dfs一次，子集的个数就加1，另外子集其实不需要显示写终止条件的。




