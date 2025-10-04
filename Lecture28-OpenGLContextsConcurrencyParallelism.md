# Lecture 28 - OpenGL Contexts, Concurrency, Parallelism

## Recap
We finally got out texture rendering correctly on all platforms!
- The root of the problem is that ordering of `std::tuple` isn't standardized.
- We rolled out constraints for types which can be legally used for buffers
  - They must tightly wrap fundamental OpenGL types

## Design Principles / Techniques in Action
Techniques for passing data out of threads  
    - `std::promise` / `std::future` and `std::packaged_task` refinement
`std::latch` as a tool for reducing multithreaded test instabilities

Lambas  
    - Lifetime management of captures  
    - Beware default captures  
    - Immediate Invocation

Tests as a tool to confirm and document understanding


## OpenGL Contexts
To use OpenGL commands requires a context - we are using GLFW to give us a context  

Roughly speaking, we can think of a context as a window  
The concept is more general than this  
However, GLFW cannot create a context without a window, but the window can be hidden  
We can make heavy use of hidden windows in our tests

We can have lots of contexts at a point in time  
Just as we can have lots of windows  
But great care needs to be taken  


## Reviewing the GLFW model  
Set various 'hints': OpenGL version, whether to try to make the context a debug context (won't work on MacOS), whether the window is visible...

Create the window

Make the new window the current context

Load the OpenGL functions for the current context  
We use glad to do this but feed it `glfwGetProcAddress`  
We don't have to use glad with GLFW, but somehow we will need to load the OpenGL functions

## Window Creation
```cpp
GLFWwindow* glfwCreateWindow(
    int width,
    int height,
    const char* title,
    GLFWmonitor* monitor, // nullptr for windowed mode
    GLFWwindow* share // Specify another context with which certain OpenGL state is shared
);
```

## The Current Context
```cpp
void glfwMakeContextCurrent(GLFWwindow* window);
// Makes the context of the specified window current on the calling thread.
```
Any thread can have multiple contexts with at most one current context  

Multiple contexts on a single thread is legal  
But only at most one can be current at a time  
The contexts can be completely independent or share some data  

Multiple contexts on multiple threads is legal  
Each thread can have at most one context current at a given time  
The contexts can be completely independent or share some data

If the same context is current on more than one thread => Undefined behaviour





## Loading OpenGL Functions
OpenGL works on myriad hardware  

It is the job of the driver to implement the OpenGL standard  

What a given driver implements is typically a function of time  
This is true even though the OpenGL standard no longer evolves  
When I stared the course, WSL/Ubuntu implemented 4.2; now it's at 4.5

The driver may implement extensions of the OpenGL standards  

Loading functions at runtime provides a flexible solution


## Approximate Mental Model

Driver Supplied Binary  
.dll, .so ...  
Implements various OpenGL functions  
`void glUseProgram(GLuint) {...}`

Glad  
For the current context setup function pointers  
`glad_glUseProgram` starts life as `nullptr`  
If glad can find `glUseProgram`, it sets `glad_glUseProgram` to point to the write address  
Else  
`glad_glUseProgram` remains `nullptr`

## It's Smoke & Mirrors!
What's this `glad_glUseProgram` business?  
We've never used this! We just use `glUseProgram`!  
Well actually somewhere in `gl.h`...  
`#define glUseProgram glad_glUseProgram`  
So although it looks like we're directly using the driver's functions we aren't!  
We are indirected through a function pointer  
Depending on the OpenGL standard implemented, some of these may be null  

## Function Loading Wars

Context A  
Context B  

`glad_glUseProgram = nullptr;`  

A and B try to load their functions at the same time => Data Race  
Undefined Behaviour  
We'll get away with it if pointer assignment is atomic (not guaranteed)  

A loads, but before we call `glUseProgram`, B loads  
Undefined Behaviour  
If the pointers are the same for A and B, and there's no data race, we'll get away with it  
If the contexts use different hardware, we're sunk  
If the driver makes a context-dependent indirection, we're sunk

## Some Patterns of Multi-Context Usage

- B made current
- WOrk on A suspended
- B can but needn't be on a different thread
- Once B is done, A resumes
  

- B made current on another thread
- A and B execute in parallel
- Periodically, we synchronize

## Concurrency & Parallelism  
Concurrency: This applies to tasks that may be resumed  
Parallelism: This applies to tasks that are done at the same time  
All combinations are valid!  
Our focus is on concurrency, this and without parallelism  


## Concurrency Without Parallelism

Chop (Onions) => Add oil => Add onions => Fry Onions => Chop (other ingredients)

This is concurrent: we resume chopping, or chopping is interruptible  
There is no parallelism

## Concurrency With Parallelism
Chop (Onions) => Add oil => Chop (Other Ingredients) && Fry Onions  
Chop Other Ingredients && Fry Onions at the same time

## Concurrency + Parallelism needs Synchronization
Excessive Chopping =====================>  
Fry Onions =========> Burnt onions

## Problems With The OpenGL Model
An OpenGL calls knows nothing about the context  

OpenGL calls implicitly apply to the current context on the calling thread  

When we make an OpenGL call, we must ensure the intended context is current  

There is no OpenGL call to  
- Get a 'current context index'
- Query which contexts (if any) are shared with the current context  

OpenGL lacks a certain self-awareness

## How This Hurts Us
Suppose we have a context, A and a share program with index 42  

Now we switch to Context B  

What will happen if we call `glUseProgram(42)`?  
- If B is shared with A, then context B will also use our shader program  
- Else it depends whether B also happens to have a shader program with Index 42

The glad multi-context loader can help us ... but not today!

## Hard Decisions
Deciding how to deal with context consistency is a hard piece of design  

For today we won't do anything beyond note the problem  

Instead, we'll focus on understanding easier, related issues
- Thread safety of image loading - nothing to do with OpenGL  
- Preliminary improvements to improving shader program tracking  

And we'll write the tests!
- These confirm and document our current understanding  
- They provide a foundation for future evolution  


## Image Loading Refresher
We use the stb image loading library  

This offers us the option to flip an image vertically when loading  

Initially we used `stb_set_flip_vertically_on_load`

By luck, we discovered an instability in our tests  
- This is because our tests which don't create OpenGL contexts happen to run in parallel
- This stb call is not thread safe  

Looking at the docs, we fixed the problem by instead using `stbi_set_flip_veritcally_on_load_thread`



## What's Going On?
`stbi_set_flip_vertically_on_load_` uses a piece of global state  

Two images loaded in parallel may fight over this state  

Whoever wins, we lose  
- An image that should be flipped isn't
- Or an image that shouldn't be flipped is  

`stbi_set_flip_vertically_on_load_thread` uses thread-local state

## We want to robustly detect mistakes
We got lucky with our test, even so it didn't fail reliably: it was unstable  

The test has changed, so it may not even be sensitive to threading issues  
- Or if it is today, it might not be tomorrow  

We want a test that will almost fail if there's a data race or race condition  

Building such a test is hard and fun  
- Temporarily we deliberately break our code by reverting to `stdbi_set_flip_vertically_on_load`
- We will try to write a test that always fails
- Then we revert our breakage, confident that it breaks for some reason in the future, we'll know about it!


## Data Race or Race Condition?

```cpp


bool vertically_flip_on_load = false;
/*
Thread 1: Flip
Thread 2: No Flip
*/


void set_vertically_flip_on_load(bool flip) {
    vertically_flip_on_load = flip;
}
/*
Data Race! Both threads writing simultaneously
Thread 1 reading, Thread 2 writing
*/


unsigned char* load_image(...) {
  // ETC
  
  if(vertically_flip_on_load) {
    // DO THE THING
}
/*
Race condition!
Thread 2 changing state before Thread 1 has used it
No UB, but a bug!
*/
```


## So What's The Issue With Testing?

If one thread is late to the party, everything works perfectly! On this run, our tests passes.


## Testing Instabilities Are A Nightmare

We either want tests to pass all the time or fail reliably  

Failing sometimes makes development extremely painful  
- If failure is relatively rare, continuous integration may not catch it
- Regardless, tracking down instabilities can be very hard work

A test for a deliberate image loading threading bug should always fail  

On my windows machine a naiive test actually only fails of order 10% of the time!  
- Loading images on two threads, one with flipping the other without  
- My Mac M4 fares much better, failing close to 100% of the time!

## How Do We Do Better
An obvious approach is to increase the number of threads  
This helps, but we can do a lot better!  
The problem is when a thread is late to the party  
Can we force everyone to arrive on time?

## `std::latch`
A `latch` is a C++20 use-once synchronization primitive  
A `latch` holds threads at the chosen point till all of them have arrived  
Then it releases them all at once  
Race horses held at the gate is an excellent analogy  

## Outlines Of The Test  
We need to create some threads  

Each thread loads an image, once all threads are ready  

We then pass the images to the main thread where we check if they're correct  

Question: how do we get data back from the thread?  
- Use `std::promise`, a low-level primitive
- Use `std:;packaged_task` which abstracts away some of the details

We'll use `std::promise` explicitly to show the details of what's going on  
- Try upgrading to `std::packaged_task` is homework!

## Shared State: `promise`, `future`
``` cpp

// make a 'pinkie promise'
// Like an empty box that could hold a value or exception
auto f = pink.get_future();

// == THREAD TIME ==
// Construct a new thread using an invocable, into which pinky is moved
pink.set_value();


// == MAIN THREAD TIME ==
f.get();
```


## Code Setup
```cpp

//Aliases for conveniance
using promise_t = std::promise<unique_image>;
using future_t = std::future<unique_image>;

// Find a good number of threads by experimenting 
constexpr std::size_t numThreads{8};

// Create an array of default constructed promises
std::array<promise_t, numThreads> imagePromises{};

auto imageFutures {
    sequoia::utilities:make_array<future_t>, numThreads>(
        [&imagePromises](std::size_t i) { return imagePromises[i].get_future(); }
           // ^ The promises are cpatured by reference                 ^ Create an array of futures by extracting them from the promises
    )
};
```



## Code: An Array Of Threads

```cpp
std::latch holdYourHorses{numThreads};
// Construct a latch, according to the number of threads
const auto iamgePath{working_materials() / "bgr+striped_2w_3h_3c.png"};
//Get the path of the desired image

auto workers {
    sequoida::utilities::make_array<std::jthread, numThreads>(
    [&imagePromises, &hooldYourHOrses, &imagePath](std::size_t i) {
        return std::jthread {
            [&holdYOurHorses, &imagePath, i](promise_t p) {
                const auto flip{i % 2 ? flip_vertically::yes : flip_vertically::no};
                holdYourHorses.arrive_and_wait();
                p.set_value(unique_image{imagePath, flip, all_channels_in_image});
            },
            std::move(imagePromises[i]);
         };
     }
  )
};
// Build an array of C+=20 jthreads
// Threads must be joined (or detached) prior to the destruction
// jthreads automatically join on destruction (RAII)
// jthreads are also integrated with stop_token
```


## Code: Constructing The Threads

```cpp
// Explicitly captuer by reference just the things we need
// Promises and latches cannot be copied
// Taking imagePath by reference is efficient, regardless a copy has deeper problems..
// We could jsut & but then we would unwittingly use e.g. imageFutures
[&imagePromises, &hooldYourHOrses, &imagePath](std::size_t i) { // Propagate the required references, but i must be captured by value
    return std::jthread {
        [&holdYOurHorses, &imagePath, i](promise_t p) {
            const auto flip{i % 2 ? flip_vertically::yes : flip_vertically::no}; // Do all incidental calculations before hitting the latch
            holdYourHorses.arrive_and_wait(); // Threads held here until the last one arrives
            p.set_value(unique_image{imagePath, flip, all_channels_in_image}); // Create iamges with/without the flip and set the promise's value
        },
        std::move(imagePromises[i]); // ith promise in, the jthread machinery prpagates this into the lambda
    };
```

## Lifetimes
```cpp

[&imagePromises, &hooldYourHOrses, &imagePath](std::size_t i) { // Outer lambda exists until jthreads is constructed 
    return std::jthread {
        [&holdYOurHorses, &imagePath, i](promise_t p) { // Inner lambda is executed by the running jthread. i is no longer captured by your value. This code still compiles but with i captured by reference
            const auto flip{i % 2 ? flip_vertically::yes : flip_vertically::no};
            holdYourHorses.arrive_and_wait(); 
            p.set_value(unique_image{imagePath, flip, all_channels_in_image});
        },
        std::move(imagePromises[i]); 
    };
```
The inner lambda outlives the outer one!  
i is captured by reference so will be dead by the time we come to use it  
Be very wary of default captures [&], [=]  
They have a place, but never as a substitute for thinking!


## Analysis

Rust catches mistakes like this at compile time  
This is unequivocally a big win for Rust.  
But the situation isn't hopeless in C++  
- This incorrect could would cause an instability even if we make the right stb call
- `stbi_set_flip_vertically_on_load_thread`
- We can detect this mistake using an address sanitizer


## Sanitizers
Sanitizers provide runtime instrumentation to catch undefined behaviour  
- They are not guaranteed to catch everything
- They may incur a significant cost  
- They are strictly inferior to preventing such code from compiling

I do some of my builds with an address sanitizer, thanks to a sequoia option
- `cmake ... -D ADDRESS_SANITIZER=ON`
- I actually made the capture-by-reference mistake and the sanitizer caught it!

What we've all been waiting for: Rust vs C++  
- I think with good testing and judicious use of sanitizers, C++ can approach Rust's safety  
- But rust has a definite and persistent advantage in this domain


## Alternative Style: Propagate Arguments as Arguments
``` cpp
auto workers



```


