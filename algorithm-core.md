# 算法 · 核心知识清单（简单 + 中等）

> 面向：5 年经验 Java 后端工程师，目标是把"套路"固化为肌肉记忆
> 难度范围：LeetCode 简单 ~ 中等（Hard 只涉及经典模型的一层变形）
> 用法：每个套路先背模板，再按「四、往下钻的入口」的题号清单刷，答不出模板就是没掌握

## 一、核心心智模型

1. **刷题的本质是建立"输入特征 → 解法套路"的映射表。** 看到"有序数组 + 查找"→ 二分；看到"连续子串/子数组的最值"→ 滑动窗口；看到"求所有方案"→ 回溯。先识别特征，再套模板，不要从零推导。
2. **先暴力，再优化，最后才谈最优解。** 一个能过 50% 用例的暴力解，比一个卡住写不出的最优解有用得多。暴力解还能暴露边界，为后续优化提供对照。
3. **所有优化手段只有四条：** 空间换时间（哈希/前缀）、双指针（把 O(n²) 降成 O(n)）、单调性（二分/单调栈/单调队列）、去重记忆（DP/记忆化搜索/visited）。想不到优化时挨个问这四个问题。
4. **递归的正确写法是"相信函数定义"。** 不要脑补递归展开过程。只需要保证：基准情形正确、递归调用时问题规模变小、当前层组合出的结果正确。这三条成立，递归就成立。
5. **边界比算法本身更容易错。** 空数组、单元素、全相同、负数、int 溢出、`Integer.MIN_VALUE` 取反 —— 面试挂在这上面的远比挂在思路上的多。每道题写完强制过一遍 checklist（见第五章）。

## 二、必懂机制

### 2.1 复杂度速估（先会估算，再谈优化）

Java 里常见的经验阈值（1 秒时限、OJ 常规机器）：

| 数据规模 | 可接受复杂度 | 说明 |
|---|---|---|
| n ≤ 20 | O(2ⁿ)、O(n!) | 回溯、排列组合。可以剪枝但通常不必 |
| n ≤ 500 | O(n³) | 三层循环、区间 DP |
| n ≤ 5000 | O(n²) | 双层循环、大部分线性 DP |
| n ≤ 10⁵ | O(n log n) | 排序、堆、二分、线段树 |
| n ≤ 10⁶ | O(n) | 双指针、滑动窗口、单调栈/队列、哈希计数 |
| n ≤ 10⁹ | O(log n) / O(1) | 数学、快速幂、二分答案 |

**反过来用**：看到 n = 10⁵ 就知道不能写 O(n²)；看到 n = 20 就大胆写回溯。这一步能直接砍掉一半的错误方向。

**递归爆栈的边界**：Java 默认栈深度约 10000（`-Xss` 控制，通常 512KB~1MB）。链表长度 10⁵ 的递归遍历几乎必爆，改用迭代。

### 2.2 数组 · 双指针

**识别特征**：有序数组上的查找/去重/求和；需要原地修改数组（删除/移动元素）；涉及"回文"。

**两种形态**：
- **对撞指针**（left 从 0、right 从 n-1）：两数之和 II、反转字符串、盛水容器、三数之和、验证回文串
- **快慢指针**（同向，一快一慢）：删除有序数组重复项、移除元素、移动零、环形链表、找中点

```java
// 模板一：对撞 —— 有序数组两数之和（LeetCode 167）
public int[] twoSum(int[] nums, int target) {
    int l = 0, r = nums.length - 1;
    while (l < r) {
        int sum = nums[l] + nums[r];
        if (sum == target) return new int[]{l + 1, r + 1};
        if (sum < target) l++; else r--;
    }
    return new int[]{-1, -1};
}

// 模板二：快慢 —— 原地移除元素（LeetCode 27）
public int removeElement(int[] nums, int val) {
    int slow = 0;                                   // slow 指向下一个待写入的位置
    for (int fast = 0; fast < nums.length; fast++) {
        if (nums[fast] != val) nums[slow++] = nums[fast];
    }
    return slow;                                    // 新长度
}
```

**去重的关键技巧**（三数之和里的两处去重）：
```java
Arrays.sort(nums);
for (int i = 0; i < n - 2; i++) {
    if (i > 0 && nums[i] == nums[i - 1]) continue;   // 外层去重：同值起点只算一次
    ...
    while (l < r && nums[l] == nums[l + 1]) l++;     // 内层去重：命中一组后跳过重复值
    while (l < r && nums[r] == nums[r - 1]) r--;
}
```

### 2.3 哈希表 · 计数

**识别特征**：需要 O(1) 判断某元素是否出现过；统计频率；"两数之和"类；需要前缀和 + 计数。

