# PHP RFC: Static Constructors

- Version: 0.1
- Date: 2024-06-08
- Author: Erick de Azevedo Lima <ericklima.comp@gmail.com>
- Status: Under Discussion
- Target Version: PHP 8.4
- Implementation: <https://github.com/erickcomp/php-src_static-constructor/tree/master>
- First Published at: TBD

## Introduction
In PHP, initializing static properties with non-constant values often requires cumbersome workarounds that can clutter the code and reduce readability. One commonly used technique is to create a method that's called inside the class constructor or before any attempt to read the static property. This method first checks if the static properties have been initialized, and if they haven't, it initializes them. Here is an example of code I've encountered in some codebases, which requires calling the initializer method before every usage of the static property to ensure it's properly initialized:

```php
class MyClass
{
    private static \DatetimeImmutable $minDate;
    
    public function __construct(
        private \DateTimeInterface $scheduledDate
    ) {
        self::initializeMinDate();
        
        if ($scheduledDate < self::$minDate) {
            $errMsg = 'Cannot set a scheduledDate before [' . self::$minDate->format('d/m/Y H:i:s') . ']';
            
            throw new \DomainException($errMsg);
        }
    }

    private static function initializeMinDate()
    {
        if(isset(self::$minDate)) {
            return;
        }
        
        // could get it from config, database or whatever 
        self::$minDate = new \DatetimeImmutable('2024-01-01 00:00:00');
        
    }
    
    // ...
}


$c = new MyClass(scheduledDate: new \DateTimeImmutable('2024-01-01 00:00:00'));

var_dump($c);
```

Such boilerplate reduces readability and could lead to bugs if a new method is implemented and the programmer forgets to call the initializer method:


```php
class MyClass
{
    private static \DatetimeImmutable $minDate;
    
    public function __construct(
        private \DateTimeInterface $scheduledDate
    ) {
        self::initializeMinDate();
        
        if ($scheduledDate < self::$minDate) {
            $errMsg = 'Cannot set a scheduledDate before [' . self::$minDate->format('d/m/Y H:i:s') . ']';
            
            throw new \DomainException($errMsg);
        }
    }
    
    public static function getMinDate()
    {        
        return self::$minDate;
    }

    private static function initializeMinDate()
    {
        if(isset(self::$minDate)) {
            return;
        }
        
        // could get it from config, database or whatever 
        self::$minDate = new \DatetimeImmutable('2024-01-01 00:00:00');
        
    }
    
    // ...
}

// Boom! getMinDate does not check if MyClass::$minDate was initialized
echo MyClass::getMinDate()->format('d/m/Y H:i:s') . PHP_EOL;

$c = new MyClass(scheduledDate: new \DateTimeImmutable('2024-01-01 00:00:00'));

var_dump($c);
```

## Proposal

This RFC proposes the introduction of a new magic method, `__staticConstruct`, in PHP to streamline the initialization of static properties and improve code clarity. If declared in a class, this method will be invoked automatically by the engine just after the current code that initializes the class's static properties. Before proposing this RFC, I considered using a userland autoloader for this purpose, but I realized that it is clearly out of the scope of a class autoloader, which is to find and load the class.


## Previous RFC on this subject

There was a previous attempt to create an RFC for static constructors, which can be found at:
<https://externals.io/message/84602>


### Addressing Potential Criticism

Some might argue against this RFC on the basis that static properties should be avoided as suggested by certain design patterns or even due to concerns about testability. However, static properties already exist in PHP and have proven to be useful in certain scenarios. Ignoring the need for a more elegant initialization method does not negate the practical use cases where static properties are beneficial and the natural approach.

### Comparison with Other Object-Oriented Languages

Object-oriented languages like Java, which adhere more strictly to object-oriented principles, include static properties and offer mechanisms for their initialization with non-trivial expressions. Java uses method calls or static blocks for this purpose, as will be demonstrated later in this text, illustrating that even in environments stricter about OOP principles than PHP, static properties are sometimes useful and require appropriate initialization methods.


## A real-life example
One real-life and prominent example is found in Composer's `ClassLoader` class, which uses a closure to initialize the `$includeFile` static property. This initialization is done in an `initialize` method, which is called inside the constructor, resulting in less elegant and less readable code.

Composer's ClassLoader:

```php
class ClassLoader
{
    // ...
    /**
     * @param string|null $vendorDir
     */
    public function __construct($vendorDir = null)
    {
        $this->vendorDir = $vendorDir;
        self::initializeIncludeClosure();
    }
    
    // ...
    
    /**
     * @return void
     */
    private static function initializeIncludeClosure()
    {
        if (self::$includeFile !== null) {
            return;
        }

        /**
         * Scope isolated include.
         *
         * Prevents access to $this/self from included files.
         *
         * @param  string $file
         * @return void
         */
        self::$includeFile = \Closure::bind(static function($file) {
            include $file;
        }, null, null);
    }	
    
```

