# 二叉搜索树中第 K 小的元素怎么找？

对应 LeetCode 230。

## 面试回答

BST 的中序遍历天然是升序，所以第 K 小就是中序遍历访问到的第 K 个节点。

最直接的写法可以先把中序结果放进 List，然后取：

```java
return res.get(k - 1);
```

注意：题目里的 k 从 1 开始，而 List 下标从 0 开始。

## 一个容易写错的点

错误写法：

```java
return res.toString().charAt(k);
```

`res.toString()` 得到的是类似：

```text
[1, 2, 3, 4]
```

`charAt(k)` 取的是字符串中的第 k 个字符，不是第 k 个元素。

正确应该是：

```java
return res.get(k - 1);
```

## 更推荐的版本：不用 List

不需要把整棵树的中序结果都存下来，直接维护一个访问计数器即可。

```java
class Solution {
    int count = 0;
    int ans;

    public int kthSmallest(TreeNode root, int k) {
        dfs(root, k);
        return ans;
    }

    public void dfs(TreeNode root, int k) {
        if (root == null) return;

        dfs(root.left, k);

        count++;

        if (count == k) {
            ans = root.val;
            return;
        }

        dfs(root.right, k);
    }
}
```

## 题型识别

看到：

- BST
- 第 K 小
- 从小到大

第一反应：

```text
中序遍历：left -> root -> right
```

因为 BST 中序遍历结果天然升序。