```java
// 两数之和（LeetCode 1）：一次遍历，边查边插
Map<Integer, Integer> idx = new HashMap<>();
for (int i = 0; i < nums.length; i++) {
    Integer prev = idx.get(target - nums[i]);
    if (prev != null) return new int[]{prev, i};
    idx.put(nums[i], i);                             // 先查后插，避免自己匹配自己
}
```

**注意**：`Map.get()` 返回的是 `Integer`，用 `Integer prev = ...` 接，不要用 `int`（值为 null 时拆箱 NPE）。

**当 key 是自定义对象时**，必须同时重写 `equals` 与 `hashCode`（尤其坐标类 `(x, y)`、单词 key 这类）。偷懒做法：把坐标编码成 `(long) x * n + y`，或对 char 计数用 `int[26]`。

### 2.4 滑动窗口（最重要的一个套路）

**识别特征**：求**连续子串/子数组**的最长/最短/个数；关键词"至多""至少""不含重复字符"。

```java
// 通用模板：找最长（满足条件就扩张，不满足就收缩）
public int longestValidSub(String s) {
    Map<Character, Integer> window = new HashMap<>();
    int l = 0, ans = 0;
    for (int r = 0; r < s.length(); r++) {
        char c = s.charAt(r);
        window.merge(c, 1, Integer::sum);            // 1. 扩大窗口
        while (!valid(window)) {                     // 2. 不满足条件，收缩左边界
            char d = s.charAt(l++);
            window.merge(d, -1, Integer::sum);
            if (window.get(d) == 0) window.remove(d);
        }
        ans = Math.max(ans, r - l + 1);              // 3. 更新答案（此时窗口一定合法）
    }
    return ans;
}
```

**最短窗口**（如最小覆盖子串）把第 3 步挪到 while 内部、用 `Math.min`，并在收缩前更新。

**核心要点**：
- 窗口 `[l, r)` 右闭不确定性统一，建议全程左闭右闭 `[l, r]`
- **更新答案的位置决定了正确性**：求最长 → 在 while 之后更新；求最短 → 在 while 内部（每次收缩前）更新
- 计数类窗口用 `int[128]` 比 HashMap 快一个数量级（字符集固定时）

### 2.5 二分查找

**识别特征**：数组有序（或隐含有序）、求"最小满足条件的值"、答案具有单调性。

**两种模板必须先选一种背死**（推荐第一个，无死循环风险）：

```java
// 模板 A：左闭右开 [l, r)，配套 while (l < r)，永不死循环
public int firstGreatOrEqual(int[] a, int target) {
    int l = 0, r = a.length;                        // 注意 r = n 不是 n-1
    while (l < r) {
        int mid = l + (r - l) / 2;                  // 防溢出，不要写 (l + r) / 2
        if (a[mid] < target) l = mid + 1; else r = mid;
    }
    return l;                                       // 退出时 l == r，是唯一候选位置
}
// 找第一个 >= target：用上面的
// 找最后一个 < target：返回 l - 1
// 找任意一个 == target：先找 lowerBound，再判断 a[l] == target
```

```java
// 模板 B：左闭右闭 [l, r]，while (l <= r)，适合"找某个值是否在"
public int search(int[] a, int target) {
    int l = 0, r = a.length - 1;
    while (l <= r) {
        int mid = l + (r - l) / 2;
        if (a[mid] == target) return mid;
        if (a[mid] < target) l = mid + 1; else r = mid - 1;
    }
    return -1;
}
```

**二分答案（单调性抽象）**：把"求最优值"转成"验证某个值是否可行"。典型题：Koko 吃香蕉、分割数组的最大值、在 D 天内送达、木材加工。
```java
int l = lowerBound, r = upperBound;                 // 答案区间，不是数组下标！
while (l < r) {
    int mid = l + (r - l) / 2;
    if (feasible(mid)) r = mid;                     // 可行 → 尝试更优（更小）
    else l = mid + 1;
}
return l;
```

### 2.6 前缀和与差分

**识别特征**：频繁查询区间和（前缀和）；频繁对区间做批量增减（差分）；"和为 K 的子数组"个数（前缀和 + 哈希）。

