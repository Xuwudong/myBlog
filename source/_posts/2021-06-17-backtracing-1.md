---
title: 算法的奥秘-回溯（1）
tags:
  - 回溯
categories:
  - 数据结构与算法
date: 2021-06-17 19:51:18
---

&ensp;&ensp;今天分享一道简单的回溯算法题（在我看来并不简单！），毁灭吧，赶紧的！
题目：二进制手表
描述：二进制手表顶部有 4 个 LED 代表 小时（0-11），底部的 6 个 LED 代表 分钟（0-59）。

每个 LED 代表一个 0 或 1，最低位在右侧。

<!-- more -->

![avatar](/images/backtracing/二进制手表.webp)

例如，上面的二进制手表读取 “3:25”。<br>
给定一个非负整数 n 代表当前 LED 亮着的数量，返回所有可能的时间。

示例：
输入: n = 1<br>
返回: ["1:00", "2:00", "4:00", "8:00", "0:01", "0:02", "0:04", "0:08", "0:16", "0:32"]<br>
提示：
* 输出的顺序没有要求。
* 小时不会以零开头，比如 “01:00” 是不允许的，应为 “1:00”。
* 分钟必须由两位数组成，可能会以零开头，比如 “10:2” 是无效的，应为 “10:02”。
* 超过表示范围（小时 0-11，分钟 0-59）的数据将会被舍弃，也就是说不会出现 "13:00", "0:61" 等时间。

分析：题目需要返回所有可能的时间，可以对手表建模，时钟定义一个数组hours,包含4盏灯，代表数值分别为【1,2，4,8】，分钟定义一个数组mins，包含6盏灯，代表数值分别为【1,2，4,8，16,32】。对于输入n,需要从hours中取出i盏灯，mins中取出n-j盏灯，并将合并的结果存入list。这就是子集的变形，很容易想到用回溯求解，但是做起来还是有一些注意点值得关注，直接看代码！

```bash
    int[] hours = new int[]{1, 2, 4, 8};
    int[] mins = new int[]{1, 2, 4, 8, 16, 32};

    public List<String> readBinaryWatch(int num) {
        List<String> res = new ArrayList<>();
        dfs(res, num, 0, 0, 0, 0);
        return new ArrayList<>(res);
    }


    /**
     * dfs
     *
     * @param res    结果集
     * @param num    灯的个数
     * @param indexH hours的索引
     * @param indexM mins的索引
     * @param hour   当前hour的值
     * @param min    当前min的值
     */
    private void dfs(List<String> res, int num, int indexH, int indexM, int hour, int min) {
        if (num == 0) {
            // 递归结束条件为num == 0,注意不是indexH + indexM == num, 因为indexH表示的是hours的索引。
            String minStr = (0 <= min && min <= 9) ? "0" + min : min + "";
            String str = hour + ":" + minStr;
            res.add(str);
            return;
        }
        // 这种循环结构就是求hours的子集，此题就是求hours求指定长度的各个子集的和
        for (int i = indexH; i < hours.length; i++) {
            if (hour + hours[i] >= 12) {
                // 如果大于12，后面的循环不用看了，直接break
                break;
            }
            dfs(res, num - 1, i + 1, indexM, hour + hours[i], min);
            // dfs后不用回溯是因为dfs前并没有改变hour的值
        }
        for (int i = indexM; i < mins.length; i++) {
            if (min + mins[i] >= 60) {
                // 如果大于60，后面的循环不用看了，直接break
                break;
            }
            // 注意这里递归到mins时，并不需要再次递归到hours,不然结果会重复,所以indexH填4
            dfs(res, num - 1, 4, i + 1, hour, min + mins[i]);
            // dfs后不用回溯是因为dfs前并没有改变min的值
        }
    }
```

注意点：
* 题目递归结束的条件是num == 0，而不是indexH + indexM == num。
* 回溯算法一般都可以通过剪枝减少递归，具体看代码中的break片段。
* 此题有两个求子集的过程，所以有两个for循环，注意下面一个循环中的dfs，indexH的取值不能传当前值了（如果传了当前值，递归进去又会走一遍hours循环，这样结果就有重复值了），所以直接取4。
* 回溯算法一般都需要在回溯时做“还原”操作，但是这里因为dfs前并没有改正hour和min的值，所以dfs后不用“还原”。


这是自己的一些想法，如果你有更好的想法欢迎留言！

