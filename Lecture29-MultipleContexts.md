# Lecture 29 - Multiple Contexts

## Recap
Explored OpenGL contexts

Discussed concurrency and parallelism

Wrote robust tests, highly sensitive to data races and race conditions

## Agenda

OpenGL contexts, again

A sneak-peek at what multiple active contexts will give us

glad’s multi-context support

Major refactoring without excessive pain
Utilizing existing structures in our code
Judicious use of regular expressions


## Design Principles / Techniques In Action

Pointers to member functions

= delete to refine APIs

Delegating constructors: DRY

Single Responsibility Principle

Always think about copy/move semantics when designing a new type

Beware explicit when refactoring

##  Why Do We Need Contexts?

OpenGL is a specification, analogous to the C++ standard

A graphics card driver implements some version of the OpenGL standard
Drivers may offer non-standard extensions

We need a Context to actually use the driver’s OpenGL implementation


## How Do We Create Contexts?

We use glfw as a cross-platform solution

In glfw, Contexts are created by making windows
Which we can hide if it’s the context we’re after, rather than the window, per se

A freshly-minted Context is pretty useless!
To do anything interesting, we need to load the driver’s OpenGL function implementations

A brand new Context is like an empty tool belt
Which is better than no tool belt
Loading the functions is like adding tools to the belt


## Why Do We Want Lots of Contexts?

We use glfw as a cross-platform solution

In glfw, Contexts are created by making windows
Which we can hide if it’s the context we’re after, rather than the window, per se

A freshly-minted Context is pretty useless!
To do anything interesting, we need to load the driver’s OpenGL function implementations

A brand new Context is like an empty tool belt
Which is better than no tool belt
Loading the functions is like adding tools to the belt



## Back To The Future Recall

Recall lecture 1 - graphs

Node and Graphs
Nodes are geometry, transforms and textures.
Each edge is a framebuffer
They allow us to render stuff offscreen to texture attachments.
These texture attachment can be used on geometry.

Then you can start rendering.

## The Creative Principle Resides In Mathematics

We identified a DAG as the natural mathematical abstraction
Directed Acyclic Graph

This facilitates creativity!

The graph encodes dependencies
If a node’s dependencies are satisfied, it’s ready to render
Independent “Ready nodes” can be rendered in parallel
We can achieve this by giving each node its own context


## OpenGL Context Rules

Every CPU thread can have multiple contexts

On a given thread, only one context can be current at a time
OpenGL commands are applied to the current context

If the same context is current on multiple threads it’s UB

Much OpenGL state can be shared between contexts
This is how we can propagate information through our graph


## The Problem We Face

We are using the glad OpenGL function loader

The necessary files are generated using a web form

The version I generated is inappropriate for multiple concurrent contexts
All the OpenGL state is global
Function loading can cause data races and/or race conditions
However, for many purposes it’s fine (and is what the LearnOpenGL tutorial uses)


## Enter the MX option
 

https://gen.glad.sh

Enable MX - multiple GL context

Function pointers and data are no longer global

Instead, they are held within a struct

Each instantiation corresponds to a Context


## Compare and Contrast

```cpp

// Normal header

#define GL_VERSION 1_0 1 // Global Variable
GLAD_API_CALL int GLAD_GL_VERSION_1_0;
#define GL_VERSION 1_1 1
GLAD_API_CALL int GLAD_GL_VERSION_1_1;
....


GLAD_API_CALL PFNGLACTIVESHADERPROGRAMPROC glad_glActiveShaderProgram;
#define glActiveShaderProgram glad_glActiveShaderProgram; // Alias of the global variable
GLAD_API_CALL PFNGLACTIVETEXTUREPROC glad_glACtiveTexture;
#define glActiveTexture glad_glActiveTexutre

// MX Header
typedef struct GladGLContext {
    void* userptr;

    int VERSION_1_0; // Analogous Member Variable
    int VERSION_1_1;
    ...

    PFNGLACTIVESHADERPGORAMPROC ActiveShaderProgram; // Analogous member variable
    PFNGLACTIVETEXTUREPROC ActiveTexture;
}
```


## What Changes?

The code is easier to understand in the multi-context case!
There’s a slightly higher level of abstraction

Naming has changed
`glActiveTexture` → `ActiveTexture`

We now require a Context to invoke OpenGL functions
`glActiveTexture(42)` → `context.ActiveTexture(42)`


## A Refactoring Challenge