```java
// 一维前缀和：pre[i] = a[0..i-1] 的和，pre[0] = 0
int[] pre = new int[n + 1];
for (int i = 0; i < n; i++) pre[i + 1] = pre[i] + a[i];
int rangeSum = pre[r + 1] - pre[l];                 // [l, r] 闭区间

// 和为 K 的子数组个数（LeetCode 560）—— 前缀和 + 计数 Map
Map<Integer, Integer> cnt = new HashMap<>();
cnt.put(0, 1);                                      // 关键：前缀和本身为 K 的情况
int cur = 0, ans = 0;
for (int x : nums) {
    cur += x;
    ans += cnt.getOrDefault(cur - K, 0);            // 之前有没有 pre == cur - K
    cnt.merge(cur, 1, Integer::sum);
}

// 差分：对 [l, r] 加 v 变成 O(1)，最后还原
int[] diff = new int[n + 1];
diff[l] += v; diff[r + 1] -= v;
// 还原：pre[i] = pre[i-1] + diff[i]
```

二维前缀和：`pre[i][j] = pre[i-1][j] + pre[i][j-1] - pre[i-1][j-1] + a[i-1][j-1]`。

### 2.7 栈 · 单调栈

**识别特征**：需要找某元素左边/右边第一个更大（更小）的元素；"柱状图最大矩形"；表达式求值；括号匹配。

```java
// 单调栈：找每个元素右边第一个更大元素的下标（LeetCode 739 每日温度）
int[] ans = new int[n];
Deque<Integer> stack = new ArrayDeque<>();          // 存下标而不是值，方便算距离
for (int i = n - 1; i >= 0; i--) {                  // 倒序遍历 → 找"右边第一个"
    while (!stack.isEmpty() && nums[stack.peek()] <= nums[i]) stack.pop();  // 比当前小的都弹出
    ans[i] = stack.isEmpty() ? -1 : stack.peek() - i;
    stack.push(i);
}
```
正序遍历找"左边第一个更小"，把 `<=` 换成 `>=`，循环方向反过来即可。

**必须用 `ArrayDeque` 而不是 `Stack`**：`java.util.Stack` 继承 `Vector`，所有方法 `synchronized`，且 API 混乱（`push/peek/pop` 混着 `add/element/remove`）。

### 2.8 堆 · TopK

**识别特征**："第 K 大/小""前 K 个高频""合并 K 个有序链表""数据流中位数"。

```java
// 模板：求前 K 个高频元素（LeetCode 347）
Map<Integer, Integer> cnt = new HashMap<>();
for (int x : nums) cnt.merge(x, 1, Integer::sum);

// 小顶堆 + 固定大小 K：进来一个比堆顶大的就替换
PriorityQueue<int[]> heap = new PriorityQueue<>((a, b) -> a[1] - b[1]);   // 按频次升序
for (var e : cnt.entrySet()) {
    heap.offer(new int[]{e.getKey(), e.getValue()});
    if (heap.size() > k) heap.poll();               // 弹掉当前最小的，剩下的就是最大的 K 个
}
```
**记住一句话**：求第 K **大**用**小**顶堆（堆里留 K 个候选），求第 K **小**用**大**顶堆。
**排序-safe 的比较器写法**：`(a, b) -> Integer.compare(a[1], b[1])`，直接 `a[1] - b[1]` 在数据极端时溢出。

### 2.9 链表

**识别特征**：几乎所有链表题都可以加一个**虚拟头节点**简化；几乎所有链表题都可以用**快慢指针**。

```java
// 虚拟头节点：统一处理"删除第一个节点"的边界
ListNode dummy = new ListNode(0, head);
ListNode cur = dummy;
while (cur.next != null) { ... }
return dummy.next;

// 反转链表（迭代版，必须默写）
public ListNode reverse(ListNode head) {
    ListNode prev = null, cur = head;
    while (cur != null) {
        ListNode next = cur.next;
        cur.next = prev;
        prev = cur;
        cur = next;
    }
    return prev;                                    // prev 是新头
}

// 快慢指针三件套
// 1) 找中点：fast 走 2 步 slow 走 1 步，fast 到尾时 slow 在中点（偶数时在后半段起点）
// 2) 判环：相遇则有环；相遇后把 slow 放回 head，两者同速再相遇即入口
// 3) 找倒数第 k 个：fast 先走 k 步，然后同步走，fast 到尾时 slow 即目标
```

**常见中档题套路**：合并两个有序链表（迭代 + 递归两种）、两数相加（进位 `carry` 变量）、K 个一组翻转（先数够不够 K）、复制带随机指针的链表（哈希表映射一把梭，或原地穿插拆分）。

### 2.10 二叉树

**递归三部曲**（写每个递归函数前先答这三个问题）：
1. 这个函数**返回什么**（给上层的信息是什么）
2. 这个函数**在当前节点要做什么**
3. **什么时候返回**（空节点怎么办）

