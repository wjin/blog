---
layout: post
title: "Consistent Hash"
description: ""
category: algorithm
tags: []
---
{% include JB/setup %}

# Introduction
Traditional hash tables must remap all keys when changing slot size: **hash(key) mod N**.

Consistent hashing is a special kind of hashing such that when a hash table is ***resized***, only ***K/N*** keys need to be remapped on average, where K is the number of keys, and N is the number of slots. [This](http://www.codeproject.com/Articles/56138/Consistent-hashing) article gives a good explanation about it.

# Property
* Monotonicity
* Balance

# Usage
* Distributed Caching System
* Load Balance

# Code Practice
As a practice, I design a cache system to store person information in a few cache servers using consistent hashing. 

And then remove one server from the cache system to observe affected data, it shows that only data on this removed server are affected. And we don't need to remap other data stored on other servers.

This simple code snippet simulates how consistent hashing works.

```cpp
#include <iostream>
#include <map>
#include <vector>
#include <string>
#include <functional>

using namespace std;

// Hash Function Class used to generate a number between 0 to MAX
// leverge stl hash template class
class HashFunction
{
public:
    size_t operator()(const string& str) const
    {
        // just let ring have 100 slots: 0 to 99
        // so that it is easy to observe output
        return m_hash(str) % 100;
    }
private:
    hash<string> m_hash;
};

// according to consistent hashing algorithm
// we will map server node and data key in this ring
// and then choose appropriate server to store data
template <typename HashFunc>
class HashRing
{
public:
    HashRing(size_t replicas)
        : m_replicas(replicas)
    {
    }

    void AddNode(const string& node)
    {
        size_t slot;
        cout << "Adding server : " << node << endl;

        for (size_t i = 0; i < m_replicas; i++) {
            slot = m_hash(node + to_string(i));
            m_ring[slot] = node;
            cout << " add virtual node " << i << " , slot " << slot << endl;
        }
    }

    void RemoveNode(const string& node)
    {
        for (size_t i = 0; i < m_replicas; i++) {
            size_t slot = m_hash(node + to_string(i));
            m_ring.erase(slot);
        }
    }

    const string& GetNode(const string& data) const
    {
        size_t slot = m_hash(data);

        // Look for the first node >= hash
        auto ite = m_ring.lower_bound(slot);
        if (ite == m_ring.end()) {
            // Wrapped around; get the first node
            ite = m_ring.begin();
        }
        return ite->second;
    }

private:
    map<size_t, string> m_ring;
    const size_t m_replicas; // virtual node number
    HashFunc  m_hash; // hash function object
};

struct Person {
    Person(int id = -1, string name="", string addr = "", size_t phone = 0) :
        m_id(id), m_name(name), m_addr(addr), m_phone(phone)
    {
    }

    int m_id;
    string m_name;
    string m_addr;
    size_t m_phone;
};


template <typename Key, typename Value>
class CacheServer
{
public:
    void Set(const Key& k, const Value& v)
    {
        m_cache[k] = v;
    }

    Value Get(const Key& k) const
    {
        Value v;
        auto ite = m_cache.find(k);
        if (ite != m_cache.end()) {
            v = ite->second;
        }
        return v;
    }

    void Remove(const Key& k)
    {
        auto ite = m_cache.find(k);
        if (ite != m_cache.end()) {
            m_cache.erase(ite);
        }
    }

private:
    // data can be stored on this server
    // simply use person's id as key
    map<Key, Value> m_cache;
};


int main()
{
    map<string, CacheServer<int, Person>> servers;
    // initialize servers according to server host name
    // here we have 3 servers used to store Person information
    // CacheServer stores person as : <id, info>
    servers["cache1.wjin.org"] = CacheServer<int, Person>();
    servers["cache2.wjin.org"] = CacheServer<int, Person>();
    servers["cache3.wjin.org"] = CacheServer<int, Person>();

    HashRing<HashFunction> ring(2);

    // persons
    vector<Person> persons = {
        { 1, "Eric King",    "Shanghai",  1111 },
        { 2, "Peter Will",   "Beijing",   2222 },
        { 3, "Smith John",   "Shenzheng", 3333 },
        { 4, "Joe Richard",  "Guangzhou", 4444 },
        { 5, "Tim Hans",     "Chengdu",   5555 },
        { 6, "Tom Paul",     "Hangzhou",  6666 },
        { 7, "Kate James",   "Hangzhou",  7777 },
        { 8, "Jim Jordan",   "Hangzhou",  8888 },
    };

    // add server to the hash ring
    for (auto ite = servers.begin(); ite != servers.end(); ++ite) {
        string name = ite->first;
        ring.AddNode(name);
    }
    cout << "-------------------------------------------------" << endl;

    // store person info
    for (auto p : persons) {
        // use id + name to find an appropriate server name in ring
        string host = ring.GetNode(to_string(p.m_id) + p.m_name);
        if (host.empty()) throw runtime_error("No server available");

        cout << "storing ID " << p.m_id << " on server " << host << endl;
        servers[host].Set(p.m_id, p);
    }
    cout << "-------------------------------------------------" << endl;

    // read data from server
    for (auto p : persons) {
        string host = ring.GetNode(to_string(p.m_id) + p.m_name);
        CacheServer<int, Person> server = servers[host];

        Person ret = server.Get(p.m_id); // read data
        cout << "ID " << p.m_id << " stored on server " << host << endl;
        cout << "  ID : " << p.m_id << " Name : " << ret.m_name << " Addr : " \
             << ret.m_addr << " Phone : " << ret.m_phone << endl;
    }
    cout << "-------------------------------------------------" << endl;

    // remove server cache3.wjin.org
    ring.RemoveNode("cache3.wjin.org");

    // read data again
    // as we removed server cache3, data stored on it are not available
    // we got default person info
    // however, other data won't be affected, that is consistent hashing
    for (auto p : persons) {
        string host = ring.GetNode(to_string(p.m_id) + p.m_name);
        CacheServer<int, Person> server = servers[host];
        Person ret = server.Get(p.m_id); // read data

        cout << "ID " << p.m_id << " stored on server " << host << endl;
        cout << "  ID : " << p.m_id << " Name : " << ret.m_name << " Addr : " \
             << ret.m_addr << " Phone : " << ret.m_phone << endl;
    }

    return 0;
}
```

# Next
Above solution is naive. Normally, when a server was failure, we can map data on that server to other servers. Also, if we could add new servers to the ring, we need to split data from others to this new added server.


# Reference
* [http://thor.cs.ucsb.edu/~ravenben/papers/coreos/kll%2B97.pdf](http://thor.cs.ucsb.edu/~ravenben/papers/coreos/kll%2B97.pdf)
* [http://en.wikipedia.org/wiki/Consistent_hashing](http://en.wikipedia.org/wiki/Consistent_hashing)
