---
layout: post
title: "Adapter and Facade"
description: ""
category: design pattern
tags: []
---
{% include JB/setup %}


# Adapter and Facade

## Adapter

The adapter pattern **converts the interface** of a class into another interface the clients expect. Adapter lets classes work together that couldn’t otherwise because of incompatible interfaces. 

There are two kinds of adapter pattern:

* object adapter using **composition**

* olass adapter using **multi-inheritance**


## Facade

The façade pattern provides a **unified interface** to a set of interfaces in a subsystem. Façade defines a higher level interface that makes the subsystem easier to use.

## Difference

* Adapter: change interface        
 
* Façade: simplify interface
 
* Decorator: do not change interface but add responsibility