We must swap out the current version of glad for the mx version

If we then try to compile, our build is very broken

We could use the compiler errors to figure out what to fix

But our design gives us a better way!


## `gl_function` to the rescue!

Invocations of OpenGL functions are wrapped in `gl_function`

Simply searching for `gl_function` illuminates the refactoring challenge

Everywhere there’s a `gl_function`, we’ll need a Context

Let’s start with `gl_function` itself and go from there…


## `gl_function` Refresher

```cpp

template<class, debugging_mode Mode=inferred+debugging_mode()> class gl_function; // Primary class template

template<class R, class... Args, debugging_mode Mode> // Specialization
class [[nodiscard]] gl_function<R(Args...), Mode> {
public:
    using function_pointer_type = R(*)(Args...); // The functino pointer type we're handling

    constexpr static num_messages max_reported_messages {10}; // Dubious hard-coding. This will come out in the wash

    gl_function(function_pointer_type f, std::source_location loc = std::source_location::current()) {...} // Constructors
    
    gl_function(unchecked_debug_output_t, functino_pointer_type f, std::source_location loc = std::source_location::current()) {...} // Constructors

    [[nodiscard]]
    R operator()(Args.. args, std::source_location loc = std::source_location::current()) const {...} // Invocation operators


    void operator()(Args... args, std::source_location loc = std::soruce_location::current()) const
        requires std::is_void_v<R>{...} // Invocation operators


private:
    function pointer_type m_Fn; // State

    static function_pointer_type validate(function_pointer_type f, std::source_location loc) {...} // throw for nullptr upon construction

    static void check_for_errors(std::source_location loc) {...} // Check for OpenGL errors after the raw OpenGL call completes
}




```



## `gl_function` Usage

`gl_function{glActiveTexture} (42);` // Invocation

Construction
glActiveTexture is a function pointer
No template arguments need be specified, thanks to the deduction guide
```cpp
template<class R, class... Args>
gl_function(R(*)(Args...)) -> gl_function<R(Args..)>;
// For this constructor signature, deduce these template arguments so you don't have to reassign the params!
```


## Where to feed in the Context?
Prior to construction
`gl_function{context.ActiveShader}`

At construction
`gl_function{context, ??? ActiveShader ???}`

At Invocation
`gl_function{??? ActiveShader ???}(context, 42)`

## Option 1 has a problem
`context.ActiveShader` is a function pointer (so far so good…)

But we can’t get the context back from this

When we come to invoke, we check for errors

This involves OpenGL calls, for which we need a context!

## Option 2
We know we need the context when it comes to invocation

Therefore, we’ll stash a handle to the context, upon construction, for later use
Which means we need to worry about lifetimes

This feels unnecessary

Why not supply the context when it’s needed?
⇒ Option3

## Option 3: The Construction Conundrum
We want something like `gl_function{ActiveTexture}`

But ActiveTexture is a member variable of GladGLContext

Writing `gl_function{ActiveTexture}` is analogous to expecting this to work:
`struct foo { int x; };`
`some_function(x); // garbage that can't compile because it actually needs to be foo.x`


## Bad Options
We could make a function pointer `context.ActiveShader`

But we’ll need to repeat context to feed to check_for_errors

And we’d better switch to a hybrid Option 1 and 2, else
We could in principle build our function pointer using one context
But check for errors using a different context
Hard to spot if invocation is deferred

This give us an API which is hard to love
`gl_function{context, context.ActiveShader}`
Even here we could use different contexts, though it’s easy to spot

## Pointers To Members
Imagine a sort of pointer to the member we’re interested in

But without an instance of a class/struct there are no addresses to point to

However, we can imagine creating a relative offset
```cpp
struct foo {
    int x;

    double y;
}
/*
[O][O][O][O][X][X][X][X][O][O][O][O][O][O][O]
^int at offset 0
            ^ padding
                        double at offset 8
*/
```

C++ gives us a type-safe, robust way to do this
Memory layout is, in general, compiler-specific
`int foo::*` is a type that can bind to any member of `foo` of type `int`
Binding uses an address-like syntax `int foo::* ptrToMem{&foo::x};` Pretending to be a pointer( really an offset ) 