```java
// 后序遍历的典型框架（LeetCode 124 最大路径和、543 直径、687 最长同值路径都是这个骨架）
int ans = 0;
public int dfs(TreeNode root) {
    if (root == null) return 0;
    int left = Math.max(0, dfs(root.left));         // 负数子树直接丢弃
    int right = Math.max(0, dfs(root.right));
    ans = Math.max(ans, left + right + root.val);   // 在"当前节点做根"更新全局答案
    return Math.max(left, right) + root.val;        // 向上只能选一条分支
}
```

**必须背的四种迭代遍历（避免递归爆栈，面试常追问）**：
```java
// 中序遍历迭代：栈 + 一路向左
while (cur != null || !stack.isEmpty()) {
    while (cur != null) { stack.push(cur); cur = cur.left; }
    cur = stack.pop();
    visit(cur);
    cur = cur.right;
}
// 层序遍历：队列，按层处理前先记录 size
while (!q.isEmpty()) {
    int size = q.size();                            // 关键：快照当前层节点数
    for (int i = 0; i < size; i++) { TreeNode t = q.poll(); ... }
}
```

**BST 的核心性质**：中序遍历结果**递增**。所有 BST 题都围绕这个展开：验证 BST（中序看是否递增，或用上下界递归）、第 K 小（中序第 K 个）、BST 转双向链表。

**最近公共祖先（LCA）**：
```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;   // 命中自身也算找到
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    if (left != null && right != null) return root;            // 分居两侧 → 当前就是 LCA
    return left != null ? left : right;                        // 都在一侧 → 返回那个非空的
}
```

### 2.11 回溯（求所有方案）

**识别特征**：求"所有组合/排列/子集/路径""有没有一种方案"，而不是求数量或最值。

```java
public void backtrack(List<Integer> path, int start, int[] choices) {
    if (满足结束条件) { res.add(new ArrayList<>(path)); return; }   // 必须 new，否则全是同一个引用
    for (int i = start; i < choices.length; i++) {
        if (i > start && choices[i] == choices[i - 1]) continue;    // 同层去重（先排序才有效）
        path.add(choices[i]);                                       // 做选择
        track[i] = true;
        backtrack(path, i + 1, choices);                            // 递归（注意 i+1 还是 i）
        track[i] = false;                                           // 撤销选择
        path.remove(path.size() - 1);
    }
}
```

**`i + 1` 还是 `i`（决定是否可重复选）**：
- **组合/子集**（元素不重复用，且 `{1,2}` 与 `{2,1}` 算同一个）→ 递归传 `i + 1`
- **排列**（顺序有关）→ 不传起点，改用 `used[]` 数组标记
- **可重复选同一个元素**（如组合总和）→ 递归传 `i`（不是 `i + 1`）

**剪枝**（把 O(2ⁿ) 压到能过的量级）：
```java
for (int i = start; i < candidates.length; i++) {
    if (sum + candidates[i] > target) break;        // 已排序，后面的更大，直接 break
    ...
}
```

**务必记住：`res.add(new ArrayList<>(path))` 必须拷贝。** 这是新手最常犯的错，表现为结果全为空列表或全相同。

### 2.12 BFS / DFS · 网格搜索

**识别特征**：最短路径/最少步数 → **BFS**；连通性/洪水填充/所有可达 → **DFS 或并查集**。

```java
// 网格 DFS 的两种写法，二选一背熟
// 写法一：原地修改 visited（推荐，省空间）
int[][] dirs = {{1,0},{-1,0},{0,1},{0,-1}};
public void dfs(char[][] grid, int i, int j) {
    if (i < 0 || i >= m || j < 0 || j >= n || grid[i][j] != '1') return;  // 越界 + 已访问合在一起
    grid[i][j] = '0';                                                     // 标记已访问
    for (int[] d : dirs) dfs(grid, i + d[0], j + d[1]);
}
// 写法二：boolean[][] seen，不改原数组（原数组还要用时用这个）
```

**BFS 求最短步数**的层级写法：队列里同时入队时带 `step`，或按层 `size` 循环。迷宫类题目要判断 `pq.isEmpty()` 时才返回 -1。

### 2.13 动态规划（中等难度的主战场）

**DP 五步法（每次写 DP 先在草稿纸过一遍）**：
1. `dp[i]` 表示什么（**定义必须精确到"以 i 结尾"还是"前 i 个"**）
2. 递推公式（从哪几个更小的状态转移来）
3. 初始化（哪些 dp 值需要手工给）
4. 遍历顺序（正向还是倒序、先 i 还是先 j —— 由"当前状态依赖谁"决定）
5. 举例验证（手算前 3 个值）

**线性 DP 经典五题**（必须能默写出状态定义）：

