# iOS Testing at XXX

The practice of automating tests is important for all apps, but specially important on XXX due to our large codebase and constant growth. It's madness to expect features to be tested by hand in an app that will have milions of users, and so we make sure that everything fully works before merging in the form of automated tests.

Here's a guideline that you should follow when developing anything here:

* If you're developing a feature, write **clear unit tests** that give no doubt that the feature works as expected.
* If you're developing a bug fix, write **clear unit tests** that will prevent this bug from comming back to haunt you in the future. This is called a **regression test**.

# Stack

You should exclusively use the following libraries to write tests:
* Unit Tests: `XCTestCase` 
* Snapshot Tests: [SnapshotTesting](https://github.com/pointfreeco/swift-snapshot-testing)

# Best practices

* Don't forget about the [F.I.R.S.T.](https://hackernoon.com/test-f-i-r-s-t-65e42f3adc17) principles.
* All **SnapshotTests** should be recorded using the **iPhone8** simulator, since it's the same configured for our CI.
* Each individual test should be **fast**. Being that, avoid using sleeps and dispatches that waits for things to happen. You should *mock* your feature's components instead to make things return faster.
* You should **never have networking** events in unit tests. If it does, it is **not** a unit test! Mock your components to return hardcoded instances.
* Each test should be completely independent from the others. If the tests were to run in a random order, would your test pass? If the answer is no, it's not a unit test. Tests should not produce side effects for other tests. You can usually make sure this won't happen by properly mocking your test's components.
* Tests should not depend on things outside of your control, like fixed timezones and locales. These cases should be handled by the app itself, and so the test needs to be neutral regarding this.
* Avoiding testing multiple things in a single test case. Each component should have its own test for modularity reasons.
* Be descriptive - the test's name and contents should be enough to understand what's being tested. Comments should only be used for clarity of intent.
* **Avoid using force unwrapping.** A test that crashes will not only make it more difficult to understand what exactly failed, but it will also prevent other tests from the suite from running. This can hide the existence of additional issues that would show up if all tests were executed normally, lowering your productivity. There are a few alternatives, including:
    * Asserting directly on the optional value:
    ```swift
    XCTAssertEqual(try? projectInfo.getBuildNumber(), "123")
    ```
    * Guarding with a `XCFail` call:
    ```swift
    guard let myValue = myValue else {
        XCFail("myValue doesn't exist!")
    }
    XCTAssert(...)
    ```

# About the naming and contents of tests

We use G/W/T (Given, When and Then) to define the contents of our tests. In general, this means that your tests have a clear singular objective that is asserted after creating and setting up the environment. 

- **Given:** is where you setup the environment for your test.
- **When:** is what needs to happen for you validate an expected outcome. Basically, it's where you are going to call the method that is being tested.
- **Then:** here you compare the outcome received with the expected one.

Here's an example:

```swift
func test_whenProductIsAddedToCart_cartItemsShouldContainsProduct() {
    // Given
    let product = Product()
    let cart = Cart()
    // When
    cart.add(product: product)
    // Then
    XCTAssertTrue(cart.items.contains(product))
}
```

This should be your general guideline for developing tests, but you might find cases where this rule can't be followed by the book. 

To make the tests easier to understand, we have a pattern on how tests should be named. The name of your test should consist of the following pieces of information divided by an underscore:

1. What is being tested
2. Under what circumstances (if it's relevant to the test)
3. What is the expected result

Here are some examples:

* `test_profileView_whenLoading_hidesContent`
* `test_analyticsPlugin_whenReceivingNotification_triggersCallback`
* `test_clientService_isLazy` (a case where the circumstance doesn't matter)

# Components for developing tests

## Doubles (Test Objects)

Double are components used to help us isolate the subject under test from any side effect. 

There are 5 types of doubles:

- **Dummy**: Objects used to fill parameters and configure different scenarios for your test. *You don't use this object to do asserts*.
- **Fake**: Objects with working implementation but usually with shortcuts or a reduced logic (ex.: in-memory database)
- **Stubs**: Objects that respond calls with values that were pre-configured conforming with the scenario of our test.
- **Spy**: Objects used to record information about how they were consumed.
- **Mocks**: Objects pre-configured with expectations about how they will be consumed and encapsulate the verification itself.

Since Mocks are complex classes, the community always rely on Frameworks. This kind of frameworks uses a feature present in many languages called Reflection. Unfortunately, this feature is not present on Swift in the form as this kind of frameworks require. So all the frameworks for Mock that exist in Swift are limited and require some boilerplates. None of them is ready for use and clean like Mockito (Java), jMock (Java), etc.

For that reason, when we write - manually - our doubles in the project, it is most of the time a **Stub** or **Spy**. And this is not a problem.

## Dummies
A **Dummy** is basically a test double that **doesn't do anything**.
You might use it as a placeholder for something that you need in order to setup your tests, but won't have any effect on them.

**Example:**
```swift
protocol UserServiceProtocol {
    func login(_ user: User, then: (Result<Void, Error>) -> Void)
}

struct UserServiceDummy: UserServiceProtocol {
    func login(_ user: User, then: (Result<Void, Error>) -> Void) {
        // Do nothing.
    }
}
```

## Fakes
A **Fake** is a double that aims to decrease the complexity of the test, simplifying the implementation, and mimicking the real behavior. 
For example, when creating an integration test for something that depends on a local database, you can make a fake that simplifies it and store the data in memory.

**Example:**
```swift
struct Movie {
    let id: String
    let name: String
    let posterURL: String
}

protocol FavoritesManagerProtocol {
    func add(_ movie: Movie) throws
    func remove(_ movie: Movie) throws
    var favorites: [Movie] { get }
}

final class MovieListViewModel {
    private let favoritesManager: FavoritesManagerProtocol
    
    init(favoritesManager: FavoritesManagerProtocol) {
        self.favoritesManager = favoritesManager
    }
    
    func addFavorite(_ movie: Movie) -> Bool {
        do {
            try favoritesManager.add(movie)
            return true
        } catch {
            return false
        }
    }
}
```
If we are testing something that uses the ** FavoritesManagerProtocol**, we shouldn't use the real local database, since it's could be slow and require other setups that would make our test more complex.
In this case, we could create a fake for ** FavoritesManagerProtocol**, like the example below:
```swift

final class FavoritesManagerFake: FavoritesManagerProtocol {
    var favorites: [Movie] = []
    
    func add(_ movie: Movie) throws {
        guard !favorites.contains(where: { $0.id == movie.id } ) else {
            throw NSError(domain: "FavoritesManagerFake", code: -1, userInfo: nil)
        }
        favorites.append(movie)
    }
    
    func remove(_ movie: Movie) throws {
        guard let index = favorites.firstIndex(where: { $0.id == movie.id }) else {
            throw NSError(domain: "FavoritesManagerFake", code: -2, userInfo: nil)
        }
        favorites.remove(at: index)
    }
}

// Usage
final class MovieListViewModelTests: XCTestCase {
    func test_toggleFavorite_whenMovieIsNotFavorite_itShouldReturnTrue() {
        // Given
        let favoritesManagerFake = FavoritesManagerFake()
        favoritesManagerFake.favorites = []
        
        let movieMock: Movie = .init(id: "id", name: "name", posterURL: "posterURL")
        
        let sut = MovieListViewModel(favoritesManager: favoritesManagerFake)
        
        // When
        let result = sut.addFavorite(movieMock)
        
        // Then
        XCTAssertTrue(result)
    }
}
```

## Stubs
**Stub** is a kind of test double that you use to control the outcome of the said dependency.

**Example:**
```swift

struct Post {
    let title: String
    let text: String
}

protocol PostsServiceProtocol {
    func fetchAll(then: (Result<[Post], Error>) -> Void)
}

final class UserFeedViewModel {
    
    private let postsService: PostsServiceProtocol
    private(set) var numberOfPosts: Int = 0
    
    init(postsService: PostsServiceProtocol) {
        self.postsService = postsService
    }
    
    func loadData(then: @escaping () -> Void) {
        postsService.fetchAll { [weak self] result in
            let numberOfPosts = (try? result.get())?.count ?? 0
            self?.numberOfPosts = numberOfPosts
            then()
        }
    }
    
}
```

Considering that you want to test if the **numberOfPosts** changed, you can do something like this:
```swift
final class PostsServiceStub: PostsServiceProtocol {

    var fetchAllResultToBeReturned: Result<[Post], Error> = .success([])
    func fetchAll(then: (Result<[Post], Error>) -> Void) {
        then(fetchAllResultToBeReturned)
    }

}

// Usage
final class UserFeedViewModelTests: XCTestCase {
    
    func test_fetchAll_shouldReturnTheCorrectAmountOfPosts() {
        // Given
        let postsServiceStub = PostsServiceStub()
        let stubbedPosts: [Post] = [
            .init(title: "Post 1", text: "Post Text 1"),
            .init(title: "Post 2", text: "Post Text 2")
        ]
        postsServiceStub.fetchAllResultToBeReturned = .success(stubbedPosts)
        let sut = UserFeedViewModel(postsService: postsServiceStub)
        let initialNumberOfPosts = sut.numberOfPosts
        
        // When
        let loadDataExpectation = expectation(description: "loadDataExpectation")
        sut.loadData {
            loadDataExpectation.fulfill()
        }
        wait(for: [loadDataExpectation], timeout: 1.0)
        
        // Then
        XCTAssertNotEqual(initialNumberOfPosts, sut.numberOfPosts)
    }

}
```
**Notes:**
* Consider defining a pattern when naming the stub properties. 
I like to use `method name + ToBeReturned`, as shown in the example above.

* Avoid setting up the stubbed values on the stub's **init** method. This way, you can change it after initialization, and avoid duplication on your test code.

## Spies
**Spy** is a test double that you can use to **inspect the properties of some dependency** used by the **SUT** (System Under Test).

**Example:**
```swift
struct User: Equatable {
    let name: String
    let password: String
}

protocol UserServiceProtocol {
    func login(_ user: User, then: (Result<Void, Error>) -> Void)
}
protocol SafeStorageProtocol {
    func storeUserData(_ user: User)
}

final class LoginViewModel {
    private let userService: UserServiceProtocol
    private let safeStorage: SafeStorageProtocol
    
    init(
        userService: UserServiceProtocol,
        safeStorage: SafeStorageProtocol
    ) {
        self.userService = userService
        self.safeStorage = safeStorage
    }
    
    func performLoginForUser(_ user: User) {
        userService.login(user) { [weak self] result in
            switch result {
            case .success:
                self?.saveLastUser(user)
            case let .failure(error):
                // do something with the error
                debugPrint(error)
            }
        }
    }
    
    private func saveLastUser(_ data: User) {
        safeStorage.storeUserData(data)
    }
}
```
To verify that the last user logged in was properly saved, we can use a **Spy** like shown below:
```swift
final class UserServiceStub: UserServiceProtocol {
    
    var loginResultToBeReturned: Result<Void, Error> = .success(())
    func login(_ user: User, then: (Result<Void, Error>) -> Void) {
        then(loginResultToBeReturned)
    }
}

final class SafeStorageSpy: SafeStorageProtocol {
    
    private(set) var storeUserDataCalled = false
    private(set) var userPassed: User?
    func storeUserData(_ user: User) {
        storeUserDataCalled = true
        userPassed = user
    }
    
}

// Usage
final class LoginViewModelTests: XCTestCase {
    
    func test_fetchAll_shouldReturnTheCorrectAmountOfPosts() {
        // Given
        let userServiceStub = UserServiceStub()
        userServiceStub.loginResultToBeReturned = .success(())
        let safeStorageSpy = SafeStorageSpy()
        let sut = LoginViewModel(
            userService: userServiceStub,
            safeStorage: safeStorageSpy
        )
        let userMock = User(name: "name", password: "password")
        
        // When
        sut.performLoginForUser(userMock)
        
        // Then
        XCTAssertTrue(safeStorageSpy.storeUserDataCalled)
        XCTAssertEqual(userMock, safeStorageSpy.userPassed)
    }
    
}
```
**Spy Property conventions:**
* if it is a method: Name + Called
* if it is a parameter: Name + Passed
* if it is a computed property: Name + Accessed
* if it is a method and # of calls is relevant: Name + Count
* all of them should be private(set)
* a good location to put them is just above the corresponding function.

**Note:** You can create a **Spy** that is also a **Stub**, and combine their power to write your tests.

## Mocks
> **Mocks** are pre-programmed with expectations which form a specification of the calls they are expected to receive. They can throw an exception if they receive a call they don’t expect and are checked during verification to ensure they got all the calls they were expecting.

**Source:** [https://martinfowler.com/bliki/TestDouble.html](https://martinfowler.com/bliki/TestDouble.html)

Mocks can be one of the double types mentioned above, like a Spy or Stub, that handles some of the validations and asserts by themselves. We won't use them much...


## Fixtures
Fixtures are basically custom initializers to simpify the construction of objects needed for our tests.
**Example:**
```swift
struct User {
    let id: Int
    let name: String
    let lastName: String
    let nickname: String?
}

final class ProfileViewModelProtocol {
    func getUserName() -> String
}

final class ProfileViewModel {
    private let user: User
    init(user: User) {
        self.user = user
    }   
    
    func getUserName() -> String {
        guard let nickname = user.nickname else { return user.name }
        return nickname
    }
}

final class ProfileViewModelTests: XCTestCase {
    
    // Without fixtures
    func test_getUserName_whenNickNameIsNotNil_itShouldReturnNickname_1() {
        // Given
        let userMock: User = .init(
            id: 1,
            name: "my_name",
            lastName: "last_name",
            nickname: "Zé"
        )
        let sut = ProfileViewModel(user: userMock)
        // When
        let returnedValue = sut.getUserName()
        // Then
        XCTAssertEqual(returnedValue, userMock.nickname)
    }
}
```

Let's suppose that the  `User` struct has even more properties, but your tests are interested only in the `nickname` property.
You could use a `fixture` to make it clear what is being tested, while also simplifying your test...

The fixture for user, could be this:
```swift
extension User {
    static func fixture(
        id: Int = 1,
        name: String = "my_name",
        lastName: String = "last_name",
        nickname: String? = nil
    ) -> User {
        return .init(
            id: id,
            name: name,
            lastName: lastName,
            nickname: nickname
        )
    }
}
```

After creating the fixture, our test can be refactored to:
```swift
final class ProfileViewModelTests: XCTestCase {
    func test_getUserName_whenNickNameIsNotNil_itShouldReturnNickname_1() {
        // Given
        let userMock: User = .fixture(nickname: "Zé")
        let sut = ProfileViewModel(user: userMock)
        // When
        let returnedValue = sut.getUserName()
        // Then
        XCTAssertEqual(returnedValue, userMock.nickname)
    }
}
```

## Snapshot Testing

We are current using the library [SnapshotTesting](https://github.com/pointfreeco/swift-snapshot-testing).
There is a lot of alternatives to this library, but the simplicity and some others features like testing resources made [SnapthostTesting](https://github.com/pointfreeco/swift-snapshot-testing) a great choice for us.

> You can go to his GitHub page to learn more on how to write snapshot tests. It's pretty simple.

**Note:** There is an important issue happening when we execute our tests in CI environment. You can track the issue [here](https://github.com/pointfreeco/swift-snapshot-testing).
We don't have any clever solution to this issue yet. So, if you get stuck with tests that don't pass in CI, just comment test case.