## Example
``` cpp
struct foo {
    int x{42},
        y{1729};
    
    double z{3.14};
}

/*
Pointer-to-member syntax is pretty arcane
For a member of foo of type T:
	T foo::*
But this is just expressing a type
So we can make a nice alias (template) 
*/
template<class T>
using foo_mem_ptr = T foo::*;


/*
A pointer-to-member is initialized by taking the “address” of the member
There’s no actual object, so this is really an offset, known to the compiler
The construction is type-safe


*/

int main() {
    foo_mem_ptr<int> xMemPtr{&foo::x},
                    yMemptr{&foo::y};
    
    foo_mem_ptr<double> zMemPtr{&foo::z};

    foo f{};

    std::println("{}," f.*xMemPtr);

    std::println("{}," f.*yMemPtr);

    std::println("{}," f.*zMemPtr);
}

/*
A pointer-to-member is initialized by taking the 'address' of the member
There's no actual object, so this is really an offset, known to the compiler
The constructor is type-safe


To use a pointer-to-member we need an object, so we create one

Ordinarily, to access member x, we write f.x
Analogously:
f.(Dereferenced pointer-to-member)
Or    f.*pointer-to-member


*/

```


## In Practice, Take 1

```cpp
template<class R, class... Args>
using function_pointer_type = R(*)(Args...); // Alias template for function pointers

template<classR, class ... Args> // Alias template for relevant pointers to memebers.
using glad_ctx_ptr_to_mem_fn_ptr_type = function_pointer_type<R, Args...> GladGLContext::*;

template<class R, class... Args, debugging_mode Mode>
class [[nodiscard]] gl_function<R(Args ...), Mode> {
public:
    using pointer_to_member_type = glad_ctx_ptr_to_mem_fn_ptr_type<R, Args...>; // The ptr-to-mem type we're handling

    constexpr static num_messages max_reported_messages{10};

    gl_function(function_pointer_type f, std::source_location loc = std::source_location::current()) {...} 
    
    gl_function(unchecked_debug_output_t, functino_pointer_type f, std::source_location loc = std::source_location::current()) {...} 

    [[nodiscard]]
    R operator()(Args.. args, std::source_location loc = std::source_location::current()) const {...} // Invocation operatos now take a context


    void operator()(Args... args, std::source_location loc = std::soruce_location::current()) const
        requires std::is_void_v<R>{...} 


private:
    function pointer_type m_Fn; // State

    static function_pointer_type validate(function_pointer_type f, std::source_location loc) {...} // Two validations

    function_pointer_type<R, Args ...> validate(const GladGLContext& ctx, std::source_location loc) const {...} // Two validations

    static void check_for_errors(std::source_location loc) {...} // Check with context
}



```


## Why do we validate twice?
Upon construction, it’s possible to feed in a nullptr
Although ptr-to-mem is morally an offset, it is legal to set it to nullptr
Actually, `gl_function{nullptr}` doesn’t compile, because CTAD fails
But if clients really want to, they can get around this
`gl_function<void(GLuint)>{nullptr}`
This leaves us validating solely for a weird edge case
Can we do better?

Upon invocation, we acquire a function pointer, which could be null
This is an irreducible problem
Recall that glad may simply not load certain function pointers at runtime
So we can never assume that they’re not null

## Refining our API with `= delete`

We can explicitly remove the unwanted signatures!

```cpp
// nullptr has a type!
gl_function(nullptr_t) = delete;
gl_function(unchecked_debug_output_t, nullpt_t) = delete;
```
Now the compiler will reject
    `gl_function<void(GLuint)>{nullptr}`
But hold on, shouldn’t there be a defaulted source location argument?
After all, the pre-existing constructors take one…
`gl_function(pointer_to_member_type pMember, std::source_location loc = std::source_lcoation::current());`


## It keeps getting better!

The only reason for the source location argument was for validation

But we’ve shifted this piece of validation from runtime to compile time
Which means some of our existing tests no longer make sense and can be deleted

The source location argument can be axed, along with the validation method

```cpp
gl_function(pointer_to_member_type pMember) {...}

gl_function(unchecked_debug_output_t, poitner_to_member_type pMember) {...}

gl_function(nullptr_t) = delete;

gl_function(unchecked_debug_output_t, nullptr_t) = delete;

```


## Delegating Constructors

Our non-deleted constructors are very similar

