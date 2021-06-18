---
title: 算法的奥秘-回溯（3）
tags:
  - 回溯
categories:
  - 数据结构与算法
date: 2021-06-17 20:07:58
---

&ensp;&ensp;今天打算分享下回溯算法中的另一种题型--是否存在***，这种题型一般都不用走完整个dfs过程，如果存在就可以直接返回了，这样相当于剪枝了，可以很大程度提升算法执行时间。那还等什么，毁灭吧，赶紧的！

    
1、单词搜素

给定一个二维网格和一个单词，找出该单词是否存在于网格中。

单词必须按照字母顺序，通过相邻的单元格内的字母构成，其中“相邻”单元格是那些水平相邻或垂直相邻的单元格。同一个单元格内的字母不允许被重复使用。

<!-- more -->

示例:

board =

[

  ['A','B','C','E'],

  ['S','F','C','S'],

  ['A','D','E','E']

]

给定 word = "ABCCED", 返回 true

给定 word = "SEE", 返回 true

给定 word = "ABCB", 返回 false

提示：

* board 和 word 中只包含大写和小写英文字母。

* 1 <= board.length <= 200

* 1 <= board[i].length <= 200

* 1 <= word.length <= 10^3

代码：
```bash

    public boolean exist(char[][] board, String word) {
        for (int i = 0; i < board.length; i++) {
            for (int j = 0; j < board[i].length; j++) {
                boolean[][] used = new boolean[board.length][board[i].length];
                if (dfs(i, j, board, word, 0, used)) {
                    return true;
                }
            }
        }
        return false;
    }

    private boolean dfs(int row, int col, char[][] board, String word, int index, boolean[][] used) {
        if (index == word.length()) {
            return true;
        }
        if (row < 0 || row >= board.length || col < 0 || col >= board[row].length || board[row][col] != word.charAt(index) || used[row][col]) {
            return false;
        }
        used[row][col] = true;

        boolean res = dfs(row + 1, col, board, word, index + 1, used) ||
                dfs(row, col + 1, board, word, index + 1, used) ||
                dfs(row - 1, col, board, word, index + 1, used) ||
                dfs(row, col - 1, board, word, index + 1, used);
        used[row][col] = false;
        return res;
    }
```

注意点：

1. 可以看到连续四个dfs那里，用的是短路或，这样保证了只要存在true的结果，其他的dfs也就不用递归进去了，也就达到了剪枝的效果。

2. 注意这里dfs进去前没有判断边界条件，而是放在了进去后再进行判断，有时候知道怎么做就是写不出来，这点也要注意。

<br>
2、累加数

累加数是一个字符串，组成它的数字可以形成累加序列。

一个有效的累加序列必须至少包含 3 个数。除了最开始的两个数以外，字符串中的其他数都等于它之前两个数相加的和。

给定一个只包含数字 '0'-'9' 的字符串，编写一个算法来判断给定输入是否是累加数。

说明: 累加序列里的数不会以 0 开头，所以不会出现 1, 2, 03 或者 1, 02, 3 的情况。

示例 1:

输入: "112358"

输出: true 

解释: 累加序列为: 1, 1, 2, 3, 5, 8 。1 + 1 = 2, 1 + 2 = 3, 2 + 3 = 5, 3 + 5 = 8

示例 2:

输入: "199100199"

输出: true 

解释: 累加序列为: 1, 99, 100, 199。1 + 99 = 100, 99 + 100 = 199

```bash

    private String s;

    private int n;

    public boolean isAdditiveNumber(String num) {
        s = num;
        n = s.length();
        return dfs(0, 0, 0, 0);
    }

    /**
     *
     * @param index 下标
     * @param sum 前两个数之和
     * @param previous 前一个数
     * @param count 已经添加的个数
     * @return
     */
    private boolean dfs(int index, long sum, long previous, int count) {
        if (index == n) {
            if (count >= 3) {
                return true;
            }
        }
        // value值用于累加值
        long value = 0;
        for (int i = index; i < n; i++) {
            // 第一个数是0，而且当前value大于一位，直接break
            if (i > index && s.charAt(index) == '0') {
                break;
            }
            value = value * 10 + s.charAt(i) - '0';
            if (count >= 2) {
                if (value < sum) {
                    // 继续累加value
                    continue;
                } else if (value > sum) {
                    // 累加value无意义，直接break
                    break;
                }
            }
            // 忽略false结果，妙！
            if (dfs(i + 1, previous + value, value, count + 1)) {
                return true;
            }
        }
        return false;
    }
```

注意点：
* 这里dfs进去直接忽略了false结果，试想这里如果不忽略false，也返回一个false结果，那么只要一次尝试不成功就不会继续尝试了,这样就没有达到穷举的效果，而一点返回true，说明我们这种字符串分割方法就是我们要的结果，因此可以直接返回true，一层接着一层，最终最外层返回true，给到调用方，结束递归。