---
layout: post
title: "Singleton"
description: ""
category: design pattern
tags: []
---
{% include JB/setup %}


# Singleton

## Introduction

The singleton pattern ensures a class has only one instance, and provides a global point of access to it.

**Multithread**

Singleton is the simplest design pattern among all patterns, however, we must be careful when dealing with multithread environment. Basically, we have following ways:

* java **synchronized** method if performance is not critical to application

* move to an **early created instance** rather than a lazily created one (jvm can guarantee thread-safe when load this class, however, it may fail when using multiple class loaders. Also it may waste memory if we do not use it later)

* use **double-checked locking** to reduce use of synchronization in getInstance(), define unique instance as volatile. This method does not suit c++

* **static code block** in java and and **static constructor** in C#

## Code

### Cpp

```cpp
class Singleton
{
public:
    static Singleton& getInstance() // reference
    {
        // Guaranteed to be destroyed. Instantiated on first use
        static Singleton instance;
        return instance;
    }

    void dump()
    {
        cout << "I am singleton pattern: " << this << endl;
    }

private:

    Singleton() {}  // Constructor? (the {} brackets) are needed here.

    // Dont forget to declare these two. You want to make sure they
    // are unaccessable otherwise you may accidently get copies of
    // your singleton appearing.
    Singleton(const Singleton &) = delete; // Don't Implement
    Singleton& operator=(const Singleton&) = delete; // Don't implement
};

int main()
{
    Singleton::getInstance().dump();
    Singleton::getInstance().dump();
    Singleton::getInstance().dump();

    return 0;
}
```

### Java

```java
// do not consider multithread
public class Singleton {
	private static Singleton instance; // static

	private Singleton() { // private
	}

	public static Singleton getInstance() { // static
		if (instance == null) {
			instance = new Singleton();
		}
		return instance;
	}

	public static void main(String[] args) {
		Singleton obj1 = getInstance();
		Singleton obj2 = getInstance();

		if (obj1 != obj2) {
			System.out.println("Wrong");
		} else {
			System.out.println("Right");
		}
	}
}


// synchronized method
public class Singleton {
	private static Singleton instance; // static

	private Singleton() { // private
	}

	public static synchronized Singleton getInstance() { // static
		if (instance == null) {
			instance = new Singleton();
		}
		return instance;
	}
	// ...
}


// double-checked locking
public class Singleton {
	private volatile static Singleton instance; // volatile static

	private Singleton() { // private
	}

	public static Singleton getInstance() { // static
		if (instance == null) {
			synchronized (Singleton.class) {
				if (instance == null) {
					instance = new Singleton();
				}
			}
		}
		return instance;
	}
    // ...
}


// early create object
public class Singleton {
	private final static Singleton instance = new Singleton(); // static

	private Singleton() { // private
	}

	public static Singleton getInstance() { // static
		return instance;
	}
	// ...
}


// static code block
public class Singleton {
	private final static Singleton instance; // static

	private Singleton() { // private
	}

	static {
		instance = new Singleton();
	}

	public static Singleton getInstance() { // static
		return instance;
	}
	// ...
}
```
