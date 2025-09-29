# Lecture 2

## Design Principles

### Heed Warnings
- e.g. `int error code;` should be used!

### Don't overcomment
- 'The compiler doesn't read comments'

### Resource Initialization Is Acquisition
- Acquire a resource in the constructor
- Release the resource in the destructor
``` cpp
struct glfw_manager {
    glfw_manager() { glfwInit(); } // Constructor 
    ~glfw_manager() { glfwTerminate(); } // Destructor
}
```
- Destructor is called when you leave scope - even if you throw or return early there is a caveat, if you pass in `glfw_manager` through a function call, when you leave that function scope it will get destructed. This can be prevented with Singleton design pattern (however overcommit on design)! Better solution for this use case is to disabled copies and moves.
- We never explicitly call `glfwInit()` or `glfwTerminate()`
- Additional calls to constructor returns true immediately or returns immediately if the destructor called.
- Always have `glfwTerminate()` before returning
- Have separation of concerns: Initializing/Terminating GLFW is different to Creating/Destroying a window, you'll want different RAII classes for each.
- GLFW likes pointers - deferencing them is undefined behaviour. Always check a pointer before dereferencing it.

### Prefer References to Pointers where you can
``` cpp
class Window {
    GLFWwindow* m_Window;
public:
    GLFWwindow& get() { return *m_Window;}
};
/*
    The get() exposes a refernece in the public interface
    This means that clients never have to do a null check
    We are guaranteeing that it will always be valid as a reference must never be null as it dereferences the pointer immediately.
    Basically you either - get a valid pointer, or crash immediately (then you know it's because of the deference and not somewhere else down the line)
    You also never have to do a null pointer chekc
*/
```
Options to handle the null deference pointer:
- Add a check for extra safety and throw if null
- Enforce non-null as an invariant of the class (The class is small and the problem is localized - but you have to be careful if/when the class is extended)

If the RAII window wrapper has state, then e.g. Window1 => resource <= Window2. If one of the windows goes out of scope, it will also destruct the resource!

A unique pointer `std::unique_ptr` is a smart pointer - a C++ object that manages the lifetime of an object that onely one unique_ptr can own a given object at a time. When it goes out of scope, it automatically deletes the object. You can transfer ownership by moving the pointers
``` cpp
std::unique_ptr<int> p1 = std::make_unique<int>(42);
std::unique_ptr<int> p2 = std::move(p1) // now p2 owns it
// p1 is now null
``` 

Now in the context of if you have the setup Window1 => Resource, and Window2 is in the process of being initalized, then you can have Resource move ownership to Window2. Window1 => nullptr Window2 => Resource.
HOWEVER! There is a potential nullptr deference. There is a small subtle bug.
Now you can do something like
``` cpp
GLFWwindow& get() {
    if(!m_Window) throw std::runtime_error("null ptr");
    return *m_Window;
}
// Code is future proof, but this is redundant if m_Window is guaranteed to be non null, and it's cheap and this is not performance critical code
```
Advanced cpp! Don't be tempted by `noexcept`  
` GLFWwindow& get() noexcept {return *m_Window; }`
- `noexcept` is part of the function signature and says that the return type never raises an exception, and if you want to change the design in the future, you will have to remove it
The code can subtly break (c.f. Contracts - the Lakos Rule)


Summary!
- Do I need to worry about cleanup? Use RAII
- Do I need to do a null pointer check? Don't use pointers
- what happens if I create a window before initializing GLFW? Don't allow it


Other features:
```cpp
glfw_manager(const glfw_manager&) = delete;
/*
    This means that the compiler will refuse to do copy constructor of the class - feature of c++11
    glfw_manager a;
    glfw_manager b = a; // copy not allowed
*/
glfw_manager& operator=(const glfw_manager&) = delete;
/*
    This means that the compiler will refuse to do copy assignment.
    glfw_manager a;
    glfw_manager b;
    a = b; // not allowed
*/

// These two lines ensure that object is unique - 'non-copyable pattern' - this is what std::unique_ptr does internally
```
