---
layout: post
title: .NET Core Unit Testing (Complete guide with sample project and exercises/solutions)
---

*This post is a thorough guide into starting unit tests in .NET, utilizing two popular libraries in Moq and xUnit.*

Before I started doing software development professionally, writing unit tests was usually an afterthought or something I neglected while working on my personal projects. After working for a while at a job that necessitates writing unit tests, the benefits of unit test coverage has become undeniably clear.

Among many other reasons, the tests help catch regressions in your codebase (something that used to work before stopped working at some point after code has been changed), they provide confidence that refactoring or adding/changing code won't break existing code. With that said, I can admit that for new and very tiny personal projects, the benefits aren't as profound. You most likely will have the small codebase and product understood well enough, such that when you make a change you'll also be conscious of other places that are affected.


<!--more-->

Regardless, it's a great habit to get into writing tests as you go along. Start writing them early on in the beginning of a new project, and try to keep your tests in parity as you add new features or change code. It's a lot easier writing tests as you go, as opposed to writing a huge application without them and having to painstakingly write them all at once. Also, most companies out there require unit testing (among other kinds of tests) so you might as well start getting into a habit that has great practical value.

This is a tutorial to help you get started. It's not going to cover every aspect of unit testing or get into advanced edge cases, but it will be thorough enough to provide a foundation for you to start implementing tests in your projects.

I am going to assume that you are at least somewhat familiar with using C# and building .NET applications. I've also created a sample project for this tutorial, hosted on github that you can clone, to follow along. I personally use Visual Studio 2015, and the projects contained in the solution are .NET Core 1.1 projects.

> VS 2017 has been released since I first wrote this blog post. I may update this in the future,
> but you shouldn't have problems going through this. Just make sure you're familiar with the changes
> of going from project.json in VS 2015 to the .csproj formats in VS 2017.

To start, here's my github repo for the tutorial: <a href="https://github.com/rushfive/UnitTestingExample" target="_blank">https://github.com/rushfive/UnitTestingExample</a>

Or you can just open a shell and run the command:

`git clone https://github.com/rushfive/UnitTestingExample.git`

I highly recommend following along by having the project opened up locally in visual studio. Towards the end of the tutorial, I've setup some empty practice tests that you can fill in using what you'll learn as practice. Now lets dive in!

---

## Testing Dependencies - xUnit and Moq

We'll be using two popular libraries for our unit testing:

**xUnit**

