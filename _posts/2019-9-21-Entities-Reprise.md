---
layout: post
title: Entities - a reprise
---

In the [last post](https://darkwyng.github.io/Entities/) I described an alternative to a POJO with getters and setters for holding data. I used beans for access:
```java
@Autowired private PlanetName planetName;
...
Planet newPlanet = ...
String name = ...
planetName.setValue(newPlanet, name);
```
I ended with a few pros and cons and 
> When the data is stored together (in the map), it _is_ jumbled up. Maybe that will prove painful, maybe when migrating data?

## Conclusion
Planning the next step in _Stars!_ - persistence - confirms my worries: it would be a mess:

I would need a (small) framework to convert my entities into maps to be able to store and reread them. This is something that has been solved by others for POJOs already.

To avoid reading all the data in one go, I would need to add further code for lazy loading and things like that.

[DDD](https://en.wikipedia.org/wiki/Domain-driven_design) is the way to go. How boring.