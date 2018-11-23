---
layout: post
title: I have to go delete some tests
---

I watched [this talk](https://www.youtube.com/watch?v=EZ05e7EMOLM) and I feel like I've reached some kind of enlightenment. I've been in the guts of a bunch of data layer unit tests in work lately, and I've seen things with this pattern.


## Integration test
```csharp
[Test]
public void get_cat_gets_a_cat(){
    // given
    Cat expectedCat = // setup stuff

    // when
    var cat = _catRepo.GetCat(catId); // real database call

    // then
    Assert.TheSame(expectedCat, cat);
}
```

## Unit test

```csharp
[Test]
public void get_cat_calls_stored_proc_correctly(){
    // setup stuff

    // when
    _mockCatRepo.GetCat(catId);

    // then
    _mockCatRepo.Verify(x => x.ExecuteStoredProcedure("dbo.get_a_nice_cat"));
    _mockCatRepo.Verify(x => x.CalledWithParam("catId" == catId);)
}
```

You get the idea. There's loads of it. The idea is that the unit test is completely pointless. Why do I care if it calls a particular stored procedure? All I want is a cat. The implications! If you're only testing your public API, and you have loads of internal code doing stuff, does that mean that code coverage metrics are meaningless? It seems that way. 

In summation, I need to do some deleting.