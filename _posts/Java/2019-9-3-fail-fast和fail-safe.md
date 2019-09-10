---
layout: post
title: Java集合类中的fail-fast和fail-safe
tags: Java
---

***

***

## 1. 简介

如果一个系统，在有异常或错误发生时立即中断执行，这种设计称为fail-fast

如果系统可以在发生某种异常或错误时继续执行，不会被中断，这种设计称为fast-safe

## 2. 集合类迭代器中的设计

