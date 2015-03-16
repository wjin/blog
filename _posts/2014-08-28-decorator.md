---
layout: post
title: "Decorator"
description: ""
category: design pattern
tags: []
---
{% include JB/setup %}

# Decorator

## Introduction

The decorator pattern **attaches additional responsibilities** on an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality. Decorators have **the same supertype** as the objects they decorate.

## Example

Everybody plays games, right? When we get a fantastic weapon, we can hardly wait to *decorate* our character to a superman. Here is an example that using Sword and Wand to decorate our worrior and wizard respectively, plus a crazy worrior with Sword and Wand.

## Code

### Cpp

```cpp
// abstract component
class Character
{
public:
    virtual void attack() = 0;
};

// concrete component
class Warrior : public Character
{
public:
    void attack()
    {
        cout << "Warrior is attacking" << endl;
    }
};

// another concrete component
class Wizard : public Character
{
public:
    void attack()
    {
        cout << "Wizard is attacking" << endl;
    }
};

// abstract decorator
// decorator has the same type of the object
// that it will decoreate
class Weapon : public Character
{
};

// concrete decorator
class Sword : public Weapon
{
private:
    shared_ptr<Character> person;

public:
    Sword(shared_ptr<Character> ptr) : person(ptr) {}
    void attack()
    {
        cout << "Begin...";
        person->attack();
        cout << "with Sword";
        cout << "...End" << endl;
    }
};

// another concrete decorator
class Wand : public Weapon
{
private:
    shared_ptr<Character> person;

public:
    Wand(shared_ptr<Character> ptr) : person(ptr) {}
    void attack()
    {
        cout << "Begin...";
        person->attack();
        cout << "with Wand";
        cout << "...End" << endl;
    }
};

int main(int argc, const char* argv[])
{
    shared_ptr<Character> war(new Warrior());
    shared_ptr<Character> wiz(new Wizard());

    war->attack();
    wiz->attack();

    shared_ptr<Character> warWithSword(new Sword(war));
    shared_ptr<Character> warWithWand(new Wand(war));
    shared_ptr<Character> wizWithSword(new Sword(wiz));
    shared_ptr<Character> wizWithWand(new Wand(wiz));

    warWithSword->attack();
    warWithWand->attack();
    wizWithSword->attack();
    wizWithWand->attack();

    // crazy warrior with sword and wand :(
    shared_ptr<Character> warWithSwordAndWand(new Wand(warWithSword));
    warWithSwordAndWand->attack();

    return 0;
}
```

### Java

```java
// abstract component
abstract class Character {
	public void attack() {
	}
}

// concrete component
class Warrior extends Character {
	@Override
	public void attack() {
		System.out.println("Warrior is attacking");
	}
}

// another concrete component
class Wizard extends Character {
	@Override
	public void attack() {
		System.out.println("Wizard is attacking");
	}
}

// abstract decorator
// decorator has the same type of the object
// that it will decorate
class Weapon extends Character {
}

// concrete decorator
class Sword extends Weapon {
	private final Character person;

	public Sword(Character ptr) {
		person = ptr;
	}

	@Override
	public void attack() {
		System.out.print("Begin...");
		person.attack();
		System.out.print("with Sword");
		System.out.println("...End");
	}
}

// concrete decorator
class Wand extends Weapon {
	private final Character person;

	public Wand(Character ptr) {
		person = ptr;
	}

	@Override
	public void attack() {
		System.out.print("Begin...");
		person.attack();
		System.out.print("with Wand");
		System.out.println("...End");
	}
}

public class Decorator {
	public static void main(String[] args) {
		Character war = new Warrior();
		Character wiz = new Wizard();

		war.attack();
		wiz.attack();

		Character warWithSword = new Sword(war);
		Character warWithWand = new Wand(war);
		Character wizWithSword = new Sword(wiz);
		Character wizWithWand = new Wand(wiz);

		warWithSword.attack();
		warWithWand.attack();
		wizWithSword.attack();
		wizWithWand.attack();

		// crazy warrior with sword and wand :(
		Character warWithSwordAndWand = new Wand(warWithSword);
		warWithSwordAndWand.attack();
	}
}
```