```cpp
gl_function(pointer_to_member_type pMember) 
    : M_Fn(pMember) // Forward argument to the wrapped type
{}

gl_function(unchecked_debug_output_t, pointer_to_member_type pMember)
    : M_Fn(pMember)
{
    static assert(Mode == debugging_mode:: none); // I added this to give a hard error if clients try to manually specify a different Mode for unchecked output
}
```
But how do we protect against them potentially becoming out of sync in future?
Avoid repetition by delegation
```cpp
gl_function(unchecked_debug_output_t, pointer_to_member_type pMember) 
    : gl_function{p_Member}
    {
        static_assert(Mode == debugging_mode::none);
    }
```

## Find and Replace
We need to make the following transformation:
`gl_function{glFoo}(args...)`
→  `gl_function{&GladGLContext::Foo}(ctx, args...)`
We can’t do this with standard find-and-replace for all Foo

But this is what regular expressions were made for!

Although…

## The Regular Expression
```
Recall the example:
`gl_function{glFoo}(args...)`
→  `gl_function{&GladGLContext::Foo}(ctx, args...)`
Find: `gl_function{gl(.*?)}\((.*?)\)`
The red brackets are capture groups: we can refer back to their contents
Brackets as plain characters are \( or \)
. matches any character except line breaks
.* any number of such matches
? stops the * being greedy
Replace: gl_function{&GladGLContext::$1}(ctx, $2)
$n is the contents of the nth capture group, at least on MSVC

## Does it work?
Kinda!
In the Demo project it misses:
Unchecked debug output (7):             `gl_function{unchecked_debug_ouput, glFoo}(...)`
Invocation with zero arguments (1):     `gl_function{glFoo}()`
Delayed invocation (7):                 `gl_function{glFoo}`
Multiline (1)                           `gl_function{glFoo}(`
                                                            `)`
There are few enough of these it’s easy to do them by hand or by modifying the regex

Used judiciously, regex can be a big time saver…

But they can become more trouble than they’re worth
```
## Dude where's my context?

Our new code cannot possibly compile!
gl_function{&GladGLContext::Foo}(ctx, args...)


We need to create a context and figure out how to propagate it



## Creating a Context is Easy!
```cpp
class [[nodiscard]] window {
    friend glfw_manager;

    window_resource m_Window;
    GladGLContext m_Context{};

    window(const window_config& config, const avocet::opengl::opengl_version& version_);
public:
    window(const window&) = delete;

    window & operator=(const window&) = delete;

    ~window() = default;

    [[nodiscard]] GLFWwindow& get() noexcept { return m_Window.get(); }

    [[nodiscard]] const GladGLContext& context(0 const noexcept { return m_Context; })
}
```
Each window now wraps a GladGLContext

We only need a const getter
- Clients can no longer reset OpenGL function pointers to null  
- Certain old tests can b e axed


## Configuring a Context is Fairly Easy
Loading the OpenGL functions requires
A function with a slightly different name gladLoadGL → gladLoadGLContext
Passing a pointer to the wrapped instance of GladGLContext

``` cpp
window::window(const window_config& config, const agl::opengl_version& version) : m_Window{config, version} {
    glfwMakeCOntextcurrent(&m_Window.get());

    if(!gladLoadGLContext(&m_Context, glfwGetProcAddress))
        throw std::runtime_error{"Failed to initialize GLAD"};

    init_debug(m_context;)
}
```
init_debug contains gl_functions, so the context must be propagated 



## Propagating The Context
Naïvely, we can take the following approach:
Find every function which calls gl_function
Furnish each such function with an additional argument
R foo(const GladGLContext& ctx, ...)

Much of the time, this is what we want to do

But where we are managing OpenGL resources, a bit more thought is needed

## Resources are Bound to Contexts
Previously, we dealt with Contexts somewhat implicitly

We could largely get away with it, since Context lifetimes didn’t overlap

Now we want to deal with the more general case

The problem, in a nutshell:
Suppose I create two Shader Programs
On Context A with index 42
On Context B with index 42
How do I tell them apart?

## Recall `resource_handle`

```cpp
class resource_handle {
    GLuint m_index{}; // Wraps an OpenGL handle
public:
    resource_handle() noexcept = default;

    explicit resource_hande(GLuint index) noexcept : m_Index{index} {}

    resource_handle(resource_handle&& other) noexcept : m_Index{std::exchange(other.m_Index, 0 )}

    resource_handle& operator=(resource_handle&& other) noexcept
    {
        std::ranges::swap(m_index, other.m_Index);
        return *this;
    }
    [[nodiscard]]
    GLUint index() const noexcept { return m_Index; }

    [[nodiscard]]
    friend bool oeprator==(const resource_handle&, const resource_handle&) noexcept = default;
}
```
We don’t want to mess with this!
It has a single responsibility that it carries out well
We want to supplement it with a context_ref 


