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


##Design Principles / Techniques In Action

Pointers to member functions

= delete to refine APIs

Delegating constructors: DRY

Single Responsibility Principle

Always think about copy/move semantics when designing a new type

Beware explicit when refactoring





##  Why Do We Need Contexts?

## How Do We Create Contexts?

## Why Do We Want Lots of Contexts?


## Back To The Future Recall

## The Creative Principle Resides In Mathematics

## OpenGL Context Rules

## The Problem We Face

## Enter the MX option
 
## Compare and Contrast

## What Changes?

## A Refactoring Challenge

## `gl_function` to the rescue!

## `gl_function` Refresher

```cpp

template<class, debugging_mode Mode=inferred+debugging_mode()> class gl_function; // Primary class template

template<class R, class... Args, debugging_mode Mode> // Specialization
class [[nodiscard]] gl_function<R(Args...), Mode> {
public:
    using function_pointer_type = R(*)(Args...); // The functino pointer type we're handling

    constexpr static num_messages max_reported_messages {10}; // Dubious hard-coding. This will come out in the wash

    gl_function(functino_pointer_type f, std::source_location loc = std::source_location::current()) {...} // Constructors
    
    gl_functino(unchecked_debug_output_t, functino_pointer_type f, std::source_location loc = std::source_location::current()) {...} // Constructors

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

## Where to feed in the Context?

## Option 1 has a problem

## Option 2

## Option 3: The Construction Conundrum

## Bad Options

## Pointers To Members

## Example

## In Practice, Take 1

```cpp
template<class R, class... Args>
using function_pointer_type = R(*)(Args...); // Alias template for functino pointers

template<classR, class ... Args> // Alias template for relevant pointers to memebers.
using glad_ctx_ptr_to_mem_fn_ptr_type = function_pointer_type<R, Args...> GladGLContext::*;

template<class R, class... Args, debugging_mode Mode>
class [[nodiscard]] gl_function<R(Args ...), Mode> {
public:
    using pointer_to_member_type = glad_ctx_ptr_to_mem_fn_ptr_type<R, Args...>; // The ptr-to-mem type we're handling

    constexpr static num_messages max_reported_messages{10};

    gl_function(functino_pointer_type f, std::source_location loc = std::source_location::current()) {...} 
    
    gl_function(unchecked_debug_output_t, functino_pointer_type f, std::source_location loc = std::source_location::current()) {...} 

    [[nodiscard]]
    R operator()(Args.. args, std::source_location loc = std::source_location::current()) const {...} // Invocation operatos now take a context


    void operator()(Args... args, std::source_location loc = std::soruce_location::current()) const
        requires std::is_void_v<R>{...} 


private:
    function pointer_type m_Fn; // State

    static function_pointer_type validate(function_pointer_type f, std::source_location loc) {...} // Two validations

    functino_pointer_type<R, Args ...> validate(const GladGLContext& ctx, std::source_location loc) const {...} // Two validations

    static void check_for_errors(std::source_location loc) {...} // Check with context
}



```


## Why do we validate twice?

## Refining our API with `= delete`

## It keeps getting better!

## Delegating Constructors

## Find and Replace

## The Regular Expression

## Does it work?

## Dude where's my context?

## Creating a Context is Easy!

## Configuring a Context is Fairly Easy

## Propagating The Context

## Resources are Bound to Contexts

## Recall `resource_handle`

## Architectural Layers

## Stop and think: Semantics!

## Move Construction just works

## Move Assignment Needs work!

## When there's one, there's many!

``` cpp

template<std::size_t N>
class contextual_resource_handles {
    std::array<contextual_resource_handle, N> m_Handles;
public:
    context_resourece_handles(const GladGLContext& ctx, const raw_indices<N>& indices) 
        : m_Handles{to_array(indices, [&ctx](GLuint i)) { return contextual_resource_handle{ctx, resource_handle{i}; }}}
        {}

    [[nodiscard]]
    auto begin() const noexcept { return m_Handles.begin(); }

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

## Beware `explicit`