If there were a ```__staticConstruct``` method, it could be written like this:

```php
class ClassLoader
{
    // ...
    /**
     * @param string|null $vendorDir
     */
    public function __construct($vendorDir = null)
    {
        $this->vendorDir = $vendorDir;
    }
    
    private static function __staticConstruct()
    {
         /**
         * Scope isolated include.
         *
         * Prevents access to $this/self from included files.
         *
         * @param  string $file
         * @return void
         */
        self::$includeFile = \Closure::bind(static function($file) {
            include $file;
        }, null, null);
    }
    
    // ...    
}
```

This case demonstrates how introducing a static constructor in PHP would not only simplify the initialization of static properties but also improve code readability and maintainability. This RFC aims to enhance PHP's object-oriented capabilities by providing a cleaner mechanism for initializing static properties.

## What this RFC is not

This RFC does not intend to promote the indiscriminate usage of static properties but rather to provide a way to properly initialize them when they are useful.

### Implementation on other programming languages

#### Java

Java does not have a so-called static constructor, but it provides other ways to initialize static properties:

##### 1 - ```static``` blocks:

```java
import java.util.*;
import java.lang.*;
import java.io.*;

class Whatever {
    public static int myVar;

    static {
        myVar = 10;
    }
}

class Main {
    public static void main(String[] args) {
        System.out.println("Hello world (" + Whatever.myVar + ")!");
    }
}
```
##### 2 - Initialization using method calls:

```java
import java.util.*;
import java.lang.*;
import java.io.*;

class Whatever {
    public static int myVar = initializeInstanceVariable();
        
    protected static int initializeInstanceVariable() {
        return 20;
    }
}

class Main {
    public static void main(String[] args) {
        System.out.println("Hello world (" + Whatever.myVar + ")!");
    }
}
```

<https://docs.oracle.com/javase/tutorial/java/javaOO/initial.html>

#### Kotlin
Kotlin, strictly speaking, does not offer static members/methods. However, they can be implemented using companion objects and the `@JvmStatic` annotation:

```kotlin
class Whatever {
    companion object {
        @JvmStatic
        var myVar: Int

        init {
            myVar = 10
        }
    }
}

fun main() {
    println("Hello, world(" + Whatever.myVar + ")!")
}
```

<https://kotlinlang.org/docs/object-declarations.html#companion-objects>

#### C\#

C# implements a mechanism called "static constructor". It's a method with no modifiers and has even more restrictions than the private access modifier. It can only be called by the CLR (Common Language Runtime), which gives guarantees about its execution:


###### 1 - Static constructor is called at most one time by the CLR;
###### 2 - Static constructor is called before any instance constructor is invoked or member is accessed;

```csharp
using System;
using System.Reflection;

public class C1
{
	public static int myVar;
	
	static C1()
	{
		C1.initStaticProps();
		Console.WriteLine("C1::__staticConstructor");
	}
	
	public static void initStaticProps()
	{
		C1.myVar = 10;
		Console.WriteLine("Initializing C1::test");
		
	}
	
	public static void test()
	{
		Console.WriteLine("C1::test");
	}
}

public class Program
{
	public static void Main()
	{
		/* Any reference to the C1 class will trigger the static constructor: */
        
		// Reflection
		//Type c1Type = typeof(C1);
		
		// Instance constructor
		//var c = new C1();
		
		// Static method call
		//C1.test();
		
		// Property access
		Console.WriteLine("C1::myVar:" + C1.myVar);
	}
}
```

<https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/static-constructors>