## Architectural Layers
To first approximation, context_ref just wraps a pointer to a context
	`class context_ref { const GladGLContext* m_Context; }`
Clients get access to a reference, hence the _ref

We can then build a wrapper that holds a context and a resource
```cpp
class contextual_resource_handle {
		context_ref m_Context;
resource_handle m_Resouce;
	public:
			// ETC
};
```

## Stop and think: Semantics!
`resource_handle` is `move-only`

By default, `contextual_resource_handle` will be `move-only`
So far, so good…

But what should happen when we move a `context_ref?`!
In particular, do we need to worry about the moved-from state?

## Move Construction just works

`context_ref` => Context A
`Shader Program 42`
`resource_handle 42`


Creating new:  
`new context_ref` => Context A
`new resource_handle 42`  
`new context_ref` is not owning, no harm in copies. It's beneficial as we can enforce non-null as an invariant
`new resource_handle 42` is now owning, we make sure that the old points to 0 `shader_program 0 `  

## Move Assignment Needs work!

`context_ref_A` => Context A
`Shader Program 42`
`resource_handle_A 42`


`context_ref_B` => Context B
`Shader Program 7`
`resource_handle_B 7`


Move assignment has been crafted to swap `resource_handle`
This means that it will change to
`resource_handle_A 7`
`resource_handle_B 42`  
Therefore we'd better do the same with context ref too!
`context_ref_A` => Context B
`context_ref_B` => Context A
(OK THIS IS A SAM NOTE - the variables were all actually called context_ref x2, it did not have the _A or _B, it was just to illustrate. but this is to enforce that the resource handles and contexts matched in pairs)



## When there's one, there's many!

``` cpp
template<std::size_t N>
class contextual_resource_handles {
    std::array<contextual_resource_handle, N> m_Handles; // N contextual resources...
public:
    context_resourece_handles(const GladGLContext& ctx, const raw_indices<N>& indices) // But this is constructed from a single context. Couldn't we just one context_ref and N resource_handles? Yes, but not today 
        : m_Handles{to_array(indices, [&ctx](GLuint i)) { return contextual_resource_handle{ctx, resource_handle{i}; }}}
        {}

    [[nodiscard]]
    auto begin() const noexcept { return m_Handles.begin(); } // Previously had handles in an array - now exposes a range

    [[nodiscard]]
    auto end() const noexcept { return m_Handles.end(); }

    [[nodiscard]]
    raw_indices<N> get_raw_indices() {
        return to_array(m_Handles, [](const contextual_resource_handle& h) {return h.handle().index(); })
    }

    [[nodiscard]]
    const GladGLContext& context() const noexcept
        requires (N > 0)
        {
            return m_Handles.front().context();
        }
    
    [[nodiscard]]
    friend bool oeprator==(const contextual_resource_handles&, const contextual_resource_handles&) noexcept = default;

}




```


## And then we crank the handle

By and large, refactoring is pretty straightforward

Some names will need subsequently changing
`handle().handle().index()`, anyone?

But we’ll encounter a nasty pitfall


## Beware `explicit`
We want to add const GladGLContext& as the first argument

`resource_wrapper() : m_Handles{lifecycle_type::generate()} {} `

Seems easy enough

`resource_wrapper(const GladGLContext& ctx) : m_Handles{lifecycle_type::generate(ctx)} {} `

Oops! We should have made this single argument constructor `explicit`. This is an incredibly easy refactoring mistake to make!

We must also take care here:
```cpp
template<class... Args>
    requires std::is_constructible_v<lifeEvents, Args...>
explicit(sizeof...(Args) == 1) generic_shader_resource(const Args&... args) // the Sizeof == 1 but we add an argument at the front.
    : m_Handle{LifeEvents{args...} create(ctx)}
    {}
```




## Coding


# `GLFunction.hpp`
Move function_pointer_type alias out, and wrap it with a template class

Define an alias template for the glad context
`using glad_ctx_ptr_to_mem_fn_ptr_type = function_pointer_type<R, Args...> GladGLContext::*;`
Inside the template class
`using pointer_to_mem_type =  glad_ctx_ptr_to_mem_fn_ptr_type`

I think he is basically generalizing the new member functions to the new GLAD context
But he is using the offset instead of pointer.

A lot of propagating context