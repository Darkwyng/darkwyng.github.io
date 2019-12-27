---
layout: post
title: Spring configurations
---

This post is about how Spring configurations are used in this project. 

[_Stars!_](https://github.com/Darkwyng/stars) is structured into modules that represent a feature, e.g. concerning [minerals](https://github.com/Darkwyng/stars/tree/master/parent/addons/src/main/java/com/pim/stars/mineral) (This module handles mineral concentrations of planets, building  mines and mining itself):

```
mineral
|   MineralConfiguration.java
|---api
|   |---policies
|   |       MiningPolicy.java
|   ...
|   
|---imp  
|   |   MiningCalculator.java 
|   |---...
|   ...
```

E.g. the `MiningCalculator` will load all implementations of `MiningPolicy` and calculate how many minerals are mined. A `MiningPolicy` might be implemented by another module (e.g. for remote mining).

### Configurations for productive code
`MineralConfiguration.java` defines which classes are part of the application context:

```
public interface MineralConfiguration {

	@Configuration @ComponentScan(excludeFilters = {... })
	@EnableMongoRepositories(basePackageClasses = { MineralRaceRepository.class, MineralPlanetRepository.class })
	public static class Provided {

	}

	@Configuration @Import({ Provided.class, CargoConfiguration.Complete.class, ... })
	public static class Complete {

	}
}
```

It actually contains two configurations:
- the `Provided` application context of the module. These are classes added by the module. It cannot be started, because it lacks Spring beans added by other modules (e.g. the `CargoProcessor` added by `CargoConfiguration` is used by `MiningCalculator`).
- the `Complete` application context, which really can be started, because it imports all other `Complete` configurations that it needs (e.g. `CargoConfiguration.Complete`).

I like this separation, because I can use `Provided` for _component tests_ (that test a few classes interacting; together with `@MockBean` for beans provided by other modules) and `Complete` for productive code and also for _integration tests_ (where (in my case) a module is tested together with other modules).


#### excludeFilters

The component scan of `Provided` needs to exclude the `Complete` configuration explicitly. Otherwise both would contain all the beans of `Complete`:

```
@ComponentScan(excludeFilters = { @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, value = Complete.class) })
```

### Configurations for tests

As explained [e.g. here](https://reflectoring.io/unit-testing-spring-boot/#dont-use-spring-in-unit-tests), _unit tests_ (that test a single class) usually don't need a Spring application context. For a _component test_ I do set up an application context.

```
public class MineralTestConfiguration {

	@MockBean
	private CargoProcessor cargoProcessor;

	...
	
	@Configuration
	@Import({ MineralTestConfiguration.class, MineralConfiguration.Provided.class })
	@Profile("WithoutPersistence")
	public class WithoutPersistence {

		@MockBean
		private MineralRaceRepository mineralRaceRepository;
		@MockBean
		private MineralPlanetRepository mineralPlanetRepository;
	}

	@Configuration
	@EnableAutoConfiguration // Required by @DataMongoTest
	@DataMongoTest
	@Import({ MineralTestConfiguration.class, MineralConfiguration.Provided.class })
	@Profile("WithPersistence")
	public class WithPersistence {

	}
}
```

The noteworthy detail is that I again have two configurations:
- An application context that mocks repositories for tests that do not need persistence.
- An application context that starts up a MongoDB for the test, when persistence is tested.

Testing with the MongoDB slows down the tests significantly, so it is worth making this distinction. I save 20 seconds in a 90-seconds-maven-install by not starting the DB every time.

#### Profile

The two configurations are marked with `@Profile`. This is partnered with `@ActiveProfiles` in the tests that use them. This is done, so that the two profiles are not used at the same time (due to the component scan).

```
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = MineralTestConfiguration.WithoutPersistence.class)
@ActiveProfiles("WithoutPersistence")
public class MineralConfigurationTest {
    ...
```
#### Integration tests

The `Complete` configurations are used in integration tests:

```
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = { GameConfiguration.Complete.class, MineralConfiguration.Complete.class,
		PersistenceTestConfiguration.class, RaceTestApiConfiguration.class, MineralTestApiConfiguration.class })
public class MineralGenerationIntegrationTest {
    ...
```

### Things I have given up

#### Bean vs. ComponentScan

I started out using `@Bean`-methods to define every bean instead of using `@ComponentScan`. I had hoped that would be more explicit, reducing the _Spring magic_. It does get tedious quickly though and clutters the configuration class with repetitive methods that do not contain any logic. So I switched to component scan.

Instead I use `@Bean`, where it is really needed, e.g. for a list of beans, where the length depends on the configuration:

```
/** Create the mineral types that are configured in the properties. */
@Bean
public List<MineralType> mineralTypes(final MineralProperties mineralProperties) {
	return mineralProperties.getTypeIds().stream().map(GenericMineralType::new).collect(Collectors.toList());
}
```

The component scan is the reason for the use of `@Profile` and `@ActiveProfiles` (see above). Nobody's perfect.

#### Required beans

I also set out to include an interface (called `Required`) that lists the beans needed for the `Provided` context to run. The interface would be implemented by the configuration used for tests. Since this is the only place, where it is used, it does not really serve any purpose.

```
public class MineralTestConfiguration implements MineralConfiguration.Required {

	@Bean
	@Override
	public CargoProcessor cargoProcessor() {
		return mock(CargoProcessor.class);
	}
    ...
```