#### C++
Since its 2017 specification "C++17", C++ provides a way to declare constant expression lambdas (equivalent to PHP's closures). This allows leveraging this feature to initialize complex values for static properties using an `inline` with `constexpr` lambda:

```c++
#include <iostream>
#include <vector>

class Example {
    public:
        inline static const std::vector<char> letters = [] {
            std::vector<char> letters;
            
            for (char c = 'a'; c <= 'z'; c++) {
                letters.push_back(c);
            }
    
            return letters;
        }();
};

int main() {
    for (auto i = 0; i < Example::letters.size(); i++) {
        std::cout << "Example::letters[" << i << "]: " << MyClass::letters[i] << std::endl;    
    }
    
    return 0;
}
```

#### Swift

Just like C++, Swift allows properties to be initialized by the return values of closures:

```swift
class Example {
    static var myVar: Int = {
        print("Static initializer called")
        return 42
    }()

    static func display() {
        print("Example::myVar: \(myVar)")
    }
}

Example.display()
```

## Design decisions

This implementation draws inspiration from the C# implementation but is less restrictive. Programmers have the option to call the `__staticConstruct` method to reset static properties values if desired.

The implementation enforces the `private` access modifier on the `__staticConstruct` method intentionally to prevent inheritance of the static constructor. This prevents unintended re-initialization of parent class static properties, which is often not desired. If a programmer wants to share code from a parent class, they can create a new method and call it within `__staticConstruct`.

The static constructor in this RFC supports two possible signatures:

1. `private static __staticConstruct();`
2. `private static __staticConstruct(): void;`

Similar to the C# implementation, the static constructor can take no arguments.

## Implementation notes

- Unlike C#, this implementation of the static constructor does not get triggered at the first reference to the class in the code. Instead, it utilizes the existing lazy initialization logic for static properties implemented in the `zend_class_init_statics` function within the `zend_object_handlers.c` file. Initialization occurs upon the first reference to any static property of the class;

- Even if the method were `protected` or `public`, it would not be automatically called by the engine if not explicitly declared. Unlike the `__construct` method, the static constructor method is intentionally not copied to the correspondent "shortcut" variable dedicated to this magic method during inheritance operations (performed at the `do_inherit_parent_constructor` function of the `zend_inheritance.c` file).


## Examples

###### 1 - No code sharing to the child class:
```php
class MyClass
{
    protected static \DatetimeImmutable $minDate;
    
    public function __construct(
        private \DateTimeInterface $scheduledDate
    ) {
        if ($scheduledDate < self::$minDate) {
            $errMsg = 'Cannot set a scheduledDate before [' . self::$minDate->format('d/m/Y H:i:s') . ']';
            
            throw new \DomainException($errMsg);
        }
    }
    
    public static function getMinDate()
    {        
        return self::$minDate;
    }

    private static function __staticConstruct(): void
    {
        // could get it from config, database or whatever 
        self::$minDate = new \DatetimeImmutable('2024-01-01 00:00:00');
    }
    
    // ...
}

// No more "Boom!", as MyClass::$minDate was initialized on the __staticConstruct method
echo MyClass::getMinDate()->format('d/m/Y H:i:s') . PHP_EOL;
```

###### 2 - Sharing code to the child class:
```php
class MyClass1
{
    protected static \DateTimeImmutable $minDate;

    public function __construct(
        private \DateTimeImmutable $scheduledDate
    ) {
        if ($scheduledDate < self::$minDate) {
            $errMsg = 'Cannot set a scheduledDate before [' . self::$minDate->format('d/m/Y H:i:s') . ']';

            throw new \DomainException($errMsg);
        }
    }

    public static function getMinDate()
    {
        return static::$minDate;
    }

    public function getScheduledDate(): \DateTimeImmutable
    {
        return $this->scheduledDate;
    }

    protected static function initMinDate(): \DateTimeImmutable
    {
        // could get it from config, database or whatever 
        return new \DatetimeImmutable('2024-01-01 00:00:00');
    }

    private static function __staticConstruct(): void
    {
        self::$minDate = self::initMinDate();
    }

    // ...
}

class MyClass2 extends MyClass1
{
    protected static \DateTimeImmutable $minDate;

    private static function __staticConstruct(): void
    {
        self::$minDate = parent::initMinDate()->modify('+1 day');
    }
}

echo MyClass1::getMinDate()->format('d/m/Y H:i:s') . PHP_EOL;
echo MyClass2::getMinDate()->format('d/m/Y H:i:s') . PHP_EOL;
// prints:
// 01/01/2024 00:00:00
// 02/01/2024 00:00:00
```

## Backward Incompatible Changes

This RFC introduces backward incompatible changes only for projects that have defined a `__staticConstruct` method. It's worth noting that methods starting with `__` are discouraged in userland codebases. A search conducted on GitHub did not yield any results for projects with such a method.

## Version

Next minor version, PHP 8.4

## Vote

VOTING_SNIPPET

## Future scope

In case any use cases for early execution of the static constructor arise (such as during class load, as implemented in C#), an opt-in mechanism for even earlier initialization can be considered.


## References
Composer's ClassLoader class: <https://github.com/composer/composer/blob/main/src/Composer/Autoload/ClassLoader.php>

Java's approach on "properties" initialization:
<https://docs.oracle.com/javase/tutorial/java/javaOO/initial.html>

Kotlin's companion objects:
<https://kotlinlang.org/docs/object-declarations.html#companion-objects>

C#'s static constructors:
<https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/static-constructors>

