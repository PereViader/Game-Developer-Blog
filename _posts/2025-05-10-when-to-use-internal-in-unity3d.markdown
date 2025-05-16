---
layout: post
title: "When to use internal in Unity3d"
date: 2025-05-16 00:00:00 +1
---

My take on whether to use `public` or `internal` for members in my types is that I lean towards making them `public`.

Experience has shown me that when I feel the urge to hide a piece of functionality using `internal`, it's often a nudge that the functionality might be happier living in a different type altogether. If I take the step to extract it into a new class and make its methods public there, the new component becomes much easier to reuse down the line. Plus, testing that specific piece of logic in isolation turns a whole lot simpler.

Types that mix `internal` and `public` methods in application code usually signal they might be juggling too many responsibilities.

This code smell however should be reconsidered for library code. In that case, hiding some of the functionality in persue of a cleaner API might be preferable. Internal functionality can allow libraries to optimize the code while at the same time providing a smaller API surface thus resulting in fewer breaking changes during version updates.