| 题 | 状态定义 | 转移 |
|---|---|---|
| 爬楼梯 | `dp[i]` 到第 i 阶的方法数 | `dp[i] = dp[i-1] + dp[i-2]` |
| 打家劫舍 | `dp[i]` 前 i 家最大金额 | `dp[i] = max(dp[i-1], dp[i-2] + nums[i])` |
| 最大子数组和 | `dp[i]` 以 i 结尾的最大连续和 | `dp[i] = max(nums[i], dp[i-1] + nums[i])` |
| 最长递增子序列 | `dp[i]` 以 i 结尾的 LIS 长度 | `dp[i] = max(dp[j]) + 1` for `j < i && nums[j] < nums[i]` |
| 最长公共子序列 | `dp[i][j]` 前 i 与前 j 的 LCS | 相等 `dp[i-1][j-1]+1`，否则 `max(dp[i-1][j], dp[i][j-1])` |

**0-1 背包（一维滚动数组）**：
```java
// dp[j] = 容量 j 能装的最大价值
for (int i = 0; i < n; i++) {
    for (int j = W; j >= weight[i]; j--) {          // 倒序！保证每个物品只用一次
        dp[j] = Math.max(dp[j], dp[j - weight[i]] + value[i]);
    }
}
```
**完全背包**（物品无限）：内层 `j` 改**正序**。这一个字符的差别是面试必考点。
**排列 vs 组合**：求组合数（顺序无关）外层遍历物品、内层遍历容量；求排列数外层遍历容量、内层遍历物品。

**区间 DP**：`dp[i][j]` 表示区间 `[i, j]` 的最优值，三层循环，外层枚举**区间长度**，再枚举起点。典型：最长回文子串、戳气球、石子合并。

**状态机 DP**（买卖股票系列，用几组状态刻画"持有/卖出/冷冻/限制次数"）：定义多个状态并相互转移。
```java
// 含冷冻期/手续费/最多两次 的股票题，状态数扩展即可
int hold = -prices[0], sold = 0, rest = 0;
for (int p : prices) {
    int prevSold = sold;
    sold = hold + p;                                // 今天卖出
    hold = Math.max(hold, rest - p);                // 继续持有 或 今天买入
    rest = Math.max(rest, prevSold);                // 继续休息 或 冷冻结束
}
```

### 2.14 区间问题 · 贪心

**识别特征**：求最少移除多少个区间使得互不重叠、会议室排期、合并/插入区间。
**核心套路**：先按起点（或终点）排序，再单次遍历。

```java
// 无重叠区间（LeetCode 435）：按终点升序，能留就留 —— 经典贪心
Arrays.sort(intervals, (a, b) -> Integer.compare(a[1], b[1]));
int count = 0, end = Integer.MIN_VALUE;
for (int[] iv : intervals) {
    if (iv[0] >= end) { count++; end = iv[1]; }     // 不重叠，保留，更新右边界
}
return intervals.length - count;
```
**贪心的正确性前提**：能用贪心必须能证明"局部最优能推到全局最优"。常用的两种证法：交换论证、数学归纳。面试时至少要说出思路。

### 2.15 并查集

**识别特征**：连通性判断、动态加边、等价关系分组、最小生成树 Kruskal。

```java
class UnionFind {
    int[] parent, size;
    int count;                                      // 连通分量个数，很多题直接要这个
    UnionFind(int n) {
        parent = new int[n]; size = new int[n]; count = n;
        for (int i = 0; i < n; i++) { parent[i] = i; size[i] = 1; }
    }
    int find(int x) {
        while (x != parent[x]) { parent[x] = parent[parent[x]]; x = parent[x]; }  // 路径压缩
        return x;
    }
    boolean union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return false;                 // 已连通，本题可用于判环
        if (size[ra] < size[rb]) { int t = ra; ra = rb; rb = t; }   // 小树挂大树
        parent[rb] = ra; size[ra] += size[rb]; count--;
        return true;
    }
}
```
务必两个优化都做（路径压缩 + 按秩合并），复杂度才是反阿克曼级别近似 O(1)。

### 2.16 拓扑排序

**识别特征**：依赖关系、先后约束、判断 DAG、课程表系列。

```java
// Kahn 算法：入度表 + 队列。若最终出队数 < 节点数，说明有环
Queue<Integer> q = new ArrayDeque<>();
for (int i = 0; i < n; i++) if (indeg[i] == 0) q.offer(i);
int visited = 0;
while (!q.isEmpty()) {
    int cur = q.poll(); visited++;
    for (int next : graph.get(cur)) if (--indeg[next] == 0) q.offer(next);
}
return visited == n;                                // false 表示有环
```

### 2.17 位运算

常考的几条：

