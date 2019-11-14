---
layout: post
title: Builders; not building?
---

I had an issue with a [builder](https://dzone.com/articles/design-patterns-the-builder-pattern). I had used it like this:
```java
reportCreator.start(game, raceId).type(PlanetHasBuiltMinesReport.class)
		.addArguments(name, numberOfItems);
```

My Test showed that it was not working - the report was not created - and it took me a while to find the cause. It should have been:
```java
reportCreator.start(game, raceId).type(PlanetHasBuiltMinesReport.class)
		.addArguments(name, numberOfItems).create();
```
As you can see, the omission is hard to spot.

So I think, maybe one should always pass a parameter with the last method - so that the call is not forgotten.
```java
reportCreator.start(game, raceId)
		.addArguments(name, numberOfItems)
		.create(PlanetHasBuiltMinesReport.class);
```
Looks strange?