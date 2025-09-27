# Lecture 3

## Recap
- ChatGPT generated GLFW window creation - plagirized from LearnOpenGL

# Design Principles
- Minimize Dependencies
- Don't accidentally throw away state [[nodiscard]]
- Don't assign member variables in the constructor body
- Small, iterative improvements really add up
- Judicious use of `friends` can improve APIs


## Separating out code
- Currently all the GLFW window library is in the `DemoCode.cpp`
Options:
- Just keep it in `DemoCode.cpp`
    Removes all dependencies
- Move it into core library
    Core library is usable out of the box, but then it will depend on GLFW. A window is primitive, expanding the functionality would be difficult.
- Create a new layer between Demo and Core Library
    We limit core library's dependencies.
Other considerations:
- What if other people want to use SDL2 instead of GLFW?
- Multiple windows / context sharing?

To go with option 1: keep it in `DemoCode.cpp`. Core library does not have GLFW dependency, easily changeable in the future.


## Headers and Sources
- Headers (.h, .hpp, .hxx) contains declarations
- Sources (.cpp .cxx) contain definitions
Sometimes headers contain definitions, mostly they are in sources

Sources are parallel universes. During compilation they know nothing about each other. 

A.cpp => A.o
B.cpp => B.o ==> Linking
C.cpp => C.o

The case for reducing dependencies:
- If you make changes and recompile A.cpp, then only that will recompile. 
- If you make changes and recompile A.hpp, then it will have to recompile everything that depends on it - if you have a big dependency chain then that is expensive.

The case for increasing visibility:
- B can see the definition for bar(), but it can't see the definition for foo. If it could the compiler could have optimized it better.

``` cpp
// === A.hpp ===
int foo();
int bar() { return 1729; }

// === A.cpp ===
int foo() { return 42; }


// === B.hpp ===
int baz();

// === B.cpp ===
int baz() { return foo() + bar() ;}
```
What do we do?
We may be constrained:
- Templated code is usually defined in headers
- constexpr functions are defined in headers
    - Although they can somtimes partially delegate to things only declared in a header
And if we aren't constrained: everything depends!
Advice is: Minimize dependencies


## Code organization: Namespaces
` namespace demo { struct glfw_manager; }`
Bad idea to have everything in global namepsaces
Be careful for `using namespace` as it can introduce some subtle bugs

Anonymous namespaces
``` cpp
// === A.cpp ===
int helper() { return 42; }

// === B.cpp ===
int helper() { return 43; }
```
Different implementations with the same name
Linker will throw an error (multiple defined symbols)
DO NOT use `inline`. It is used as a tip for the linker that you will encounter multiple definitions but it doesn't matter as they are all the same. The code will compile, but it will be an ODR Violation. If definitions are not the same then it has some fuzzy behaviour.
Better to use anonymous namespaces!
``` cpp
// === A.cpp ===
namespace { int helper() { return 42; }
}
// === B.cpp ===
namespace { int helper() { return 43; }
}
```
These namespaces are implicitly different
Inline does have its place
E.g.

``` cpp
// === Z.hpp ===
int foo() { ... } // Full definition

// === A.cpp ===
#include Z.hpp

// === B.cpp ===
#include Z.hpp
```
Linker sees two defintiions!
Options:
- Move definition of foo() into Z.cpp
- Else make foo() constexpr, if appropriate (implicitly inline)
- Else you can make foo() in the .hpp inline


## Don't accidentally throw away state
``` cpp
GLFWwindow& get() { return *m_window; }
void foo (window& w) {
    w.get(); // Probably a refactoring mistake!
    // Other code
}
```
get() returns a handle but we're throwing it away.
Benign: we meant to delete this line but forgot to
Problematic: we meant to use the result but forgot to


We can use `[[nodiscard]]
` [[nodiscard]] GLFWwindow& get() { return *m_Window; } `
This will  throw a warning if you call `w.get()` by itself.

But don't go overboard!
``` cpp
class string {
    // ...
    string& operator +=(); // Allows chaining (a += b) += c
}
```
If you add a `[[nodiscard]] in the operator line then you will always get a warning because at some point the chain ends (a += b)

Rule of thumb:
If you have a function that has no side-effects, then you will probably want the return value [[nodiscard]]
oeprator += mutates state
[[nodiscard]] T operator+(const T& lhs, const T& rhs) is pure // You will almost certainly want to keep the result and keep it nodiscard.
If a function has no side effects, what other point can it have beyond generating a value?

RAII: A common pitfall
If you introduce RAII wrapppers and don't give it a name, it will be immediately destroyed
` glfw_manager{}; `
You can prevent this by putting nodiscard on the struct
` struct [[nodiscard]] glfw_manager; `



## Don't assign member variables in the constructor body
``` cpp
class Window {
    GLFWwindow* m_Window{};
public:
    GLFWwindow() {
        // Lots of init code
        m_Window = ... // Did you remember to initialize this? Otherwise it would be a nullptr
    }
};

// Option 1:
class window {
    GLFWwindow* m_Window{};
public:
    GLFWwindow() : m_window{make()} {}
    //Constructor Declaration : Initalizer list, Constructor Body
    // Initializer list inits member variables
}

// Option 2:
class window {
    GLFWwindow* m_Window{make()}; // Moves it inline
public:
    GLFWwindow() = default;
}
```
Shared benefits of both options
- Two-step initialize-assign is avoided
- The desired state is set at the optimal time
- Complex logic for setting state is discouraged
- This has to be done for types which cannot be default initalized


Option 1:
You can move `GLFWwindow() : m_Window{make()} {}` into cpp, reducing dependencies
Care needs to be taken for multiple members:
- The order should match the declaration order
- If it doesn't match and there are dependencies, there will be problems

`GLFWwindow* m_Window{make()};`
This is minimal. For multiple members, initialization order is always unsurprising
This has to be done in the header


Header vs Cpp
General considerations:
- Putting code in the cpp reduces dependencies (improves compile times / logic separation)
- Putting code in the header increases visibility (more optimization opportunities)
Don't be seduced by premature optimization! (We are not opening a billion windows in a tight loop (hopefully))

Currently the OpenGL window creation is not performance critical code for now.
It is higly likely to change. Therefore we can put it in the cpp using an anonymous namespace


## Incremental Refactoring = Opportunities
- Move code out of the constructor body into a helper function
- Then use the initializer list
- Suddenly we can work with a member reference rather than pointer
`GLFWwinodw* m_Window -> GLFWwindow& m_Window`
A word of caution:
- References cannot be assigned to
- So types with copy/move semantics like this need to stick to pointers


## Use Forward Declarations To Minimize Dependencies
When a compiler looks at a class, one thing it needs is the size of its member variables in memory. Here it doesn't know anything about GLFWwindow, but it knows it exists but it doesn't need to because it only deals with ref -> Therefore a pointer! This means we don't even neeed to include GLFW in the header
```cpp
struct GLFWwindow; // Forward declaration => Telling compiler that this exists, you'll find it sometime in the future.
class window {
    GLFWwindow& m_Window;
public:
    GLFWwindow& get() { return m_Windows; }
}
```


## Friends can improve your API
``` cpp
struct [[nodiscard]] glfw_manager {...};
class window {
    window(); // Private constructor
    friend window_manager; // Access granted to window_manager
};
Only a window_manager can create windows!
Windows cannot be created before the windows_manager is created
```
Friendship grants access to private data
If overused it can break encapsulation
But used judiciously, it can improve it!
- Does it reduce your clients' chances of making a mistake without excessive tradeoffs?