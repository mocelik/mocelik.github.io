---
categories: [cpp]
date: '2025-02-26T23:55:36-05:00'
layout: post
permalink: /c/naming-and-software-identity/
tags: [cpp, Software Architecture]
title: Naming and Software Identity
---

Your name is one of the most important parts of your identity. It is the same with code. Each module, class, function, and variable has a unique identity and fulfills a unique purpose identified by its name.

The precision and length of a name can correspond to the scope: if a name is seen often, it’s probably a highly reusable and generic entity. These entities can have shorter, more vague (reusable) and generic names. If a name is seen less often, it’s probably in a small scope and may be an implementation detail. These entities can have longer, more precise and narrow names. Using a generic name for a precise concept or a precise name for a generic concept forces readers to think about the concept in the wrong scope, and reduces readability.

*The precision of naming takes away from the uniqueness of seeing* — Pierre Bonnard \[[1](https://abseil.io/tips/130)\]

Pierre Bonnard was an artist, and made the statement above in the context of art. Writing is also a form of art, as is writing software. I interpret this statement as claiming that the more precisely something is named, the less mysterious the thing is: the intention and functionality has been narrowed down. If the name is not precise enough, readers need to make assumptions, and may be wrong. If the name is too precise, readers may be overwhelmed with the level of detail, and the definition may be too brittle to survive a refactor.

I don’t mean to understate the difficulty of finding an appropriate name. However, if this is a consistent issue, it may be because the roles of the components in the system are not properly defined. It may also be due to a lack of understanding or application of design patterns. Design patterns have ubiquitous terminology. While one may unknowingly use design patterns in their code with their own custom terminology, a reader who is already familiar with design patterns would grasp the concepts quicker if existing terminology is reused.

When it comes to naming standards like capitalization, camelCase or snake\_case and so on, it’s extremely important to consistently uphold those standards throughout the codebase. Naming standards can help a reader identify, at a glance, what the type or scope of something is. If the standards are not consistently applied and enforced, then readers will either not trust the standards, in which case they’re no longer effective, or worse, they will trust the standards and make wrong assumptions.