| 技巧 | 写法 | 用途 |
|---|---|---|
| 判断第 i 位 | `(n >> i) & 1` | 状态压缩 DP、子集枚举 |
| 消去最低位的 1 | `n &= (n - 1)` | 数 1 的个数（Brian Kernighan 法，比逐位快）；判断是否为 2 的幂：`(n & (n-1)) == 0` |
| 取最低位的 1 | `n & -n` | lowbit，树状数组 |
| 异或消重 | `a ^ a == 0` | 只出现一次的数字（全员异或） |
| 无临时变量交换 | `a ^= b; b ^= a; a ^= b;` | 了解即可，别在生产用 |

### 2.18 排序（手写与选择）

| 算法 | 平均 | 最坏 | 稳定 | 适用 |
|---|---|---|---|---|
| 快排 | O(n log n) | O(n²) | ❌ | 通用最快，常数小 |
| 归并 | O(n log n) | O(n log n) | ✅ | 需要稳定性、链表排序、求逆序对个数 |
| 堆排 | O(n log n) | O(n log n) | ❌ | 原地 + 无最坏退化，但常数大、cache 不友好 |
| 计数/桶 | O(n + k) | — | ✅ | 值域有限（如分数 0-100） |

**JDK 的实现要知道**：`Arrays.sort(int[])` 用**双轴快排**（Dual-Pivot Quicksort），`Arrays.sort(Object[])` 用 **TimSort**（归并 + 插入的混合）。前者不稳定，后者稳定。

```java
// 归并求逆序对个数 —— 归并排序唯一在生产真有用的副产品
long mergeSort(int[] a, int l, int r) {
    if (l >= r) return 0;
    int mid = l + (r - l) / 2;
    long cnt = mergeSort(a, l, mid) + mergeSort(a, mid + 1, r);
    // 合并时：右半的 mid+1+j 入队，产生 mid-i+1 个逆序对
    ...
}
```

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| 二分陷入死循环 | 左右分支的更新没跨过 mid（写成 `r = mid` 却配 `while (l <= r)`） | 固定一套模板：`[l, r)` 配 `while (l < r)` 配 `r = mid`；或 `[l, r]` 配 `while (l <= r)` 配 `r = mid - 1`。两套不要混用 |
| `mid = (l + r) / 2` 溢出 | l 和 r 都是接近 `Integer.MAX_VALUE` 时求和溢出成负数 | 一律写 `l + (r - l) / 2` |
| 回溯结果全是空列表或全相同 | `res.add(path)` 加的是同一个引用，后续 remove 清掉了 | `res.add(new ArrayList<>(path))` |
| `int[][] dp` 用 `Infinity` 初始化后加减溢出 | `Integer.MAX_VALUE + 1` 变成 `Integer.MIN_VALUE` | 用 `Integer.MAX_VALUE / 2` 做"无穷大"，或先判 ub 是否可达再算 |
| 用 `Stack` 类导致莫名慢 | `java.util.Stack` 继承 `Vector`，方法带锁 | 用 `ArrayDeque`（`push/pop/peek` API 一致） |
| `PriorityQueue` 默认弹出最小的，与预期相反 | 默认自然序是**小顶堆** | 求 TopK 大元素用小顶堆留 K 个；或用 `Comparator.reverseOrder()` / `(a,b) -> b - a` |
| 比较器里写 `a - b` 结果错乱 | 两数差超过 int 范围溢出 | 写 `Integer.compare(a, b)` |
| `Arrays.asList(int[])` 只有一个元素且是 `int[]` | 基本类型数组不能作为泛型参数，整个数组被当成一个元素 | 用 `Integer[]`；或 `Arrays.stream(arr).boxed().collect(toList())` |
| 链表题写了半天 `head` 被改导致丢链 | 没用虚拟头节点，删除头节点时逻辑分支爆炸 | 一律加 `ListNode dummy = new ListNode(0, head)` |
| 递归爆 StackOverflowError | 链长 10⁵ 级别用了递归 | 改迭代；或确认无法改再 `-Xss` 调栈（治标） |
| `Map` 计数时 `getOrDefault` 之后忘了 `put` | 只查不写，下一轮计数丢失 | 用 `merge(key, 1, Integer::sum)` |
| `list.remove(i)` 在正序遍历时跳过元素 | 删除后下标前移 | 倒序遍历删除；或 `removeIf` |
| 滑动窗口答案永远不对 | 更新答案的时机不对（求最长在 while 里更新了） | 求最长 → while 之后；求最短 → while 内部 |
| 字符串拼接导致超时 | `String` 不可变，`+=` 循环里生成大量对象 | 用 `StringBuilder` |
| 输入量大时用 `Scanner` 超时 | `Scanner` 有正则解析开销 | 用 `BufferedReader` + `StringTokenizer`，或手写 `FastScanner`（`byte[]` 缓冲区） |
| 完全背包/0-1 背包结果一样错 | 内层遍历方向没反过来 | 0-1 倒序，完全正序 |
| 求组合数与排列数搞混答案翻倍 | 内外层遍历顺序反了 | 组合：外层物品；排列：外层容量 |
| 负数取模得到负结果 | Java 的 `%` 是带余除法，符号随被除数 | `(a % m + m) % m` |
| `Integer.MIN_VALUE` 取反溢出 | `-Integer.MIN_VALUE == Integer.MIN_VALUE` | 转 long 再取反 |
| 递归里重复 `new` 对象导致 GC 压力大 | 每层循环都新建 List/Map | 复用容器 + 回溯时撤销；或传参进去 |

