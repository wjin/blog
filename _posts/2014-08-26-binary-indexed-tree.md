---
layout: post
title: "Binary Indexed Tree"
description: ""
category: data structure
tags: []
---
{% include JB/setup %}



# Introduction

A **Fenwick tree** or **Binary Indexed Tree** is a data structure providing efficient methods for calculation and manipulation of the **prefix sums** or **cumulative frequency** of a table of values. It was proposed by Peter Fenwick in 1994. Here is an excellent explanation from [topcoder](http://community.topcoder.com/tc?module=Static&d1=tutorials&d2=binaryIndexedTrees).

**Question Definition**

Let's define the following problem: We have n boxes. Possible queries are:

1. add marble to box i 

2. sum marbles from box k to box l

The naive solution has time complexity of O(1) for query 1 and O(n) for query 2. Suppose we make m queries. The worst case (when all queries are 2) has time complexity O(n * m). 

Using some data structure (i.e.: segment tree for **RMQ**), we can solve this problem with the worst case time complexity of **O(m log n)**.

Another approach is to use Binary Indexed Tree data structure, also with the worst time complexity **O(m log n)**. However, Binary Indexed Tree is much easier to code, and requires less memory space, than RMQ.

**Basic idea**

Let f[idx] is frequencey for each idx, and r is a position in idx of the **last digit 1**. For example, idx = 100100 in binary, r is 2.

> bit[idx] = f[idx-2^r+1] + f[idx-2^r+2] + ... + f[idx]

**Useful Tips**

1. BIT input data have some dependency according to each problem's definition, normally we need to sort them.

2. BIT has a input scope, sometimes we need to discretize input data when input data range is too large (cannot allocate a huge BIT array).

3. 2D BIT


**Code Template**

```cpp
class BIT
{
private:
    const int maxIdx = 10000; // max index
    int bitMask; // bit mask for binary search
    vector<int> tree; // binary indexed tree array, tree[0] is not used

public:
    BIT() : tree(maxIdx)
    {
        bitMask = highbit(maxIdx);
    }

    // clear idx except the LSB '1'
    // 10101000 -> 10100000
    int lowbit(int idx)
    {
        return idx & (-idx);
    }

    // clear idx except the MSB '1'
    // 01010000 -> 01000000
    int highbit(int idx)
    {
        while (lowbit(idx)) {
            idx -= lowbit(idx);
        }

        return idx;
    }

    // update tree
    void update(int idx, int num)
    {
        while (idx <= maxIdx) {
            tree[idx] += num;
            idx += lowbit(idx);
        }
    }

    // get cumulative sum of f[1]...f[idx]
    int read(int idx)
    {
        int sum = 0;
        while (idx) {
            sum += tree[idx];
            idx -= lowbit(idx);
        }
        return sum;
    }

    // get frequency of f[idx]
    /*
    int readSingle(int idx)
    {
        return (read(idx) - read(idx - 1));
    }
    */
    int readSingle(int idx)
    {
        int sum = tree[idx]; // sum will be decreased
        if (idx > 0) { // spetree[idx]al case
            int z = idx - lowbit(idx); // make z first

            idx--;
            while (idx != z) { // at some iteration idx (y) will become z
                sum -= tree[idx];
                // substruct tree frequency which is between y and "the same path"
                idx -= lowbit(idx);
            }
        }
        return sum;
    }

    // following two functions using binary search to search a
    // cumulative frequencey which is equal to target cumFre.
    // be careful binary search is applicable if and only if
    // f[i] >= 0, 1 <= i <= maxIdx

    // bitMask - initialy, it is the greatest bit of maxIdx
    // bitMask store interval which should be searched

    // if in tree exists more than one index with a same
    // cumulative frequency, this procedure will return
    // some of them (we do not know which one)
    int find(int cumFre)
    {
        int idx = 0; // this var is result of function

        while ((bitMask != 0) && (idx < maxIdx)) { // nobody likes overflow :)
            int tIdx = idx + bitMask; // we make midpoint of interval
            if (cumFre == tree[tIdx]) // if it is equal, we just return idx
                return tIdx;
            else if (cumFre > tree[tIdx]) {
                // if tree frequency "can fit" into cumFre,
                // then include it
                idx = tIdx; // update index
                cumFre -= tree[tIdx]; // set frequency for next loop
            }
            bitMask >>= 1; // half current interval
        }

        if (cumFre != 0) // maybe given cumulative frequency doesn't exist
            return -1;
        else
            return idx;
    }

    // if in tree exists more than one index with a same
    // cumulative frequency, this procedure will return
    // the greatest one
    int findG(int cumFre)
    {
        int idx = 0;

        while ((bitMask != 0) && (idx < maxIdx)) {
            int tIdx = idx + bitMask;
            if (cumFre >= tree[tIdx]) {
                // if current cumulative frequency is equal to cumFre,
                // we are still looking for higher index (if exists)
                idx = tIdx;
                cumFre -= tree[tIdx];
            }
            bitMask >>= 1;
        }

        if (cumFre != 0)
            return -1;
        else
            return idx;
    }
};
```

# Examples

## Stars

**Description**

Astronomers often examine star maps where stars are represented by points on a plane and each star has Cartesian coordinates. Let the level of a star be an amount of the stars that are not higher and not to the right of the given star. Astronomers want to know the distribution of the levels of the stars.

You are to write a program that will count the amounts of the stars of each level on a given map.

Input

The first line of the input file contains a number of stars N (1<=N<=15000). The following N lines describe coordinates of stars (two integers X and Y per line separated by a space, 0<=X,Y<=32000). There can be only one star at one point of the plane. Stars are listed in ascending order of Y coordinate. Stars with equal Y
coordinates are listed in ascending order of X coordinate.

Output

The output should contain N lines, one number per line. The first line contains amount of stars of the level 0, the second does amount of stars of the level 1 and so on, the last line contains amount of stars of the level N-1.

Sample Input

5

1 1

5 1

7 1

3 3

5 5

Sample Output

1

2

1

1

0

**Analysis**

As input data have already been sorted, just insert it to BIT.

**Code**

```cpp
class Solution
{
private:
    const int MAX = 32002;
    vector<int> c; // binary indexed tree
    vector<int> level; // record each level's number

public:
    Solution() : c(MAX), level(MAX) {}

    int lowbit(int x)
    {
        return x & (-x);
    }

    void update(int x, int num)
    {
        while (x <= MAX) {
            c[x] += num;
            x += lowbit(x);
        }
    }

    // read returns cumulative sum of c[1]...c[x]
    // here it is the level of this star
    int read(int x)
    {
        int sum = 0;
        while (x) {
            sum += c[x];
            x -= lowbit(x);
        }
        return sum;
    }

    void solve()
    {
        int x, y, n;

        cin >> n;
        for(int i = 1; i <= n; i++) {
            cin >> x >> y;

            // as points are sorted by y coordinate with ascending order,
            // for each point, only count points before it in the input stream
			// x + 1 is to avoid case: x=0
            level[read(x + 1)]++;
            update(x + 1, 1);
        }

        for(int i = 0; i < n; i++)
            cout << level[i] << endl;
    }
};

int main(int argc, const char* argv[])
{
#ifndef ONLINE_JUDGE
    freopen("input", "r", stdin);
    // freopen("output","w",stdout);
#endif

    Solution sol;

    sol.solve();

    return 0;
}
```

## Circus Pyramid

**Description**

A circus is designing a tower routine consisting of people standing atop one
another's shoulders. For practical and aesthetic reasons, each person must be
both shorter and lighter than the person below him or her. Given the heights
and weights of each person in the circus, write a method to compute the largest
possible number of people in such a tower.

EXAMPLE:

Input (ht,wt): (65, 100) (70, 150) (56, 90) (75, 190) (60, 95) (68, 110)

Output:The longest tower is length 6 and includes from top to bottom:

(56, 90) (60,95) (65,100) (68,110) (70,150) (75,190)

**Analysis**

First sort person according to height, and then construct BIT by inserting person pairs, meanwhile calculate that person's largest possible number in the tower before inserting.

**Code**

```cpp
class Solution
{
private:
    struct Person {
        int ht; // height
        int wt; // weight

        Person(const int h = 0, const int w = 0) : ht(h), wt(w) {}
    };

    static bool cmp(const Person &lhs, const Person &rhs)
    {
        return lhs.ht < rhs.ht;
    }

    const int MAX = 1000;
    vector<int> c; // binary indexed tree

public:
    Solution() : c(MAX) {}

    int lowbit(int x)
    {
        return x & (-x);
    }

    void update(int x, int num)
    {
        while (x <= MAX) {
            c[x] += num;
            x += lowbit(x);
        }
    }

    int read(int x)
    {
        int sum = 0;
        while (x) {
            sum += c[x];
            x -= lowbit(x);
        }
        return sum;
    }

    void solve()
    {
        int n;
        cin >> n;
        vector<Person> v(n);

        for(int i = 0; i < n; i++) {
            cin >> v[i].ht >> v[i].wt;
        }

        sort(v.begin(), v.end(), cmp);

        int ret = INT_MIN;
        for (auto e : v) {
            ret = max(ret, read(e.wt));
            update(e.wt, 1);
        }

        cout << ret + 1<< endl; // +1 means including himself
    }
};

int main(int argc, const char* argv[])
{
#ifndef ONLINE_JUDGE
    freopen("input", "r", stdin);
    // freopen("output","w",stdout);
#endif

    Solution sol;

    sol.solve();

    return 0;
}
```

## Ultra-QuickSort

**Description**

In this problem, you have to analyze a particular sorting algorithm.
The algorithm processes a sequence of n distinct integers by swapping
two adjacent sequence elements until the sequence is sorted in ascending
order.

For the input sequence

9 1 0 5 4 ,

Ultra-QuickSort produces the output

0 1 4 5 9

Your task is to determine how many swap operations Ultra-QuickSort needs to
perform in order to sort a given input sequence.

Input

The input contains several test cases. Every test case begins with a line that
contains a single integer n < 500,000 -- the length of the input sequence.
Each of the the following n lines contains a single integer 0 <= a[i] <= 999,999,999,
the i-th input sequence element. Input is terminated by a sequence of length n = 0.
This sequence must not be processed.

Output

For every input sequence, your program prints a single line containing an integer number
op, the minimum number of swap operations necessary to sort the given input sequence.

Sample Input

5

9

1

0

5

4

3

1

2

3

0

Sample Output

6

0

**Analysis**

1. discretize and then BIT

2. merge sort

**Code**

```cpp
class Solution
{
private:
    struct node {
        int val; // value
        int id; // order in the input stream
    };

    static bool cmp(const node &lhs, const node &rhs)
    {
        return lhs.val < rhs.val;
    }

    int MAX;
    vector<int> c; // binary indexed tree

public:
    int lowbit(int x)
    {
        return x & (-x);
    }

    void update(int x, int num)
    {
        while (x <= MAX) {
            c[x] += num;
            x += lowbit(x);
        }
    }

    int read(int x)
    {
        int sum = 0;
        while (x) {
            sum += c[x];
            x -= lowbit(x);
        }
        return sum;
    }

    void solveOne(int n)
    {
        MAX = n;
        c.clear();
        c.resize(n + 1);
        vector<node> v(n + 1);

        // read data
        for(int i = 1; i <= n; i++) {
            cin >> v[i].val;
            v[i].id = i;
        }

        // discretize input
        sort(v.begin() + 1, v.end(), cmp);
        vector<int> discrete(n + 1);
        for(int i = 1; i <= n; i++) {
            discrete[v[i].id] = i;
        }

        // counting result
        // i is the input index
        // read(discrete[i]) is the number that little than or equal to discrete[i]
        // i - read(discrete[i]) is the number that grater than discrete[i]
        long long ret = 0;
        for(int i = 1; i <= n; i++) {
            update(discrete[i], 1);
            ret += (i - read(discrete[i]));
        }

        cout << ret << endl;
    }

    void solve()
    {
        int n = 0;
        while (cin >> n, n != 0) {
            solveOne(n);
        }
    }
};

int main(int argc, const char* argv[])
{
#ifndef ONLINE_JUDGE
    freopen("input", "r", stdin);
    // freopen("output","w",stdout);
#endif

    Solution sol;

    sol.solve();

    return 0;
}
```

## Count Cards

**Description**

There is an array of n cards. Each card is putted face down on table. You have two queries:

  1. T(i, j) (turn cards from index i to index j, include i-th and j-th card - card which was face down will be face up; card which was face up will be face down)
  
  2. Q(i) (answer 0 if i-th card is face down else answer 1)

**Analysis**

There is time complexity O(log n) for each query (and 1 and 2). In array f (of length n + 1) we will store each query T (i , j) - we set **f[i]++** and **f[j + 1]--**.

For each card k between i and j (include i and j) sum f[1] + f[2] + ... + f[k] will be increased for 1, for all others will be same as before, so our solution will be described sum (which is same as cumulative frequency) **module 2**.
