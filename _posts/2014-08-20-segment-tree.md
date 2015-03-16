---
layout: post
title: "Segment Tree"
description: ""
category: data structure
tags: []
---
{% include JB/setup %}


# Introduction

Segment tree is a tree data structure for storing intervals, or segments. It allows querying which of the stored segments contain a given point. It is, in principle, a static structure; that is, its content cannot be modified once the structure is built. A similar data structure is the interval tree.

A segment tree for a set of n intervals uses **O(n log n)** storage and can be built in **O(n log n)** time. Segment trees support searching for all the intervals that contain a query point in **O(log n + k)**, k being the number of retrieved intervals or segments.

**Reference**

* [http://en.wikipedia.org/wiki/Segment_tree](http://en.wikipedia.org/wiki/Segment_tree)

# Application

## [Balanced Lineup](http://poj.org/problem?id=3264)

**Description**

For the daily milking, Farmer John's N cows (1 <= N <= 50,000) always line up in the same order. One day Farmer John decides to organize a game of Ultimate Frisbee with some of the cows. To keep things simple, he will take a contiguous range of cows from the milking lineup to play the game. However, for all the cows to have fun they should not differ too much in height.

Farmer John has made a list of Q (1 <= Q <= 200,000) potential groups of cows and their heights (1 <= height <= 1,000,000). For each group, he wants your help to determine the difference in height between the shortest and the tallest cow in the group.

**Input**

Line 1: Two space-separated integers, N and Q.

Lines 2..N+1: Line i+1 contains a single integer that is the height of cow i

Lines N+2..N+Q+1: Two integers A and B (1 <= A <= B <= N), representing the range of cows from A to B inclusive.

**Output**

Lines 1..Q: Each line contains a single integer that is a response to a reply and indicates the difference in height between the tallest and shortest cow in the range.

**Sample Input**

6 3

1

7

3

4

2

5

1 5

4 6

2 2

**Sample Output**

6

3

0

**Code**

Using array to implement segment tree.

```cpp
class SegmentTree
{
private:
    // tree node
    struct TreeNode {
        int left; // segment's left point
        int right; // segment's right point

        // here we can record anything specific to a problem
        // i.e.: sum, max or min element, and so on
        int minEle; // min element in the scope[left...right]
        int maxEle; // max element in the scope[left...right]

        TreeNode(): left(0), right(0), minEle(0), maxEle(0) {}
    };

    vector<TreeNode> treeNode; // tree node set, treeNode[1] is root

    // build perfect binary tree
    // using array(treeNode) to store it
    void build(vector<int> &data, int t, int left, int right)
    {
        // set segment's start and end point
        treeNode[t].left = left;
        treeNode[t].right = right;

        if (left == right) {
            treeNode[t].minEle = treeNode[t].maxEle = data[left];
            return;
        }

        // build left and right tree recursively
        int mid = left + ((right - left) >> 1);
        int leftRoot = t * 2;
        int rightRoot = leftRoot + 1;
        build(data, leftRoot, left, mid);
        build(data, rightRoot, mid + 1, right);

        // update info
        treeNode[t].minEle = min(treeNode[leftRoot].minEle, treeNode[rightRoot].minEle);
        treeNode[t].maxEle = max(treeNode[leftRoot].maxEle, treeNode[rightRoot].maxEle);
    }

    void query(int t, int left, int right, int &lower, int &upper)
    {
        if (treeNode[t].left == left && treeNode[t].right == right) {
            lower = min(lower, treeNode[t].minEle);
            upper = max(upper, treeNode[t].maxEle);
            return;
        }

        int mid = treeNode[t].left + ((treeNode[t].right - treeNode[t].left) >> 1);
        if (left > mid) { // search right child
            query(2 * t + 1, left, right, lower, upper);
        } else if (right <= mid) { // search left child
            query(2 * t, left, right, lower, upper);
        } else { // search both
            query(2 * t, left, mid, lower, upper);
            query(2 * t + 1, mid + 1, right, lower, upper);
        }
    }

public:
    // 2 * n is enough to store the tree with n leaves
    SegmentTree(vector<int> &v) : treeNode(v.size() * 2)
    {
        build(v, 1, 1, v.size() - 1);
    }

    int maxDiff(int left, int right)
    {
        int maxData = INT_MIN, minData = INT_MAX;
        query(1, left, right, minData, maxData);
        return maxData - minData;
    }
};

int main(int argc, char *argv[])
{
#ifndef ONLINE_JUDGE
    freopen("input", "r", stdin);
    // freopen("output","w",stdout);
#endif

    int n, q;
    int a, b;
    cin >> n >> q;
    vector<int> v(n+1);

    for (int i = 1; i <= n; i++) cin >> v[i];
    SegmentTree sol(v);

    while (cin >> a >> b) {
        cout << sol.maxDiff(a, b) << endl;
    }

    return 0;
}
```

Using chain list to implement segment tree.

```cpp
class SegmentTree
{
private:
    // tree node
    struct TreeNode {
        TreeNode *lChild, *rChild;

        int left; // segment's left point
        int right; // segment's right point

        // here we can recode anything specific to a problem
        // i.e.: sum, max or min element, and so on
        int minEle; // min element in the scope[left...right]
        int maxEle; // max element in the scope[left...right]
        TreeNode(const int l = 0, const int r = 0):
            lChild(nullptr), rChild(nullptr), left(l), right(r), minEle(0), maxEle(0) {}
    };

    TreeNode *root;

    // build binary tree
    TreeNode* build(vector<int> &data, int left, int right)
    {
        // set segment's start and end point
        TreeNode *root = new TreeNode(left, right);
        if (root == nullptr) throw runtime_error("no memory");

        if (left == right) {
            root->minEle = root->maxEle = data[left];
            return root;
        }

        // build left and right tree recursively
        int mid = left + ((right - left) >> 1);
        root->lChild = build(data, left, mid);
        root->rChild = build(data, mid + 1, right);

        // update info
        root->minEle = min(root->lChild->minEle, root->rChild->minEle);
        root->maxEle = max(root->lChild->maxEle, root->rChild->maxEle);

        return root;
    }

    void query(TreeNode *t, int left, int right, int &lower, int &upper)
    {
        if (t == nullptr) return; // not happen
        if (t->left == left && t->right == right) {
            lower = min(lower, t->minEle);
            upper = max(upper, t->maxEle);
            return;
        }

        int mid = t->left + ((t->right - t->left) >> 1);
        if (left > mid) { // search right child
            query(t->rChild, left, right, lower, upper);
        } else if (right <= mid) { // search left child
            query(t->lChild, left, right, lower, upper);
        } else { // search both
            query(t->lChild, left, mid, lower, upper);
            query(t->rChild, mid + 1, right, lower, upper);
        }
    }

public:
    SegmentTree(vector<int> &v)
    {
        root = build(v, 1, v.size() - 1);
    }

    int maxDiff(int left, int right)
    {
        int maxData = INT_MIN, minData = INT_MAX;
        query(root, left, right, minData, maxData);
        return maxData - minData;
    }
};
```