---
categories: [cpp]
date: '2025-02-16T23:07:00-00:00'
layout: post
permalink: /c/software-readability/
tags: [c, cpp]
title: Software Readability
---

Writing readable code is important because code is read more often than it is written. The most readable codebases contain code that doesn't need to be read. This is possible by building on readers' intuition on how code should work and by structuring and annotating code such that readers only end up reading the parts of the code relevant to them at that specific time.

Readability, while sometimes subjective, can have some metrics that can be objectively measured. There is a term called Cognitive Complexity (not to be confused with Cyclomatic Complexity), used by SonarQube, that tries to objectively measure readability. There is an excellent white paper explaining it [here](https://www.sonarsource.com/docs/CognitiveComplexity.pdf) ([alt](https://web.archive.org/web/20241121022851/https://www.sonarsource.com/api/download?asset=aHR0cHM6Ly9hc3NldHMtZXUtMDEua2MtdXNlcmNvbnRlbnQuY29tOjQ0My82OTQxNDk0NS1mMDIyLTAxYjAtNjJjOS1mZGQxMzIzZjVmZTUvMzk0NzUyMzAtYzNmZi00ZTczLThhYjMtZmUwYzlmMjFlOWRkL0NvZ25pdGl2ZV9Db21wbGV4aXR5X1NvbmFyX0d1aWRlXzIwMjMucGRm)). There are also some great recommendations from the [ISO CPP guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-naming).

All code is technical debt. More code is typically more debt. Less code is typically less debt. The most readable codebase contains the most code that doesn't need to be read. I'm sure of this on a macro-scale; and while it does sometime apply on a micro-scale, it's not as ubiquitous at the scope of a single function implementation compared to the class or namespace scope. 

On average, the less code a reader has to read, the easier they will understand a concept. This is one of the tangential benefits of software abstraction layers: you only need to know the abstraction, not the myriad of implementations or implementation details on the other side. 

Readable code is simple; modular. It is also familiar. However, familiarity must not be mistaken for simplicity: what you are familiar with, someone else will not be.
