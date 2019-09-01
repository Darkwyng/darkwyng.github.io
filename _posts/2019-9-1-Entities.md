---
layout: post
title: Entities in Stars!
---

## Requirements
Like a lot of other software, [_Stars!_](https://github.com/Darkwyng/stars) contains lots of features that are totally independent from each other.
But an implementation of Stars! also needs to allow for interaction between these components, for example:
- The disadvantage of _Only Basic Remote Mining_ gives you an extra 10 % planet capacity as compensation. 
- You can put _gadgets_ into your ships. They provide several benefits, that will be handled by different (Java-)components, impacting several features. And also
    - you need to _research_ gadgets to be able to use them 
    - each gadget has a weight, which affects _fuel usage_
    - depedending on the _race_ you are playing, some gadgets may be available to only you

Due to such requirements and because I also want to allow for addons that are not part of the original game, I am aiming for a very high level of [Extensibility](https://en.wikipedia.org/wiki/Extensibility).

## Code
To take this to an extreme, I started as [_entities_](https://github.com/Darkwyng/stars/blob/ca51c2e16b7933f90fefc2c1a9884ecd63c03bb9/parent/base/src/main/java/com/pim/stars/dataextension/api/Entity.java) being basically just maps
```java
public interface Entity {
	public Object get(String key);
	public void set(String key, Object value);
}
```
and I implemented a [bean](https://github.com/Darkwyng/stars/blob/ca51c2e16b7933f90fefc2c1a9884ecd63c03bb9/parent/addons/src/main/java/com/pim/stars/planets/api/extensions/PlanetName.java) for each _attribute_ of an entity, e.g:
```java
@Component
public class PlanetName implements DataExtensionPolicy<Planet, String> {

	@Override
	public Class<Planet> getEntityClass() {
		return Planet.class;
	}

	@Override
	public Optional<? extends String> getDefaultValue() {
		return Optional.empty();
	}
}
```
These beans implement [this interface](https://github.com/Darkwyng/stars/blob/ca51c2e16b7933f90fefc2c1a9884ecd63c03bb9/parent/base/src/main/java/com/pim/stars/dataextension/api/policies/DataExtensionPolicy.java):
```java
public interface DataExtensionPolicy<E extends Entity, T> {

	public Class<E> getEntityClass();
	public Optional<? extends T> getDefaultValue();

	public default String getKey() { /* ... */ }

	public default T getValue(final E entity) {
		return (T) entity.get(getKey());
	}
	public default void setValue(final E entity, final T newValue) {
		entity.set(getKey(), newValue);
	}
}
```

### Usage
Getting and setting the attribute is done like [so](https://github.com/Darkwyng/stars/blob/023e14ce84b7793c2100266d21e2897ba1a73e9b/parent/addons/src/main/java/com/pim/stars/planets/imp/effects/PlanetGameInitializationPolicy.java):
```java
@Autowired private PlanetName planetName;
...
Planet newPlanet = initializeNewPlanet(game, data);
String name = selectNewPlanetName(availableNames);
planetName.setValue(newPlanet, name);
```
It works.

## How is it?
It is interesting to see that it works. It might fit well with storing data in a NoSQL database, which I might get to yet.

It's great that I can explicitly reference the attributes when a _turn_ is created from the game (when the complete game data is transformed into the view that a single player has)
```java
builder.transformExtension(PlanetName.class).copyAll().build();
```

All that could be done with getters, setters and reflection, but I wanted to try something new.

An alternative would be for each component to have their own entity, e.g. "BasicPlanet" (with an ID and name), "CargoPlanet" (which stores minerals and colonists). That would also be very [DDD](https://en.wikipedia.org/wiki/Domain-driven_design) and would also allow for extensions, because you can just add your own "MyFeaturePlanet".
But I have the feeling I have done that: it's just that you add an "MyFeature"-attribute instead of a "MyFeature"-entity.

When the data is stored together (in the map), it _is_ jumbled up. Maybe that will prove painful, maybe when migrating data?

## PS
Thank you, [Migrate Maven Projects to Java 11](https://winterbe.com/posts/2018/08/29/migrate-maven-projects-to-java-11-jigsaw/).