[https://xunit.github.io](https://xunit.github.io)

A unit testing tool for .NET. It provides decorators to mark methods as tests, along with the ability to create assertions. We'll dive into it in more depth later, but an assertion is essentially a statement that you want to test. An example, in English, would be "The expected result of running this method is false". If the method you created should produce a false return value but it returns true when running the test, you can conclude that either something in your testing logic or the actual code you're testing is flawed.

**Moq**

[https://github.com/moq/moq4](https://github.com/moq/moq4)

A mocking framework for .NET. We'll also dive into this more in depth later, but in a few words, mocks let us focus and test specifically the method that we're interested in. We know that a given method can make use of other object methods and such, but generally speaking, we're usually not interested in the validity of those other methods - you should probably be writing unit tests for those separately anyways! Thus, instead of relying on other dependencies of our method and class, we mock (think fake) the return values, so we can proceed testing our method.

---

## A Tour of the Sample Project and Architectural Design Decisions

The sample project revolves around retrieving report information from some database. I've split the solution's src codebase into three separate projects as depicted below:

![]({{ site.url }}/images/UnitTestingExampleProjectLayout.png)

The `ServerComponents` project contains the concrete implementations of our services, of which we have two. It also contains a `DatabaseContext` class that the services use to make database calls and retrieve the information we need (the functionality of the `DatabaseContext` is faked, this tutorial doesn't include a real database implementation).

`ServerComponents` references the `ServerAbstractions` project, which contains the interfaces for the services and databaseContext. It also contains the SQL entities that are returned from the DatabaseContext making calls to our pretend database layer.

And finally, `ServerAbstractions` references `CoreAbstractions` (thus `ServerComponents` also has a reference to `CoreAbstractions`). `CoreAbstractions` contains our primary Entities that are used within the service layer. It's good practice to separate your models for an Entity by layers.

For example, when you retrieve a model by making a database call, you might not need all that information for the web (presentation layer). Your web model might only need a subset of the model retrieved from the database, or maybe even some completely different bits of information. So rather than keeping one monolithic model that contains the properties needed for every aspect or layer of your application, you create a separate model for each layer.

A picture's worth a thousand words, so here's a diagram illustrating the above concerning a Person entity:

![]({{ site.url }}/images/Application_Layers_and_Model_Diagram.png)

Each of the three layers have their own `Person` model that is used within that layer. For example, in the service layer, if more than one service needs to reference a `Person`, they would all use and pass around a `Person` type object. However, once you need to jump layers, the model changes so you need to map one to another.

Here's the `UserSql` class from the `ServerAbstractions` project:

```c#
public class UserSql
{
	public Guid Id { get; set; }
	public string FirstName { get; set; }
	public string LastName { get; set; }
	public bool Enabled { get; set; }

	public static User ToEntity(UserSql sql)
	{
		return new User
		{
			Id = sql.Id,
			FirstName = sql.FirstName,
			LastName = sql.LastName,
			Enabled = sql.Enabled
		};
	}
}
```

The class contains a static method that takes an instance of a `UserSql` type and returns the mapped `User` type. Similarly, if there was a `UserModel` class that's used in the Web/Presentation Layer, I'd add a mapping method there as well, that takes a Service Layer model and maps it to the Web model. I generally like to keep my Service Layer entities as clean as possible, so they usually end up being POCOs (Plain Old CLR Objects), classes that only contain property definitions without any methods. You can take a look at the Entity classes inside the `CoreAbstractions` project for more examples.

The other main directory within our solution is `Test`, which contains several other class library projects. Generally, I like to create a class library project for each project in src that contains code we'll need to test.

We have `CoreAbstractions.Tests` and `ServerComponents.Tests`. The `CoreAbstractions` project contains a utility class with a single method that we'll write tests for. `ServerComponents` contains all the concrete implementations of the services, so we'll need to test the logic there as well. There's one additional project in `Test`, `TestsCore`. This project holds the references to xUnit, Moq, and any others that we'll need. It also contains a helper which we'll get into later. We have all our main testing projects referencing `TestsCore` such that they can pull in any of our custom testing helpers needed, along with getting a reference to the third party projects we need.

With all that out of the way, lets dive into the unit tests now, starting with some simple examples to test the utility function.

---

## Getting our Feet Wet - Unit Testing Basics

Lets start by taking a look at the utility class we'll be testing. It takes two strings as parameters, a first name and last name, and returns the full name. Needless to say, many of the examples in this tutorial aren't useful in a practical sense, but it'll do to drive the points home. Here's the utiliy class below:

```c#
public static class UserUtil
{
	public static string GetFullName(string firstName, string lastName)
	{
		if (string.IsNullOrWhiteSpace(firstName))
		{
			throw new ArgumentException("First name must be provided.");
		}
		if (string.IsNullOrWhiteSpace(lastName))
		{
			throw new ArgumentException("Last name must be provided.");
		}

		return $"{firstName.Trim()} {lastName.Trim()}".Trim();
	}
}
```

We can deduce a couple of things from it's code here:
- It requires both names to be passed in, with valid values (not null or empty)
- It'll return the two arguments concatenated together while also taking care of any leading and trailing spaces.

So, what is it exactly that we want to test here? It's simple, just the two points above. We want to ensure that our tests catch the throwing of the `ArgumentExceptions` given particular inputs, and we also need to ensure that given valid name inputs, it correctly returns a full name with leading/trailing spaces trimmed out.

Go to `CoreAbstractions.Tests\Utilities`, and open up `UserUtilTests.cs`. Lets take a look at the first unit test I've written, copied below here:

```c#
public void GetFullName_InvalidArguments_ThrowsArgumentException()
{
	Assert.Throws<ArgumentException>(() => UserUtil.GetFullName("", ""));
	Assert.Throws<ArgumentException>(() => UserUtil.GetFullName("Mary", ""));
}
```

The first thing to notice is that the method has been decorated using `[Fact]`. This marks the method as a test case (when using xUnit), and when Visual Studio builds and attempts to discover tests, it will look for these to build up the list of tests you can run.

Second, the name of the method. It's absolutely okay for the method names of test cases to be extremely long. As with the usual naming best practices, write just enough to convey the intent of the method clearly. I like to follow this format:

`MethodName_WhatsBeingTested_ExpectedResult`

So in this case, we're testing the `GetFullName` method, passing in invalid arguments, and ensuring that we get the expected result of `ArgumentExceptions` being thrown. There's no hard and fast rule to this, but it's good to come up with some naming scheme and stick to it for consistency.

Within the method body, we can see that we're making two assertions here. For a given test case to pass, all assertions within the method body must pass. Using the generic `Assert.Throws<T>` method, we can specify what type of `Exception` is thrown. We first test where the first argument is a blank string, which should trigger the first check. In the second assertion, we pass in a valid name for the first argument and a blank string for the second to test the validity of the last name.

Go ahead and build the solution, open the Test Explorer window, and run the test and you should see it passing.

> New to VS 2015 and NET Core 1.1?
>
> I ran into issues getting my unit tests to show up when I first started working on this stuff. Some of this stuff may be obvious, but here's a few things to check if you're having the same problem:
> - NET Core 1.1 is installed on your system
> - Your Visual Studio test settings Default Processor Architecture is set to x64 (or whatever architecture your builds are using)
> - Clean your solution and rebuild. Close the Test Explorer window and re-open
> You can also run the command `dotnet test` on the command line (from within the test project directories that contain the project.json) and you'll get more helpful output compared to what you see in Visual Studio's test output.
>
> You can also debug the tests like usual. Just place a breakpoint somewhere. Then, in the Test Explorer, right-click the test and select to debug it.

Now that we've tested the inputs, we want to make sure that our method is concatenating the names in the correct fashion. Here's a test testing one particular case:

```c#
public void GetFullName_ValidInputs_ReturnsCorrectResult_Example1()
{
	string fullNameResult = UserUtil.GetFullName(" Mary ", " Jane   ");
	Assert.Equal("Mary Jane", fullNameResult);
}
```

As simple as a unit test gets. We get the result of using the utility method, passing in a couple names with leading/trailing spaces. We then assert that the result matches what we expect our utility method to return.

Using xUnit's assertions, it's a convention when using `Assert.Equal()` to have the first argument be the expected value, and the second be the result of running the code you're testing. You'd get the same test result if you were to swap them, but it's good to stay consistent and stick with conventions. It'll help others more efficiently read and understand your unit tests.

You might've noticed the above method name ending with "Example1". Here's example number two, also testing for correct results:

```c#
[Theory]
[InlineData(" Mary", "  Jane  ", "Mary Jane")]
[InlineData(" Bob", "Marley", "Bob Marley")]
[InlineData(" Joe Hanson", " Lee   ", "Joe Hanson Lee")]
public void GetFullName_ValidInputs_ReturnsCorrectResult_Example2(string firstName, 
	string lastName, string expectedFullName)
{
	string fullNameResult = UserUtil.GetFullName(firstName, lastName);

	Assert.Equal(expectedFullName, fullNameResult);
}
```

There's another decorator provided by xUnit, `[Theory]`. Using Theory is also accompanied by N number of `[InlineData()]` decorators. Here's how using a Theory works:

- Each `[InlineData]` decorator represents a separate and distinct run of that test case.
- The arguments passed to the `[InlineData]` decorator are passed into the test method as it's run, and thus can be used within the method body. There are some restrictions on the type of values that can be passed in this way. For example, you cannot pass in most reference-type objects.
- Because each `[InlineData]` decorator represents a separate test, they also show up separately on the Test Explorer window. One InlineData case failing does not cause the others to fail.

In our example, we're passing three arguments through the decorator. The first and last names that our utility method expects, and the final one which is the expected result. Go ahead and run the tests to see them pass. Also, try messing with the inputs in the `[InlineData]` decorators to get some to fail and see how it all behaves.

---

## Taking it up a Notch - Testing classes that have dependencies

Now that we know the basics of creating tests and assertions, lets jump into testing the `UserService` class. Here it is below:

```c#
public class UserService : IUserService
{
	private IDatabaseContext DbContext { get; }

	public UserService(IDatabaseContext dbContext)
	{
		this.DbContext = dbContext;
	}

	public async Task<User> GetAsync(Guid userId)
	{
		if (userId == Guid.Empty)
		{
			throw new ArgumentException("Must specify a user id.", nameof(userId));
		}

		UserSql userSql = await this.DbContext.FindSingleAsync<UserSql>(userId);

		if (userSql == null)
		{
			throw new Exception($"User '{userId}' was not found.");
		}

		return UserSql.ToEntity(userSql);
	}
}
```

Again, nothing too crazy going on in our examples, but the above provides an interesting scenario not seen with the earlier utility method test.

The `UserService` has a dependency on a `IDatabaseContext`, which is what's leveraged to make the database calls for us. I've defined the interface to requiring a single generic method:

`Task FindSingleAsync(Guid id);`

We specify the `Type` that we'd like returned and pass it a `Guid` id and it returns the entity if found, `null` if otherwise.

Not too related here, but also notice that our service doesn't instantiate the `IDatabaseContext` dependency. Instead, it's passed in through constructor DI (Dependency Injection), ideally using an Inversion of Control container of some sort. By the way, if you've never used an IoC framework before, .NET Core ships with one baked in and ready to go that's absolutely worth a try.

Now, lets take a look at the one method we'll be testing in this class, `GetAsync`:

```c#
public async Task<User> GetAsync(Guid userId)
{
	if (userId == Guid.Empty)
	{
		throw new ArgumentException("Must specify a user id.", nameof(userId));
	}

	UserSql userSql = await this.DbContext.FindSingleAsync<UserSql>(userId);

	if (userSql == null)
	{
		throw new Exception($"User '{userId}' was not found.");
	}

	return UserSql.ToEntity(userSql);
}
```

It takes a `Guid` type argument representing the User we'd like to retrieve. There's three scenarios/outcomes we should be testing here:

- The method throws an `ArgumentException` if a valid `Guid` isn't passed in.
- The method throws an `Exception` if no user is returned from the database call.
- If a user is returned from the database, it returns the mapped model (of type `User`).

Before we dive in, lets discuss the tools and methodologies we have available to help us test more complicated classes and methods. In particular, classes that have external dependencies.

---

### Mocking. What is it and why do we need it?

Our first foray into unit testing focused on the simplest type of method that you can test. A simple and pure function that takes an input and returns a new string. Testing it was simply a matter of passing in various inputs and ensuring (asserting) that the method outputted the expected results.

The difference in this `GetAsync()` method is that it makes use of a class dependency, `IDatabaseContext`. As discussed, we can expect that by using it, we'll receive some SQL entity from some imaginary database. The issue is that if we were to test the actual execution of such outside dependencies, we would need to rely on whatever code it's executing to run successfully before the program continues with the rest of our method. Unit tests should be kept as tight as possible, focusing only on the logic and flow of the method itself. Calls made using dependencies should in most cases have their own separate tests to validate their functionality.

> Additionally, if you were to attempt really testing the `IDatabaseContext` implementation here, it would require your application to be wired up to a real database. That changes the scope of your testing and you're entering into the realm of *Integration Tests*. While integration tests are important, they are not within the scope of this tutorial. Unit tests should never need a live connection to a database (or any external system for that matter)

Thus, mocking is how we get around this scenario. We need the ability to get past these points in our code that references dependencies. We also need the ability to do so in a fashion that gives us control over its' mocked execution. Mocking libraries such as Moq help us setup the behavior of dependencies when our unit tests are calling them.

**How the basics of Moq works:**

You create a `Mock<T>` object, where T is the type of the dependency that needs to have its' behavior mocked. For example in code:

`var mockDatabase = new Mock<IDatabaseContext>();`

You can now proceed to Setup behaviors for your mocked object. For example, lets imagine there's a `SurveyService` with a `GetAsync()` method, returning a `Survey` type object. Lets also imagine that to test a particular scenario, you need the call on `GetAsync()` to return a new `Survey` type object, where its' `Enabled` property set to `false`. Here's how setting it up may look:

```c#
var mockSurveyService = new Mock<ISurveyService>();
mockSurveyService
	.Setup(ss => ss.GetAsync())
	.ReturnsAsync(new Survey { Enabled = false });
```

Now when the unit test is running and gets to the point of executing `GetAsync()`, the call will behave as you have set it up. Internally, what Moq does is try to override whatever method you're mocking using the setup you have configured. This bring up an important point:

Always try to test using interfaces of some dependency. It is, in theory, possible to test concrete implementations so long as the methods you're testing are virtual, but it would be horrible practice to change properties of your code just to make it testable. Using interfaces not only gives your codebase a loosely-coupled architecture, but it really is essential for unit testing. When mocking interfaces of the actual Type you'd like to test, you allow Moq to easily create the setups and mocks needed.

One other point here. In the above example, notice that we configure what's returned by passing in the object as the argument to `ReturnsAsync()`. Notice how we created one on the fly for it? This may or may not be okay depending on your particular test. If later in that same test we needed to access properties of the object returned, we'd have no reference to do so. In those kind of scenarios (which is pretty common when testing complex methods), create the object first, then pass it in as a reference. That way, you can access whatever properties you need for further testing/mocking later. For example:

```c#
var mockSurveyService = new Mock<ISurveyService>();

var survey = new Survey { Enabled = false };
mockSurveyService
	.Setup(ss => ss.GetAsync())
	.ReturnsAsync(survey);

// ... later in the same test
if (survey.Enabled)
{
	// do A
}
else
{
	// do B
}
```

Notice that we can now use the mocked return objects as desired later. When you'll need to do this will simply depend on the actual method you're testing.

Now that know how to do a basic mock using `Setup()`, we need to understand how we're going to test the actual service. How do we get an instance of the Service we'd like to test, using the mocked dependencies that we have configured?

Well, typically we'd create Mocks of all the service dependencies first (including all the Setups needed). Then, the service is initialized using the new operator, passing in the mocked dependencies. Lastly, you simply run the service method you're testing, appropriately capturing the result in a variable so you can run assertions against it. Example in code:

```c#
var mockDependency1 = new Mock<IDependency1>();
mockDependency1.Setup(d => d.Something()).Returns(something);

var mockDependency2 = new Mock<IDependency2>();
mockDependency2.Setup(d => d.Something()).Returns(somethingElse);

var serviceToTest = new SomeService(mockDependency1, mockDependency2);
var result = someService.MethodBeingTested();

Assert.Equal("some expected result here", result);
```

This works fine and is also pretty standard. However, I like to use a custom helper that makes use of .NET Core's Dependency Injection capabilities (in particular using `ServiceProvider` and `ServiceCollection`). How this DI stuff works is beyond the scope of this tutorial, but I'll explain how it's used in my testing. Also, you can always view the code for the helper called `TestingObject` in its' file found in the `TestsCore` project.

Alright, open up `UserServiceTests.cs` and take a look at the first method in the class, `GetTestingObject()`, copied below:

```c#
private TestingObject<UserService> GetTestingObject()
{
	var testingObject = new TestingObject<UserService>();
	testingObject.AddDependency(new Mock<IDatabaseContext>(MockBehavior.Strict));
	return testingObject;
}
```

The `TestingObject` helper is a generic class, where you specify the concrete type of whatever service or class you'll be testing. After initializing the testingObject, you add dependencies by passing in whatever Mocks that your service requires. The `TestingObject<T>`; class also comes with a `GetDependency()` method that'll be utilized in the actual tests. It grabs a reference to the mocked dependencies so you can configure the setups. Lastly, it has the method `GetResolvedTestingObject()` which returns an instance of whatever service/class you're testing, with the mocked dependencies properly injected. The usage will become clearer later when we get to the actual unit tests!

### Alrighty, back to testing the UserService

Lets take a look at the first unit test for this service:

```c#
[Fact]
public async Task GetAsync_InvalidArgument_ThrowsArgumentException()
{
	TestingObject<UserService> testingObject = this.GetTestingObject();

	UserService userService = testingObject.GetResolvedTestingObject();

	await Assert.ThrowsAsync<ArgumentException>(async () =>
		await userService.GetAsync(Guid.Empty));
}
```

The method name clearly describes what we're testing. We want to ensure that if the method throws an `ArgumentException` if passed an `Empty Guid`. This check happens at the very beginning of the method, and thus we don't need to configure any setups - we expect that the exception will be thrown before needing any of its' dependencies.

Thus, in this test, we can simply get the resolved `UserService` object, and then run an assertion using it. Up until this point, we've been using `Assert.Equal()`. xUnit actually comes with a bunch of useful assertion methods that make both creating and reading the tests nicer. The above is one such example, where we can easily test that a specific exception is being thrown by using the generic `ThrowsAsync<TException>` method (or `Throws<TException>` if testing a synchronous method).

Here's the next unit test:

```c#
[Fact]
public async Task GetAsync_UserNotFound_ThrowsException()
{
	TestingObject<UserService> testingObject = this.GetTestingObject();

	Guid userIdArg = Guid.NewGuid();

	var mockDbContext = testingObject.GetDependency<Mock<IDatabaseContext>>();
	mockDbContext
		.Setup(dbc => dbc.FindSingleAsync<UserSql>(It.Is<Guid>(id => id == userIdArg)))
		.ReturnsAsync(null);

	UserService userService = testingObject.GetResolvedTestingObject();
	await Assert.ThrowsAsync<Exception>(async ()
		=> await userService.GetAsync(userIdArg));
}
```

Here we start by getting a new `Guid` that we'll pass as an argument. Then, we grab a reference to the `IDatabaseContext` mock from our testing object helper.

We're testing that an `Exception` is thrown if the database call returns null, so we'll have to set our mock up appropriately. We use Moq's `Setup` method and code the logic such that when the `IDatabaseContext`'s `FindSingleAsync()` method is called with the argument passed in equals the one we created, it will return `null`.

> Note that this particular Setup will return `null` every time given that condition. If the method being tested made the same call later, but we needed a different result returned, this would be problematic as we'd get `null` every time. Fortunately, Moq provides another setup option: `SetupSequence()`. Be sure to look into it if your tests require different return results depending on when a dependency's method is called.

Lastly, we get the grab the service we're testing, and run another assertion on it to check that an `Exception` is indeed being thrown. Note that we're passing in the same `Guid` as an argument that we also used in the Moq setup. This is crucial because we need to mimic the logic of the actual method we're testing.

Our final test case ensures that on success, we get the correct and expected result:

```c#
[Fact]
public async Task GetAsync_Success_ReturnsCorrectResult()
{
	TestingObject<UserService> testingObject = this.GetTestingObject();
	
	Guid userIdArg = Guid.NewGuid();

	var mockDbContext = testingObject.GetDependency<Mock<IDatabaseContext>>();

	var userSql = new UserSql
	{
		FirstName = "Mary",
		LastName = "Jane"
	};
	mockDbContext
		.Setup(dbc => dbc.FindSingleAsync<UserSql>(It.Is<Guid>(id => id == userIdArg)))
		.ReturnsAsync(userSql);

	UserService userService = testingObject.GetResolvedTestingObject();
	User result = await userService.GetAsync(userIdArg);

	Assert.Equal(userSql.FirstName, result.FirstName);
	Assert.Equal(userSql.LastName, result.LastName);
}
```

Similar to our previous test, but notice that our setup on the database call now returns a `UserSql` object, in which we've also set some names. We get the result of calling the service method, and then run some assertions to ensure that the first and last name's are what we expect.

I'd like to note here that there's not necessarily a best way to write assertions at times. We could've written our assertion here as a simple Assert.NotNull(result) to ensure that we're getting back a valid `User` object. However, in my example I've opted to compare the names and in the process, we get extra validation that our mapping method is working correctly. Generally speaking it's not good to test too many things at once, but I typically don't write unit tests for my mapping methods due to their simplistic nature. Just go about this stuff at your best discretion!

---

## Time for some Practice!

I know the examples we've gone through are very basic, but believe it or not, we already have enough of a foundation now to start testing anything. Typically the unit tests will be more complex if the methods they're testing are complex as well. The main goal is to test every code path the execution of a method can take so strive for 100% coverage when possible. I can admit that sometimes it gets tedious writing the most basic of tests that cover an overly simplistic scenario, but it's better to spend that extra minute covering it anyways. You just never know - that extra minute spent could potentially save you a lot more time in the future if something were to break because you didn't have that particular case covered.

To end this tutorial, I'm going to have you try testing the `ReportService` using what we've learned. There's really nothing better in ingraining something recently read than spending some time doing it yourself. Here's the service copied below:

```c#
public class ReportService : IReportService
{
	private IUserService UserService { get; }
	private IDatabaseContext DbContext { get; }

	public ReportService(
		IUserService userService,
		IDatabaseContext dbContext)
	{
		this.UserService = userService;
		this.DbContext = dbContext;
	}

	public async Task<ReportMetadata> GetAsync(Guid reportId)
	{
		if (reportId == Guid.Empty)
		{
			throw new ArgumentException("Report id must be provided", nameof(reportId));
		}

		ReportMetadataSql reportSql = await this.DbContext.FindSingleAsync<ReportMetadataSql>(reportId);
		if (reportSql == null)
		{
			throw new Exception($"Report '{reportId}' was not found.");
		}


		// if report is older than 3 years, dont return it
		if (reportSql.Created.AddYears(3) < DateTime.UtcNow)
		{
			throw new Exception($"Report '{reportId}' is too old and will not be retrieved.");
		}

		UserSql ownerSql = await this.DbContext.FindSingleAsync<UserSql>(reportSql.OwnerId);
		if (ownerSql == null)
		{
			throw new Exception($"User '{reportSql.OwnerId}' not found.");
		}


		// if owner of report is disabled, dont return it
		if (!ownerSql.Enabled)
		{
			throw new Exception($"Report '{reportId}' is owned by a disabled user and will not be retrieved.");
		}

		User owner = UserSql.ToEntity(ownerSql);
		string authorFullName = UserUtil.GetFullName(owner.FirstName, owner.LastName);

		// modify title based on whether it's been updated
		if (reportSql.LastUpdated.HasValue)
		{
			reportSql.Title += " (Revision)";
		}
		else
		{
			reportSql.Title += " (Original)";
		}

		return new ReportMetadata
		{
			Id = reportSql.Id,
			LastUpdated = reportSql.LastUpdated,
			LastRevisionById = reportSql.LastRevisionById,
			Title = reportSql.Title,
			Created = reportSql.Created,
			AuthorFullName = authorFullName
		};
	}
}
```

Spend some time looking over the `GetAsync` method and come up with a list of potential test cases. Then, open up the `ReportServiceTests.cs` file and compare your cases with the ones I've created to help you get started. If there's any difference, make sure you look over the method again and really understand why they're there. Finally, take your time writing out the actual unit tests, ensuring that they build and pass.

Below are my implementations for the tests. Feel free to reference them as needed, but definitely put in some effort trying them out yourself first!

---

## My Solutions

Lets start with the code paths we should be testing for:

- `ArgumentException` is thrown if an `empty Guid` is passed in.
- `Exception` is thrown if the `Report` isn't found.
- `Exception` is thrown if the `Report` is older than three years.
- `Exception` is thrown if the report's `Owner` isn't found.
- `Exception` is thrown if the report's `Owner` is disabled.
- The report has "(Revision)" appended if it has been updated.
- The report has "(Original)" appended if it hasn't been updated.

I purposely made up many condition cases so we can get some more thorough practice in on testing this. At this point, we've already gone over the material needed in-depth to cover the above seven cases so I'm simply going to post my solution below:

```c#
public class ReportServiceTests
{
	private TestingObject<ReportService> GetTestingObject()
	{
		var testingObject = new TestingObject<ReportService>();
		testingObject.AddDependency(new Mock<IUserService>(MockBehavior.Strict));
		testingObject.AddDependency(new Mock<IDatabaseContext>(MockBehavior.Strict));
		return testingObject;
	}

	[Fact]
	public async Task GetAsync_InvalidArgument_ThrowsArgumentException()
	{
		TestingObject<ReportService> testingObject = this.GetTestingObject();

		ReportService reportService = testingObject.GetResolvedTestingObject();

		await Assert.ThrowsAsync<ArgumentException>(async () =>
			await reportService.GetAsync(Guid.Empty));
	}

	[Fact]
	public async Task GetAsync_ReportNotFound_ThrowsException()
	{
		TestingObject<ReportService> testingObject = this.GetTestingObject();

		Guid reportIdArg = Guid.NewGuid();

		var mockDbContext = testingObject.GetDependency<Mock<IDatabaseContext>>();
		mockDbContext
			.Setup(m => m.FindSingleAsync<ReportMetadataSql>(It.Is<Guid>(id => id == reportIdArg)))
			.ReturnsAsync(null);

		ReportService reportService = testingObject.GetResolvedTestingObject();
		await Assert.ThrowsAsync<Exception>(async ()
		   => await reportService.GetAsync(reportIdArg));
	}

	[Fact]
	public async Task GetAsync_ExpiredReport_ThrowsException()
	{
		TestingObject<ReportService> testingObject = this.GetTestingObject();

		Guid reportIdArg = Guid.NewGuid();

		var mockDbContext = testingObject.GetDependency<Mock<IDatabaseContext>>();

		var report = new ReportMetadataSql
		{
			Created = DateTime.UtcNow.AddYears(-4)
		};
		mockDbContext
			.Setup(m => m.FindSingleAsync<ReportMetadataSql>(It.Is<Guid>(id => id == reportIdArg)))
			.ReturnsAsync(report);

		ReportService reportService = testingObject.GetResolvedTestingObject();
		await Assert.ThrowsAsync<Exception>(async ()
		   => await reportService.GetAsync(reportIdArg));
	}

	[Fact]
	public async Task GetAsync_OwnerNotFound_ThrowsException()
	{
		TestingObject<ReportService> testingObject = this.GetTestingObject();

		Guid reportIdArg = Guid.NewGuid();

		var mockDbContext = testingObject.GetDependency<Mock<IDatabaseContext>>();

		var report = new ReportMetadataSql
		{
			Created = DateTime.UtcNow.AddYears(-2),
			OwnerId = Guid.NewGuid()
		};
		mockDbContext
			.Setup(m => m.FindSingleAsync<ReportMetadataSql>(It.Is<Guid>(id => id == reportIdArg)))
			.ReturnsAsync(report);

		mockDbContext
			.Setup(m => m.FindSingleAsync<UserSql>(It.Is<Guid>(id => id == report.OwnerId)))
			.ReturnsAsync(null);

		ReportService reportService = testingObject.GetResolvedTestingObject();
		await Assert.ThrowsAsync<Exception>(async ()
		   => await reportService.GetAsync(reportIdArg));
	}

	[Fact]
	public async Task GetAsync_OwnerDisabled_ThrowsException()
	{
		TestingObject<ReportService> testingObject = this.GetTestingObject();

		Guid reportIdArg = Guid.NewGuid();

		var mockDbContext = testingObject.GetDependency<Mock<IDatabaseContext>>();

		var report = new ReportMetadataSql
		{
			Created = DateTime.UtcNow.AddYears(-2),
			OwnerId = Guid.NewGuid()
		};
		mockDbContext
			.Setup(m => m.FindSingleAsync<ReportMetadataSql>(It.Is<Guid>(id => id == reportIdArg)))
			.ReturnsAsync(report);

		var owner = new UserSql
		{
			Enabled = false
		};
		mockDbContext
			.Setup(m => m.FindSingleAsync<UserSql>(It.Is<Guid>(id => id == report.OwnerId)))
			.ReturnsAsync(owner);

		ReportService reportService = testingObject.GetResolvedTestingObject();
		await Assert.ThrowsAsync<Exception>(async ()
		   => await reportService.GetAsync(reportIdArg));
	}

	[Fact]
	public async Task GetAsync_Success_Updated_ReturnsCorrectResult()
	{
		TestingObject<ReportService> testingObject = this.GetTestingObject();

		Guid reportIdArg = Guid.NewGuid();

		var mockDbContext = testingObject.GetDependency<Mock<IDatabaseContext>>();

		var report = new ReportMetadataSql
		{
			Created = DateTime.UtcNow.AddYears(-2),
			OwnerId = Guid.NewGuid(),
			LastUpdated = DateTime.UtcNow,
			Title = "Report Title"
		};
		mockDbContext
			.Setup(m => m.FindSingleAsync<ReportMetadataSql>(It.Is<Guid>(id => id == reportIdArg)))
			.ReturnsAsync(report);

		var owner = new UserSql
		{
			Enabled = true,
			FirstName = "Mary",
			LastName = "Jane"
		};
		mockDbContext
			.Setup(m => m.FindSingleAsync<UserSql>(It.Is<Guid>(id => id == report.OwnerId)))
			.ReturnsAsync(owner);

		ReportService reportService = testingObject.GetResolvedTestingObject();

		ReportMetadata result = await reportService.GetAsync(reportIdArg);

		Assert.EndsWith("(Revision)", result.Title);
	}

	[Fact]
	public async Task GetAsync_Success_NotUpdated_ReturnsCorrectResult()
	{
		TestingObject<ReportService> testingObject = this.GetTestingObject();

		Guid reportIdArg = Guid.NewGuid();

		var mockDbContext = testingObject.GetDependency<Mock<IDatabaseContext>>();

		var report = new ReportMetadataSql
		{
			Created = DateTime.UtcNow.AddYears(-2),
			OwnerId = Guid.NewGuid(),
			Title = "Report Title"
		};
		mockDbContext
			.Setup(m => m.FindSingleAsync<ReportMetadataSql>(It.Is<Guid>(id => id == reportIdArg)))
			.ReturnsAsync(report);

		var owner = new UserSql
		{
			Enabled = true,
			FirstName = "Mary",
			LastName = "Jane"
		};
		mockDbContext
			.Setup(m => m.FindSingleAsync<UserSql>(It.Is<Guid>(id => id == report.OwnerId)))
			.ReturnsAsync(owner);

		ReportService reportService = testingObject.GetResolvedTestingObject();

		ReportMetadata result = await reportService.GetAsync(reportIdArg);

		Assert.EndsWith("(Original)", result.Title);
	}
}
```

If you're not interested in writing the tests yourself, run this command to pull my solutions into your working branch:

`git pull origin solutions`

---

## Wrapping Things Up

That's about it for this tutorial. I hope you've found some useful bits of information from walking through, and hopefully it's also encouraged yourself to start testing regularly.

Remember, the concept of most unit tests are pretty simple at the core. Test all the code paths the method can take and ensure that the results are as you would expect. You'll probably run into times where testing a huge and complex might get nasty. That may be a sign for you to refactor and break up your gigantic method into smaller focused chunks. Or, sometimes, there's really nothing to be done than to power through and write the complex tests anyways.

It doesn't matter whether you're testing first using Test Driven Development (TDD) or writing them afterwards. The only real important thing is that they're being written at some point. And ideally you'd be writing them early while the code you've written is still fresh in your head.

Leave comments or questions and I'll try to get to them as I get time. Happy testing!