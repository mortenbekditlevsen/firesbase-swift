# Firebase and Swift

## Goals
The goal of any good tech project is naturally complete world domination. I would, however, be satisfied with being able to use Swift anywhere. On the server, on the Apple platforms, on Linux, on Android and on Windows. I find Swift to be incredibly powerful in it's focus on safety and readability - and I believe that precisely the type of safety that Swift delivers is exactly what is needed in many, many applications to combat various vulnerabilities that are much too easy to create using other languages.

I happen to like the Firebase suite of tools quite a lot and I think that they deliver on the promise of making it easy to build mobile and web apps. Firebase makes complex, complicated and tedious tasks easier and allow you to focus on modelling your domain and building functionality that directly adds value to your end users. Stuff like authentication, data synchronisation, data synchronisation in real-time, offline caching and serverside scalability are just some of the things that you get using some of the Firebase tools.

Firebase has great client tooling for many different platforms - including Swift. But there is more work to do - and there's an entire new world of possibilities ahead if Firebase for instance did not only expose Swift apis, but was in fact implemented in pure, cross-platform Swift - able to run on many platforms - including of course server side through Firebase functions.

## Previous work
For some reason I have been focusing a lot on the concept of codability in Swift. Both in general, in Foundation and in Firebase. 

I think that the `Codable` protocol and sorrounding design is very elegant - and I love the thoughts behind encoders and decoders being decoupled from the entities that are being encoded and decoded. 

For that reason I have been working to improve the overall ergonomics of `Codable` in Swift in general, and also contributed quite a bit to get `Codable` support into the Firebase components where it makes the most sense.

### Swift / `Codable`: Leave `Dictionary` keys alone 
My first bigger contribution to Swift is still 'just' a bug fix, but one that required a great deal of discussion on the Swift forums back in 2018 where `Codable` was still rather new.

Perhaps some still remember, that if you used a snake case `keyEncodingStrategy` on the `JSONEncoder`, this would also encode keys in your own dictionaries.

For instance if you had an entity:

```swift
struct User: Codable {
  var name: String
  var chatRooms: [String: ChatRoom]
}
```
Where the `chatRooms` is a `Dictionary` from some internal, perhaps randomly generated `String` to a `ChatRoom`.

Then if you encoded that as follows:

```swift
let user = User(name: "Morten", chatRooms: ["xX3mc_axkK": ChatRoom(...)])
let encoder = JSONEncoder()
encoder.keyEncodingStrategy = .convertToSnakeCase
let data = try encoder.encode(user)
```

Then the _keys_ in the `chatRooms` `Dictionary` would be transformed with the 'convertToSnakeCase' strategy - giving something like `"x_x3mc_axk_k"` for the above key. Decoding the same key would result in `"xX3mcAxkK"` which is probably not what most people expected.

So the discussion on the Swift forums were about whether or not `Dictionary` content should be considered to be _data_ that should not be altered using key coding strategies or not. Fortunately it was agreed that the keys should be left alone so that IDs that might mean something in your application doesn't get mixed up in seemingly random cases based on their containing capital letters and underscores...

Here's the first in a long series of commits leading to a fix for this issue:

