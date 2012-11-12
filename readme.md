# RHManagedObject

RHManagedObject is a library for iOS to simplify your life with Core Data.  It was motivated by the following:

- Core Data is verbose.  Have a look at [Listing 1](http://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CoreData/Articles/cdFetching.html) from the Apple Documentation and you'll see it takes ~14 lines of code for a single fetch request. RHManagedObject reduces this to a single line of code.

- Each managed object has an object context associated with it, and for some operations you must first fetch the object context in order to operate on the object. For example:

	``` objective-c
	NSManagedObjectContext *moc = [myManagedObject managedObjectContext];
	[moc deleteObject:myManagedObject];
	```

	This is more verbose than necessary since it introduces the object context when its existence is implied by the managed object. RHManagedObject simplifies the above code to:

	``` objective-c
	[myManagedObject delete];
	```
	
- Core Data is not thread safe. If you wish to mutate your objects off the main thread you need to create a managed object context in that thread, attach a `NSManagedObjectContextDidSaveNotification` notification to it, and merge the context into the main object context on an observer method on the main thread.  Bleh.  RHManagedObject does all of this for you so that you can work with your objects in any thread without having to think about all this.

- A common Core Data design pattern is to pass the managed object context between each `UIViewController` that requires it.  This gets cumbersome to maintain, so RHManagedObject puts all the Core Data boilerplate code into a singleton, which becomes accessible from anywhere in your code.  Best of all, the singleton encapsulates the entire `NSManagedObjectContext` lifecycle for you: constructing, merging, and saving (even in a multi-threaded app) such that you never need to interact with `NSManagedObjectContext` directly.  RHManagedObject lets you focus on your objects with simple methods to fetch, modify, and save without having to think about `NSManagedObjectContext`.

- The generated managed object classes leave little room to add additional methods. You can't (or shouldn't) add extra methods to the generated classes since they will be overwritten when the classes are regenerated. RHManagedObject provides a place for additional class and instance methods.

- Managing multiple models becomes tricky with the standard Core Data design pattern.  RHManagedObject (since v0.7) supports multiple models to make this simple and transparent.

## Overview

This brief overview assumes you have some experience with Core Data.

A typical Core Data "Employee" entity (say, with attributes firstName and lastName) has an inheritance hierarchy of:

	NSObject :: NSManagedObject :: EmployeeEntity

RHManagedObject changes this to:

	NSObject :: NSManagedObject :: RHManagedObject :: EmployeeEntity :: Employee
	
You'll notice that the `RHManagedObject` and `Employee` classes have been added to the hierarchy. The `RHManagedObject` class adds generic methods (i.e., not specific to your model) that simplifies interacting with Core Data. Its main features are:

- It manages the object context so that you don't have to think about it.
- It adds easier methods for fetching, creating, cloning, and deleting managed objects.
- It provides a simplified interface for saving the context, and works the same regardless from which thread it's called.

For example, the `newEntity` method introduced in RHManagedObject lets you create a new managed object with a single line:

``` objective-c
Employee *newEmployee = [Employee newEntity];
```

Fetching all employees with first name "John" is also a single line:

``` objective-c
NSArray *employees = [Employee fetchWithPredicate:[NSPredicate predicateWithFormat:@"firstName=%@", @"John"]];
```

The `delete` method lets you delete an existing managed object:

``` objective-c
[firedEmployee delete];
```

Changes can be saved with the `commit` method, which will handle all the nuances of creating and merging the contexts from different threads. In other words, you can call `commit` from your thread and forget about it:

``` objective-c
[Employee commit];
```

You'll notice that none of these examples require direct use of `NSManagedObjectContext`. That's handled for you within the library. Of course, a method is available to fetch the object context for the current thread if it's required:

``` objective-c
NSManagedObjectContext *moc = [Employee managedObjectContext];
```

## How To Get Started

