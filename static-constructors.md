# PHP RFC: Static Constructors

- Version: 0.1
- Date: 2024-06-08
- Author: Erick de Azevedo Lima <ericklima.comp@gmail.com>
- Status: Under Discussion
- Target Version: PHP 8.4
- Implementation: <https://github.com/erickcomp/php-src_static-constructor/tree/master>
- First Published at: TBD

## Introduction

## Proposal

Provide a new magic method called ```__staticConstruct```. If declared on a class, this method will be invoked automatically by the engine just after the current code that initializes the class' static variables. 

### Implementation on other programming languages

####C\#

C# has a feature static constructors

####C++
C++ (Since the 2017 specification "C++17") provides a way to initialize complex values for static variables using lambdas (the equivalent to PHP's closures)
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

####Java
Java has static blocks for this purpose:

####Kotlin

## Backward Incompatible Changes
Only for projects that defined a ```__staticConstruct``` method (No one should be creating methods that start with "\_\_" anyway...). On a search  I performed on Github, I could not find any projects with such method.
## Version

Next minor version, PHP 8.4, and next major version PHP 9.0.

## Vote

VOTING_SNIPPET

## Future scope
In case any uses cases for early execution of the static constructor arise, an opt-in method for eager init can be implemented.

## References
