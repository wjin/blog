---
layout: post
title: "Union Find Set"
description: ""
category: data structure
tags: []
---
{% include JB/setup %}

# Introduction

Union-Find is a data structure that keeps track of a set of elements partitioned into a number of disjoint subsets. It supports two useful operations:

* Find: determine which subset a particular element is in

* Union: join two subsets into a single subset

Normally, we need to **compress path** and **merge rank** when implementing this data structure. After that, by comparing the result of two Find operations, one can determine whether two elements are in the same subset in **O(log n)** time.

Besides, it is widely used in other famous algorithms, such as **Tarjan** for LCA problem and **Kruskal** for shortest path.

# Example

## People Infected Virus

**Description**

There are n (0...n-1) students, m student unions, each union has k students.
Calculate how many people are infected of virus if people zero is infected?


Input:

First line has two numbers n and m. Following m lines are each union's students.
The first number is students number k, and then following k numbers standing for students ID.

Last line 0 0 means ending input.

100 4

2 1 2

5 10 13 11 12 14

2 0 1

2 99 2

200 2

1 5

5 1 2 3 4 5

1 0

0 0

**Analysis**

We can merge those students who are in the same student union to one set when reading input data. 
And meanwhile calculate how many students in this set. The count of set with student ID 0 is the result.

This is a typical use of UFS and it just records student's number.
In other cases, we can record any info in the node specific to that question.

**Code**

```cpp
class UnionFindSet
{
private:
    struct Node {
        int parent; // parent of this node
        int rank; // rank value for merge

        // can record any data here
        int cnt; // number of people infected in this set

        Node(): parent(-1), rank(0), cnt(1) {}
    };

    vector<Node> node;

public:
    UnionFindSet(int n) : node(n + 1) {}

    int Find(int x)
    {
        if (node[x].parent == -1) return x;
        return node[x].parent = Find(node[x].parent); // compress path
    }

    void Union(int x, int y)
    {
        int u1 = Find(x);
        int u2 = Find(y);

        if (u1 == u2) return; // same set

        if (node[u1].rank < node[u2].rank) {
            node[u1].parent = u2;
            node[u2].cnt += node[u1].cnt;
        } else { // >=
            node[u2].parent = u1;
            node[u1].rank = max(node[u1].rank, node[u2].rank + 1);
            node[u1].cnt += node[u2].cnt;
        }
    }

    int GetNum(int x)
    {
        return node[Find(x)].cnt;
    }
};

int main(int argc, char *argv[])
{
#ifndef ONLINE_JUDGE
    freopen("input", "r", stdin);
    // freopen("output","w",stdout);
#endif

    int n, m, k;

    while (cin >> n >> m && n > 0) {
        UnionFindSet ufs(n);

        while (m--) {
            int x, y; // two students

            cin >> k;
            k--;
            cin >> x;

            while (k--) {
                cin >> y;
                ufs.Union(x, y);
            }
        }
        cout << ufs.GetNum(0) << endl;
    }

    return 0;
}
```
