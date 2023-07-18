
---
title: "Debugger Techniques"
layout: post
date: 2023-02-24 22:44
image: /assets/images/markdown.jpg
headerImage: false
tag:
- debug
- technique
star: true
category: blog
author: torben feldthusen
description: Debugger Techniques
---



Debugger Techniques 

Setting Breakpoints As previously mentioned in the section on disassemblers, a 6-line C program doing something as simple as outputting "Hello, World!" turns into massive amounts of assembly code. Most people don't want to sift through the entire mess to find out the information they want. It can be time consuming just to find the information one desires by just looking through the code. As an alternative, one can choose to set breakpoints to halt the program once it has reached a given point within the program's code.