## 四、往下钻的入口

### 分层刷题清单（按套路，先刷"打钩必做"）

| 套路 | 必做（刷透模板） | 进阶 |
|---|---|---|
| 数组双指针 | 27 移除元素、26 删除重复项、283 移动零、167 两数之和 II、344 反转字符串 | 11 盛水容器、15 三数之和、18 四数之和、42 接雨水 |
| 哈希表 | 1 两数之和、49 字母异位词分组、217 存在重复元素、242 有效字母异位词 | 128 最长连续序列、560 和为 K 的子数组 |
| 滑动窗口 | 3 无重复字符最长子串、209 长度最小子数组、438 找到所有异位词、567 字符串的排列 | 76 最小覆盖子串、239 滑动窗口最大值（单调队列） |
| 二分 | 704 二分查找、35 搜索插入位置、34 在排序数组中查找范围、69 x 的平方根 | 33/81 旋转数组搜索、153/154 旋转数组最小值、875 爱吃香蕉（二分答案）、410 分割数组最大值 |
| 栈 / 单调栈 | 20 有效括号、155 最小栈、739 每日温度 | 84 柱状图最大矩形、42 接雨水、496/503 下一个更大元素 |
| 堆 | 215 数组第 K 大、347 前 K 个高频、703 数据流第 K 大 | 23 合并 K 个有序链表、295 数据流中位数（双堆） |
| 链表 | 206 反转、21 合并两个有序、141/142 环形链表、876 中点 | 19 删除倒数第 N 个、24 两两交换、25 K 个一组翻转、138 随机指针复制、148 排序链表 |
| 二叉树 | 94/144/145 三种遍历、102 层序、104 最大深度、226 翻转、101 对称 | 236/235 最近公共祖先、124 最大路径和、543 直径、105/106 前序中序构造、98 验证 BST、230 第 K 小 |
| 回溯 | 78 子集、77 组合、46 全排列、39 组合总和、22 括号生成 | 40 组合总和 II、90 子集 II、47 全排列 II、51 N 皇后、79 单词搜索 |
| DFS / BFS | 200 岛屿数量、695 最大岛屿面积、733 图像渲染、994 腐烂的橘子 | 130 被围绕的区域、417 太平洋大西洋水流、207/210 课程表（拓扑） |
| DP | 70 爬楼梯、198 打家劫舍、53 最大子数组、300 最长递增子序列、1143 最长公共子序列、322 零钱兑换、416 分割等和子集、62/63 不同路径 | 72 编辑距离、518 零钱兑换 II、139 单词拆分、152 乘积最大子数组、714/309 股票含手续费冷冻期、718 最长重复子数组、647 回文子串数、5 最长回文子串 |
| 贪心 | 455 分发饼干、122 买卖股票 II、435 无重叠区间、452 用最少数量箭引爆气球 | 406 根据身高重建队列、763 划分字母区间 |
| 并查集 | 547 省份数量、200 岛屿数量（并查集版）、990 等式方程可满足性 | 684 冗余连接、130 被围绕的区域 |
| 位运算 | 136 只出现一次的数字、191 位 1 的个数、231 2 的幂、338 比特位计数 | 78 子集（位掩码解法）、190 颠倒二进制位 |

### 建议的四阶段节奏

| 阶段 | 内容 | 目标 |
|---|---|---|
| 第 1 周 | 数组 + 哈希 + 双指针 + 二分，共约 30 题 | 建立"看到特征选套路"的条件反射，学会按专题分类刷题而非随机刷 |
| 第 2 周 | 链表 + 栈队列 + 二叉树递归与迭代遍历，约 25 题 | 指针 handling 不再出错；递归三部曲成为习惯 |
| 第 3 周 | 回溯 + DFS/BFS + 滑动窗口，约 25 题 | 能够独立写出剪枝与 visited 的正确处理 |
| 第 4 周 | DP 全谱 + 贪心 + 并查集 + 位运算，约 30 题 | 五步法能在草稿纸上快速推完，中等 DP 题 15 分钟出解 |

