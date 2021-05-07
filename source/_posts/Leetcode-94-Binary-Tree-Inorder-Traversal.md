---
title: Leetcode 94 Binary Tree Inorder Traversal
date: 2021-05-07 07:50:58
tags:
    - go
---
個人覺得這題應該是easy，不過不重要。今天在710往永寧站的車上想說來試一下在手機瀏覽器leetcode用Go寫一題練習看看。感想是自己Go程式還是寫的不夠多，感覺還沒出來。

``` go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func inorder(root *TreeNode, res *[]int) {
    if root == nil {
        return
    }
    inorder(root.Left, res)
    *res = append(*res, root.Val)
    inorder(root.Right, res)
}

func inorderTraversal(root *TreeNode) []int {
    var res []int
    inorder(root, &res)
    return res
}
```