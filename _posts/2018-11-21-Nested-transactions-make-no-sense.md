---
layout: post
title: Nested transactions make no sense
---

Am I wrong? I mean, on a fundamental level. Say you have an outer transaction to do all the things, and two inner transactions A and B. If transaction A throws an error, and B is fine, you either:

- rollback the whole outer transaction, in which case B being fine is irrelevant.
- rollback A, commit B, and commit the whole thing, in which case the outer transaction was irrelevant. 