**每道题的固定动作**：5 分钟没思路就看题解 → 看懂后**关掉网页重写一遍** → 隔两天重做（隔天效应比当天连刷三遍有效得多）。

### 参考资料（具体到章节）

- 《算法（第 4 版）》Sedgewick —— 第 1 章（并查集）、2.1-2.4（排序）、2.5（优先队列）、3.2（二叉搜索树）。图论部分可略
- labuladong 的算法笔记 —— "动态规划解题框架""回溯算法套路""滑动窗口万能模板"三节，与本文的模板思路同源，可作为交叉验证
- LeetCode 官方题库的「学习计划」—— `LeetCode 75` / `LeetCode 热题 HOT 100`，覆盖 80% 面试面
- OI Wiki —— 查"复杂度分析""各种 DP 模型定义"的权威速查
- Carl 的《代码随想录》—— 按专题分層，适合 Java 选手，尤其二叉树与 DP 部分

### Java 选手专属准备

```java
// 1. ACM 模式的输入模板（很多国内厂面试要求自己读输入）
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
StringTokenizer st = new StringTokenizer(br.readLine());
int n = Integer.parseInt(st.nextToken());

// 2. 常用输出：不要每条 println，攒到 StringBuilder 一次性输出（数据量大时差 10 倍）
StringBuilder sb = new StringBuilder();
sb.append(ans).append('\n');
System.out.print(sb);

// 3. 别忘了这些工具（写对了能省一半代码）
Arrays.sort(arr);                       // 基本类型用双轴快排
Arrays.sort(arr, l, r);                 // 左闭右开的部分排序
Arrays.binarySearch(arr, key);          // 注意返回值是下标或 -(插入点) - 1
Collections.reverse(list);
Arrays.fill(dp, Integer.MAX_VALUE / 2); // 初始化无穷大
new ArrayDeque<>();                     // 栈和队列都用它
```

## 五、达标自检

**套路识别**（说出每个描述对应的套路，并写出模板）
1. 求「不含重复字符的最长子串」—— 用什么套路？答案在哪里更新？
2. 求「和为 K 的子数组个数」，数组里有负数还能用滑动窗口吗？改用哪个套路？
3. 求「数据流中的第 K 大元素」，堆应该装什么、装多大？为什么大小是固定的？
4. 「从 n 个数里求第 K 大的数」除了堆排序 Road之外还有什么 O(n) 解法？（快排 partition）
5. 「每天的温度，求至少要等几天才有更暖的一天」—— 单调栈从哪个方向遍历？栈内存什么？

**边界与实现**
6. 写出不溢出、不死循环的二分模板，并说明 `while` 条件、`mid` 计算、`l/r` 更新三者的搭配为什么必须一致。
7. 0-1 背包和完全背包的一维写法，代码上只差哪一个字符？为什么这个字符决定了正确性？
8. 「组合」和「排列」的回溯写法，除了 `used[]` 还有什么关键区别？如何实现不重复地为{1,1,2}去重？
9. 二叉树递归函数 `diameterOfBinaryTree` 里，为什么全局答案要用成员变量而不是返回值？
10. 反转链表迭代过程中，`next` 临时变量为什么必须在修改 `cur.next` 之前保存？

**取舍与证明**
11. 什么情况下该用归并排序而不是快排？说出至少两个理由。
12. 「求逆序对个数」这个问题为什么刚好可以在归并的过程中顺手算出？它的核心依据是什么？
13. 拓扑排序的两个解法（Kahn 与 DFS 逆后序）分别如何实现？如何判断有环？
14. 并查集的两个优化各自解决什么问题？只做路径压缩不做按秩合并，最坏复杂度会退化到什么？
15. 什么情况下贪心是错的，必须用 DP？举一个具体的反例。

**综合**
16. 「买卖股票的最佳时机」一共有 6 道题（121/122/123/188/309/714），它们的 DP 状态分别是怎么定义的？能否统一到一个框架？
17. 「打家劫舍」的两个变种（环形街区、树形街区）分别怎么调整状态定义？
18. 「编辑距离」的 `dp[i][j]` 定义是什么？三个转移操作分别对应什么？空间可以优化到 O(n) 吗？
19. 「最长回文子串」有中心扩散、DP、Manacher 三种写法，各自的复杂度和适用场景是什么？
20. 如果给你一道完全没见过的新题，你的解题流程是什么？（说清楚每一步的产出物和放弃条件）
