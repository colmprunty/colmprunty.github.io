---
layout: post
title: Death by Typo
---

I don't know ReactJs. Clearly. As such I was stuck for about three days on why a function wouldn't work correctly. I found out about how state works, and about binding functions in the constructor and all that. But I chopped what I had down to two identical functions doing identical things, and one of them worked. I wrote up a full Stackoverflow question and was about to post, when I saw it. I had this:

```javascript
this.makeChoice - this.makeChoice.bind(this);
this.incrementCounter = this.incrementCounter.bind(this);
 ```

 Do you see it? Can you tell which one worked?

 Nothing flagged up that bloody hyphen, there's no compilation, the runtime error was "I don't know what `this` is". 

The rubber ducky, it works. 