[apple/swift Make JSONEncoder and JSONDecoder circumvent keyEncodingStrategies](https://github.com/apple/swift/commit/8a3bc0b73b3c10ad2359c6835af904afba0260b3)

### Swift / `Codable`: SE-0320
My second contribution to Swift and the `Codable` system ended up being a full blown Swift Evolution Proposal that was accepted and included in Swift 5.6.

It could generally also be considered a bug of the initial `Codable` implementation, but after many years of Swift it was too late to change the behavior - which would also mean that the same code would behave differently on platforms including the fix than on older ones.

The issue here is that only `Dictionary` with keys of `String` and `Int` are encoded into dictionary-like structures. All other key types would result in encoding arrays of pairs of the keys and values.

The reasoning was that if the key is not a `String` or an `Int`, then it could encode to a type that cannot be used as key in the serialized format.

The error in the reasoning was to only accept `String` and `Int` and not all types that encode to `String` or `Int`.

Since the ship had sailed on implementing the above as a fix, a new protocol was introduced in Swift 5.6. If a type conforms to `CodingKeyRepresentable`, any `Dictionary` using that type as a key will encode as a dictionary and not as an array. The existence of the protocol then also becomes the API boundary for when you can adopt this functionality. 

[apple/swift-evolution SE-0320](https://github.com/apple/swift-evolution/blob/main/proposals/0320-codingkeyrepresentable.md)

### Support for `Codable` encoding and decoding in RTDB
This is actually pretty straight forward - and something that I have been using many years prior to contributing it to `firebase-ios-sdk`.

Since the RTDB apis produce and consume either plain values or nested dictionaries and arrays of plain values, this conceptually looks very similar to JSON.

The `JSONEncoder` and `JSONDecoder` provided in `Foundation` can almost be used directly for this purpose - with the only change being that they need to skip the final step of using `JSONSerialization` to serialize to and from `Data`.

So the `Codable` functionality that you get in Firebase today is actually based very closely on the open source `JSONEncoder` and `JSONDecoder` pairs from `Foundation`.

The implementation can be found here:

[Swift extension for RTDB](https://github.com/firebase/firebase-ios-sdk/commit/0175b86cb5226910511e272f5594bf9690076de0)

### Support for `Codable` encoding and decoding in the Functions apis
A while after the RTDB extension, I helped refactoring this to a shared component in Firebase so that it could be used in the `FirebaseFunctions` APIs as well.

[FirebaseFunctions Swift](https://github.com/firebase/firebase-ios-sdk/commit/aea5e89e9e19eb71a4ac429756ed5ce169f9ac36)

### Use shared `Codable` encoding and decoding support for Firestore
Finalizing the `Codable` refactor was just released with v10.0 of the `firebase-ios-sdk`; namely replacing a custom encoder and decoder pair in Firestore with the shared encoder and decoder.

The benefits for the firebase-ios-sdk code base was both to be able to throw away a lot of code, but also to allow for the same encoding and decoding 'strategies' that you might be familiar with from `JSONEncoder` and `JSONDecoder`.

For instance you could not do any conversion of keys to snake case before v10.0 without requiring manual `CodingKey` implementations for all your entities.

[Make Firestore use FirebaseDataEncoder and FirebaseDataDecoder](https://github.com/firebase/firebase-ios-sdk/commit/65f6cfa0bf6ea502ceada244c5fa912f29669661)

## Ongoing work

### Roundtripping key coding strategies (Swift / `Codable` / `Foundation`)

My next bigger attempt at improving the `Codable` system in Swift has to do with the concrete implementation of `JSONEncoder` and `JSONDecoder`. Unfortunately these are not part of the community driven Swift project, but rather developed and maintained by Apple.

The issue is that not all encoding and decoding of keys roundtrip successfully when using the `.convertToSnakeCase` `keyEncodingStrategy` and ditto when decoding.

The standard example of this is:

```swift
struct User: Codable {
   var name: String
   var imageURL: URL?
}

```
Encoding `imageURL` with snake case key conversion gives `image_url`, but decoding the same attempts to look up the converted key: `imageUrl`, since information is lost when performing the two conversions.

This leads developers to include standard workarounds for capitalized abbreviations like:

```swift
struct User: Codable {
    enum CodingKeys: String, CodingKey {
        case name
        case imageURL = "imageUrl"
    }
    var name: String
    var imageURL: URL?
}
```
Having to add this key is a 'leaky abstraction'. The knowledge about a weird behavior of a specific use case forces you to 'pollute' your model with extra code that just looks off - and might be incompatible with other encoding use cases where you might not want and snake case key conversion and would actually prefer the key to remain `"imageURL"`.

The issue is that in the current implementation, there are two transformations - and the transformations are 'lossy'. You lose information about the original key in the encoded JSON.

There is an extremely elegant solution to the issue by a contributer named Norio Nomura, who suggested an alternative `keyEncodingStrategy` and `keyDecodingStrategy` where only one transformation exists - the one that goes from the key to the snake case representation. This works since most API when decoding actually has knowledge about the original key.

For instance in this custom decodable implementation:

```swift
...
public init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)
    self.name = try container.decode(String.self, forKey: .name)
    self.imageURL = try container.decodeIfPresent(URL.self, forKey: .imageURL)
}
```

As you can see, the key is passed to the API, so here we can apply the transformation from key to snake case representation directly on the key.

**Problem solved!? :-)**

Well, almost. There is also an API called `allKeys` on `KeyedDecodingContainer` that should return all of the keys in the decoding container. In this case we have no 'original key' to ask for, and thus we need a second transformation in the direction of encoded key to key.

This means that `allKeys` would either have to return an empty list or it would need to attempt to recover some of the keys using the exact conversion from snake case that we are trying to eliminate.

That could perhaps be acceptable since it could be said to be an improvement over today's APIs, but it might be confusing why this API has some flaws that the key based ones do not.

That said, I think that there is a deeper 'flaw' with the `allKeys` API when used in conjunction with key encoding.

The new Swift API for `AttributedString` is a good example of this flaw.

The `AttributedString` API is extensible, so it does not know about all the 'attributes' that the user may wish to support. And it doesn't know about the possible key coding strategies of any encoder and decoder, so how can it possibly be working?

Well, it doesn't actually work - it has the same issues of not all attributes roundtripping when using key coding strategies.

Is the `allKeys` API to blame?

I think it is. I think that _if_ you require to encode and decode keys that you do not 'own' or cannot know up front, then you should encode them as _Data_ rather than keys that can be transformed. A situation very similar to the issue with the initial `Codable` implementation described above. [Swift / `Codable`: Leave `Dictionary` keys alone](#previous-work)

IF you want to encode keys as data untouched by key coding strategies, you can already do this by encoding a `Dictionary` from key to value. And upon decoding - again decode a `Dictionary` in which your keys will be the raw, untouched `Strings` that you originally encoded.

[AttributedString decoding without using `allKeys`](https://github.com/mortenbekditlevsen/AttributedStringExperiments)

My conclusion is: current dynamic encoding and decoding of keys you cannot know is already broken when used in conjunction with key coding strategies. Providing a 100% solid transformation in one direction from key to snake case representation is no worse than what we have today - and it avoids a ton of confusion for developers about the question of having to work around the current behavior.

Furthermore: if you have dynamic key encoding and decoding needs, encode them in a fashion where they are treated as data, because it is.

[Roundtripping key coding strategies](https://forums.swift.org/t/pre-pitch-roundtripping-key-coding-strategies/52777)

[Current discussion topic on the Swift forums](https://forums.swift.org/t/roundtripping-key-coding-strategies/58494)


## Ideas for further Swift improvements to Firebase

### Type safe paths (RTDB/Firestore)

I've written about this a loong time ago. Please don't use any of the references in that post, since it's likely outdated (more than 4 years old), but the main concept is still very useful.

[Type-safe paths using Phantom Types](https://medium.com/swift2go/swifty-firebase-apis-ka-ching-part-2-f337b579d86a)

Rather than today's:

```swift
let ref = root.child("accounts/\(accountId)/products/\(productId)")
let snapshot = try await ref.get()
let product = try snapshot.data(as: Product.self)
```
Imagine that you could just do:

```swift
let path = Path().accounts(accountId).products(productId)
let product = try await db.get(path)
```

And `product` would automatically be inferred to contain a `Product` because that is encoded in the generic `Path` type. 

Pretty neat, right? :-)

So how would a model of `Path` look for that to work?

Something like:

```swift
public struct Path<Element> {

    private var components: [String]

    fileprivate func append<T>(_ args: String ...) -> Path<T> {
        Path<T>(components + args)
    }

    fileprivate init(_ components: [String]) {
        self.components = components
    }

    var rendered: String {
        components.joined(separator: "/")
    }
}
```
So basically it carries two things. An array of path string components - and it carries a generic signature.

Notice that you cannot yet instantiate a `Path` outside of the file containing this definition.

You can add an initializer to a `Root` component using constrained extensions:

```swift
enum Root {}
extension Path where Element == Root {
  init() {
    self.init([])
  }
}
``` 
Now `Path()` will give you a `Path<Root>`. But we don't actually have an entity called `Root` - it's just a 'phantom generic' that we use to create more constrained extensions:

```swift
enum Account {}

extension Path where Element == Root {
  func account(_ id: String) -> Path<Account> {
    append(["accounts", id])
  }
}
```
And now you can finally add a `Path` to an actual `Codable` entity:

```swift
struct Product: Codable {
  var name: String
  var price: Decimal
}

extension Path where Element == Account {
  func products(_ id: String) -> Path<Product> {
    append(["products", id])
  }
}
```

The final piece of the puzzle is to add an extension to your database or some other type to fetch the data:

```swift
extension Database {
	func get<T>(_ path: Path<T>) async throws -> T where T: Decodable {
		let ref = rootRef.child(path.rendered)
		let snapshot = try await ref.get()
		let t = try snapshot.data(as: T.self)
		return t
    }
}
```

That's the gist of it! 

The above is a simplified version. In a fuller example you could also choose to model paths to collections of entities. For instance, in the above example `/accounts/[account id]/products` would be a `CollectionPath<Product>` and you could add collection specific convenience methods.

With that in place you now have Type Safe access to your database! Your `Path` extensions basically describe a schema of what data is valid in which locations.

This means that you can never write any entities to a path where that entity was not intended to be written.

Furthermore, this extension provides a quite generic API to data, and it works with both the real time database and Firestore, so it can also be used to hide away implementation details of the actual database type in use.

Alongside a tool like [Firebase Bolt](https://github.com/FirebaseExtended/bolt) (an excellent tool, unfortunately not maintained) could be used to autogenerate the `Path` schema.

Firebase Bolt can be used to generate the `rules.json` for RTDB. It supports expressing concepts like:

```
path /accounts/{$account}/products/{$product} is Product {}
```
Where the `Product` definition can also be described. But from the above you could almost entirely generate the `Path` extensions described above. That could be a fun project.

### AsyncStream

The `get` and `observeSingle` in `FirebaseDatabase` map perfectly to Swift's `async/await`.

But it might also be nice to map the other `observe` APIs, that take a callback closure with a snapshot, to an `AsyncStream`.

This would mean that you could use async for loops to access your data. It's not hugely complicated and quite similar to what others may have done in frameworks like `RxSwift` or `Combine`.

Using it might look something like:

```swift
for try await user in ref.observe(as: User.self) {
  // will be called every time the user stored at the ref changes.
} 
```
There are many more things to explore with the different ways to look at a reference, like `.childAdded`, `.childRemoved`, etc. Overloads could be made, that gather output from all of the `.child*` notifications and merge them into a collection. Depending on the use case it could be an `Array` (if there is a query order to respect) or a `Dictionary` (if you are just interested in the entire collection including keys).

### Type system validated queries

In both Firestore and the real-time database you can construct queries using a small query building language. There is however a small caveat: You need to respect the underlying technology with respect to only having one inequality operator for Firestore and only one ordering in the real-time database. There are various ways to construct a query that will result in a run-time error when running the query.

I did a small exploration trying to construct a query builder in which you cannot create invalid queries.

[Type system validated queries](https://gist.github.com/mortenbekditlevsen/9cd48ca0b0d2b1d4ceda9332b6645a3a)

The idea (and I am not certain if it's a good idea, or perhaps too weird) is to model the application of various types of predicates as independent generic parameters. Each parameter basically serves as a boolean value telling whether or not a specific predicate that requires special handling has been applied.

Using constrained extensions on the generic you can then enforce that a specific type of predicate can only be added to a query that doesn't already have it applied.

The `Predicate` type could look something like this:

```swift
protocol QueryApplication {}
enum Applied: QueryApplication {}
enum NotApplied: QueryApplication {}

struct Predicate<
    ArrayContains,
    InAndFriends,
    Inequality,
    OrderBy>
where
ArrayContains: QueryApplication,
InAndFriends: QueryApplication,
Inequality: QueryApplication,
OrderBy: QueryApplication {
    private var predicates: [QueryPredicate]
    private var inequalityField: String?
    func evaluate(_ query: Query) -> Query {
        // TODO: Actually modify query with predicates
        query
    }
}
```
In the above, the `QueryPredicate` type is an enumeration containing each of the predicate building blocks:

```swift
enum QueryPredicate {
  case isEqualTo(_ field: String, _ value: Any)
  case isNotEqualTo(_ field: String, _ value: Any)

  case isIn(_ field: String, _ values: [Any])
  case isNotIn(_ field: String, _ values: [Any])

  case arrayContains(_ field: String, _ value: Any)
  case arrayContainsAny(_ field: String, _ values: [Any])

  case isLessThan(_ field: String, _ value: Any)
  case isGreaterThan(_ field: String, _ value: Any)

  case isLessThanOrEqualTo(_ field: String, _ value: Any)
  case isGreaterThanOrEqualTo(_ field: String, _ value: Any)

  case orderBy(_ field: String, _ value: Bool)

  case limitTo(_ value: Int)
  case limitToLast(_ value: Int)
}
```

And an example of a constrained extension that allows an inequality filter in case one was not already applied:

```swift
extension Predicate where Inequality == NotApplied, OrderBy == NotApplied {
    func isLessThan(_ field: String, _ value: Any) -> Predicate<ArrayContains, InAndFriends, Applied, OrderBy> {
        .init(predicates: self.predicates + [.isLessThan(field, value)], inequalityField: field)
    }
}
```
You can also model the fact that it is ok to have more inequality predicates as long as it's on the same field - by adding overloads that don't take a 'field' parameter and only adds a further inequality predicate:

```swift
extension Predicate where Inequality == Applied, OrderBy == NotApplied {
    func andGreaterThan(_ value: Any) -> Predicate<ArrayContains, InAndFriends, Applied, OrderBy> {
        .init(predicates: self.predicates + [.isGreaterThan(inequalityField!, value)], inequalityField: inequalityField!)
    }
}
```

The only downside I have found with this approach is that one cannot fully hide away these generics in the API, so they will be visible to the end user - although they wouldn't need to deal with them in most use cases.

Building a type system proven valid query can then look something like this:

```swift
let a = Predicate.isEqualTo("state", "CA")
        .isNotIn("population", [23, 24])
        .andLessThan(4)
        .andGreaterThan(2)
        .andNotEqualTo(3)
        .isEqualTo("by_the_ocean", true)
        .order(false)
        .orderBy("a", true)
``` 


## Cross platform Swift implementation of the Firebase SDK

### Current work

I am currently in the (long) process of implementing RTDB, Auth and Firestore in pure, non-Darwin Swift:

An ongoing project of mine is to port the Realtime Database APIs from Objective-C and Darwin to Swift.

I am basically done with the porting. I have two branches: one that maintains compatibility with Objective-C. For this branch all tests (written in Objective-C) pass! :-) This branch does not compile on non-Darwin platforms, however, due to the Objective-C bridging. The other branch is rid of all Objective-C bridging, but this means that the Objective-C tests don't run. I've ported a few tests, but there's still a long, long way to go here.

One gripe that some people have had with the Firebase Apple SDKs is the amount of implicit dependencies you get when adding Firebase to a project. I am afraid that my port hasn't improved that situation, but in my eyes it's a good thing, and not a bad. The project depends on tested and well supported and documented libraries like SwiftNIO, swift-atomics and swift-collections. The Objective-C version uses a really old version of SocketRocket for websocket communication and a custom implementation of a sorted dictionary. I think there are good benefits to not include those dependencies directly into the code base, but to add the dependencies through SPM.

One part that is missing is the injection of authentication tokens, so currently you can only access unprotected databases.

So my next project is to port `FirebaseAuth`, and I'm making good progress here.

After that I guess `Firestore` is next. It's written in C++ with a huge bridging surface area to Objective-C through Objective-C++. 

Importing C++ code to Swift is improving day by day, so it ought to be possible to bridge directly to Swift - instead of attempting to port the core of the library. But the bridging surface area is still huge, so just doing that is quite an undertaking. 

### Further Swiftifications possible

#### Model Functions as Distributed Actors

Calling a function could be modelled by calling methods on a distributed actor whose implementation lives in Firebase Functions.

From the client-side this could technically already be modelled today, since the server implementation of a distributed actor does not technically need to be implemented in Swift. But the main benefit here would of course be extra code sharing between client and server, and as such it is much more valuable in the situation where the server is also implemented in Swift.

#### Eliminate `callbackQueue` APIs when the implementation builds on structured concurrency

A small thing perhaps, but structured concurrency already bakes in the concept of 'being called on a specific thread' - a call to an asynchronous function resumes execution on the actor from which it was called - if any. As such, the specifying of a `callbackQueue` is redundant and the current implementation would perhaps incur extra thread hops.

If structured concurrency was used inside of the implementation of the library, then all mentions of `DispatchQueue`s could perhaps be removed.

The existing implementation does not rely on structured concurrency, since at the time of implementing it, I was not clear about the actual goals of the port. Should it be possible to replace the current implementation with the Swift version on all currently supported platforms? I think that I have decided against that and I imagine that the Swift implementation is basically for modern Darwin platforms + cross platform.

## Issues with the current Swift APIs for Firebase

In general I think that the Swift extensions for Firestore, the real-time database and Functions are really excellent. I do, however, have an issue with a few things in the current APIs:

The introduction of third party primitive types like `Timestamp`, `GeoPoint` and property wrappers like `DocumentID`. The existence of these types suggests that it's a good idea to add a direct dependency from your model entities to Firestore, which in turn pulls in quite a few extra frameworks. 

In my world, having your models depend on a big, external codebase is just not a very good idea.

And the idea becomes even less good when there's no particular reason for those types to exist. For instance `Timestamp`. It already exists as 100% common currency in Swift: `Date` (I guess `Date` is currently in `Foundation`, but it's present in the open source, cross platform version of `Foundation` too). I doubt that the nanosecond precision of `Timestamp` has any practical application in a client-server environment.

The good news is that with the `Codable` support in `Firestore` you can just use `Date` in your models and have them being converted to `Timestamp` when serialized to Firestore.

Likewise with `GeoPoint`. A common currency here would be a `CLLocationCoordinate2D` - although that type is of course part of `Core Location`. Having a method of converting a custom type to a `GeoPoint` upon serialization would be a better way to go. Other than that I don't think that `GeoPoint` has any practical uses yet - there are no distance-based queries possible in Firestore (yet).

And finally there is the `DocumentID` property wrapper. It's this funny thing that you can add to your model - and in case you read a document from Firestore, it will be populated with the key/id of the document.

In previous versions of the SDK it was also possible to set the `DocumentID` annotated property, but this would have no effect when writing the document, so it was a bit confusing.

In my opinion: If you want an id to be part of your model, then model it explicitly.

On the other hand - if you don't want to give your model entity an identity, but you still want to know it in some situations, then you can pull out the information when you need it. If you read a collection of documents, you could choose to surface that in your client as a `Dictionary` from key/id to the entity...

I would love for the Firebase team to have stronger opinions about things are (or should be) important to their users: using libraries in a way that still allows your code to build fast and keep big dependencies out of tests - and out of things like the `SwiftUI` canvas. I have not yet been able to show a canvas where there were any dependencies to a Firebase library. 

Such considerations are not unique to Firebase. I would love for Apple to have the same stronger opinions about tightly coupling something like `Core Data` to `SwiftUI` using their built-in property wrappers. I'd much rather that they shared information about how to build that kind of performant property wrappers, so that users could add their own abstractions even though `Core Data` might be injected at runtime.

## Other issues with Firebase

I am a huge fan of Firebase and how productive the services and client SDKs enable me and my colleagues to be. I do, however, have a few issues which I sincerely hope will be addressed at some point.

### Real-time database indexes

The biggest of those is the hard-to-find truth about real-time database indexes. These indexes are currently not persisted. This means that they are re-created over and over. They do remain in memory in case the same index is used repeatedly. For my use-case with a multi-tenant system there are so many indices that they of course can't all be kept in memory.

What does that mean in practise? Well, an index lookup ought to be logarithmic in time. But if the index must be recreated it will actually be linear instead. This puts a size boundary on a node in the database that is much, much, much lower than the advertised 75 million, because the creation of the index can cause queries to time out. 

This means that if you for instance log events continuously to a node in RTDB, it can become so big that it cannot be queried. 

Attempting to query the node might bog down the database instance so much that other requests start timing out!

It appears that you can still get a shallow list of keys through the CLI, but dealing with this, having to clean up many nodes in the system across multiple tenants have just added so much pain / manual scripting / risk of introducing manual errors.

So it is very, very high on my wish-list to have actual persisted indexes in the Real-time database.

### Multiple Firestore/Storage instances
When Firebase Functions was still a young product and in beta, we made the decision to set our Storage instance to be in the us-central1 region - in order to be 'close' to the functions instances that were only available in one region at that time.

When Firestore was introduced it became evident that we had locked the default region to 'us-central1' based on our choice for Storage.

Today we have customers that would like all data to be stored in the EU and we cannot create new Storage or Firestore instances - and we have basically no migration strategy. We simply cannot accomodate the request to keep data in the EU - and we cannot get the performance benefit of defaulting to EU-based services for EU-based customers and US-based services for US-based customers.

Supporting multiple instances for those to components is also very high on our wish-list.

### Synchronizing users across Firebase projects
Finding a workaround for the above issue could involve an app accessing multiple firebase projects and migrating our customer's accounts over one by one. An issue here is that an account is not equal to a user in our system. One user may be invited to multiple accounts. In order to be able to see accounts across multiple projects we would need to keep users in sync across projects. Not just as a one-time export and import, but as a continuously synchronizing component.