- [Download RHManagedObject](https://github.com/chriscdn/RHManagedObject/zipball/master)
- Copy `RHManagedObject.h`, `RHManagedObject.m`, `RHManagedObjectContextManager.h`, and `RHManagedObjectContextManager.m` into your project
- Include the CoreData framework in your project

## Setup

Recall the new object hierarchy from the overview:

	NSObject :: NSManagedObject :: RHManagedObject :: EmployeeEntity :: Employee

Your entity class (e.g., EmployeeEntity) is generated by XCode as usual (CMD-N, NSManagedObject subclass, etc.).  However, there are a few manual tasks before and after generation.

- Before generation you must ensure the `Class` setting on your entity is set to the entity name.  That is, open the xcdatamodeld in XCode, select the entity, and set the `Class` property (at the right) to the entity name. In the employee example this would be `EmployeeEntity`.  Repeat for each entity to be generated.
- After your entity classes have been generated you must go back to the xcdatamodeld and change the `Class` property to your entity subclass.  In the employee example this would be `Employee`.
- The generated classes must be modified to inherit from `RHManagedObjectClass` instead of `NSManagedObjectClass`.  It's a small hack, but only requires changing two lines of code (if anyone has an easier way of doing this then please let me know).
- The entity subclass (e.g., `Employee`) is created by normal means, but requires two methods to identify which Core Data entity it extends and to which model it belongs. These are used by the `RHManagedObject` superclass and looks like the following in the `Employee` example:

	``` objective-c
	@implementation Employee
	
	// This returns the name of the Entity it extends (basically the name of the superclass)
	+(NSString *)entityName {
		return @"EmployeeEntity";
	}
	
	// This returns the name of your xcdatamodeld model, without the extension
	+(NSString *)modelName {
		return @"SimplifiedCoreDataExample";
	}
	
	@end
	```
	
	However, it's also the place where additional methods can be added without disrupting the generated entity class. For example:
	
	``` objective-c
	-(NSString *)fullName {
		return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
	}
	```

## Other Features

<!-- The library also contains a singleton class called `RHManagedObjectContextManager`, which contains the Core Data boilerplate code that's normally found in the AppDelegate. It also handles the managed object contexts among the different threads, and the merging of contexts when saving. You'll likely never need to use this class directly. -->

### Populate Store on First Launch

The library contains code to populate the store on first launch. This was motivated by the [CoreDataBooks example](http://developer.apple.com/library/ios/#samplecode/CoreDataBooks/Introduction/Intro.html), and all you have to do is copy the sqlite file generated by the simulator into your project. The library takes care of the rest.

### ARC

The library is using ARC as of version 0.8.0.

<!-- ### Mass Update Notification -->

## Example Usage

Once you have setup RHManagedObject it becomes easier to do common tasks.  Here are some examples.

### Add a new employee

``` objective-c
Employee *employee = [Employee newEntity];
[employee setFirstName:@"John"];
[employee setLastName:@"Doe"];
```

### Fetch all employees

``` objective-c
NSArray *allEmployees = [Employee fetchAll];
```

### Fetch all employees with first name "John"

``` objective-c
NSArray *employees = [Employee fetchWithPredicate:[NSPredicate predicateWithFormat:@"firstName=%@", @"John"]];
```

### Fetch all employees with first name "John" sorted by last name

``` objective-c
NSArray *employees = [Employee fetchWithPredicate:[NSPredicate predicateWithFormat:@"firstName=%@", @"John"] sortDescriptor:[NSSortDescriptor sortDescriptorWithKey:@"lastName" ascending:YES]];
```

### Get a specific employee record

The `getWithPredicate:` method will return the first object if more than one is found.

``` objective-c
Employee *employee = [Employee getWithPredicate:[NSPredicate predicateWithFormat:@"employeeID=%i", 12345]];
```

### Count the total number of employees

``` objective-c
NSUInteger employeeCount = [Employee count];
```

### Count the total number of employees with first name "John"

``` objective-c
NSUInteger employeeCount = [Employee countWithPredicate:[NSPredicate predicateWithFormat:@"firstName=%@", @"John"]];
```

### Get all the unique first names

``` objective-c
NSArray *uniqueFirstNames = [Employee distinctValuesWithAttribute:@"firstName" withPredicate:nil];
```

### Get the average age of all employees

``` objective-c
NSNumber *averageAge = [Employee aggregateWithType:RHAggregateAverage key:@"age" predicate:nil defaultValue:nil];
```

### Fire all employees

``` objective-c
[Employee deleteAll];
```

### Fire a single employee

``` objective-c
Employee *employee = [Employee get ...];
[employee delete];
```

### Commit changes

This must be called in the same thread where the changes to your objects were made.

``` objective-c
[Employee commit];
```

### Completely destroy the Employee model (i.e., delete the .sqlite file)

This is useful to reset your Core Data store after making changes to your model.

``` objective-c
[Employee deleteStore];
```

### Get an object instance in another thread

Core Data doesn't allow you to pass managed objects between threads.  However you can generate a new object in a separate thread that's valid for that thread.  Here's an example using the `-objectInCurrentThreadContext` method:

``` objective-c
Employee *employee = [Employee getWithPredicate:[NSPredicate predicateWithFormat:@"employeID=%i", 12345]];

dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
	
	// employee is not valid in this thread, so we fetch one that is:
	Employee *employee2  = [employee objectInCurrentThreadContext];
	
	// do something with employee2	
	
	[Employee commit];
});
```

## I'm stuck.  Help!

Check out the included SimplifiedCoreDataExample for a working example.  You can also contact me if you have any questions or comments.

## Where is this being used?

RHManagedObject is being used with:

- [TrackMyTour.com](http://trackmytour.com/) - Travel Sharing for iPhone and iPad
- [Warmshowers.org](http://warmshowers.org/) - Hospitality for Touring Cyclists
- [PubQuest.com](http://pubquest.com/) - Not yet released

Contact me if you'd like your app to be listed.

## Contributions

- James Carlson

## Contact

[Christopher Meyer](https://github.com/chriscdn)  
[@chriscdn](http://twitter.com/chriscdn)

## License
RHManagedObject is available under the MIT license. See the LICENSE file for more info.