# Lecture 4 - Hello Triangle

# Design Principles
- Don't repeat yourself
- Make iterative improvements
- Keep your scopes small
- Premature optimization is the root of all evil
- Code to standards, not implementations

# Rendering Pipeline Overview

CPU Side: Data  
On the heap or on the stack?  
`std::vector` manages memory on the heap    
Built-in arrays are on the stack; `std::array` is a nice wrapper  
Either way, we can improve expressivity by introduving a `vertex` type e.g.
``` cpp
struct vertex {
    float x{}, y{}, z{};
};
```

Option 1: On the heap  
`std::vector` manages memory on the heap.
`std::vector<vertex> verticies{{-1,-1,0}, {1,-1,0}, {0,1,0}};`
Pros: It can be sized at runtime => If the program is reading in data, this is a good option
Cons: We may lose information we know at compile time (E.g. we know a triangle has 3 verticies, this is not a known static property right now). It will allocate - but don't prematurely optimize for this reason alone!

Option 2: On the stack  
Built-in arrays live on the stack  
Prefer the modern abstraction, std::array  
`std::array<vertex, 3> vertices{{-1,-1,0}, {1,-1,0}, {0,1,0}};`
Pros: No allocations and very fast access, retention of things we know at compile time (A triangle has 3 vertices, which is encoded in the type).
Cons: Doesn't play nicely with things that aren't known until runtime, the amount it can hold is limited by your stack size

LearnOpenGL Uses Built-In Arrays:
```cpp
float vertices[] = {
        -0.5f, -0.5f, 0.0f, // left
        0.5f, -0.5f, 0.0f, // right
        0.0f, 0.5f, 0.0f // top
};
```
(No Vertex Abstractions)

Configuring the GPU  
We need memory on the GPU => Vertex Buffer Objects (VBO)  
We pass our CPU-side vertices to the VBO  
But the GPU has no idea what this data represents (just a bag of bits)  
So we need a Vertex Attribute Object (VAO) to tell it what's what.

CPU             =>  GPU  
Vertices        =>  VBO  
Interpretation  =>  VAO  


On the GPU side, you will input Vertex Data  
[0][1][2]  
=> Each piece of data is sent through the Vertex Shader (Processing each vertex, translate/rotate etc)  
=> Then it is sent through the Primitive Assembly (We then interpret them as triangles)  
=> Then it is sent through the Rasterizer (The output is from mathematical ideal of geometry to the representation on a screen) At this stage they are fragments (kind of like pixels)  
=> Then it is sent through the Fragment Shader (Coloured, lighting etc)


Shader Programs  
Programs that run on the GPU  
Pipeline Structure  
Vertex Shader => Optional Stages => Fragment Shader  
We will use GLSL programming language - a bit like C but adapted for Graphics Programming

``` cpp
const char *vertexShaderSource = "#version 330 core\n"
	"layout (location = 0) in vec3 aPos;\n"
	"void main()\n"
	"{\n"
	"   gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);\n"
	"}\0";
```

OpenGL Basic Patterns  
Create / Generate Resources on the GPU  
We get a handle, of type GLuint (roughly equivalent to an unsigned int)  

``` cpp
// Example 1: A Shader
unsigned int vertexShader = glCreateShader(GL_VERTEX_SHADER);
glShaderSource(vertexShader, 1, &vertexShaderSource, NULL); // Config
glCompileShader(vertexShader); // Config


// Example 2: A Shader Program
unsigned int shaderProgram = glCreateProgram();
glAttachShader(shaderProgram, vertexShader); // Config
glAttachShader(shaderProgram, fragmentShader); // Config 
glLinkProgram(shaderProgram); // Config

// Example 3: Vertex Buffer Object (VBO)
unsigned int VBO; // Handle
glGenBuffers(1, &VBO); // Passed in as a pointer

// Example 4: Vertex Array Object (VAO)
unsigned int VAO; // Handle
glGenVertexArrays(1, &VAO); // Passed in as a pointer
```



Bindings
``` cpp 
// The shaders are quite intuitive
unsigned int shaderProgram = glCreateProgram();
glAttachShader(shaderProgram, vertexShader);

// VBOs / VAOs are much less intuitive
// Generate, Bind using Handle, Subsequent config implicitly applies to the bound object

glBindVertexArray(VAO);
glBindBuffer(GL_ARRAY_BUFFER, VBO);
// !! Config: no further mention of the handles !! 
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);


// OpenGL Evolution: OpenGL 4.5 Introduces:
void glNamedBufferData(GLuint buffer, GLsizeiptr size, const void *data, GLenum usage);
```
However, OpenGL is deprecated in Mojave. It is actively maintained but Apple refuses to go beyond OpenGL 4.1 (Currently 4.6 in 2017, probably final version)

# We need to start refactoring
Start using vocabulary types  
` float vertices[]{...} => std::array<vertex<float>,3>`  
` const char*           => constexpr std::string_view`  
Introduce RAII wrappers  
`glCreateShader ... glDeleteShader`  
Write a shader-loading class  
Hard-coding them in C++ code does not scale, we want to write our shaders in files and load them  

Next: Reduce Repetition  
A lot of code for the shader / fragment shader compilation are almost exactly the same: Find & Replace (vertex => fragment, VERTEX => FRAGMENT)
Considerations:  
Commonality is obscured: Are two nearly identical pieces of code doing the same thing or not? Are differences important or incidental?  
There are multiple maintenance points: If i change code, I have to remember to do so in several places. This is brittle and error prone.  

Other Problems: Variables aren't initialized
``` cpp
int success; // Uninitialized, so garbage
char infoLog[512];
glGetShaderiv(vertexShader, GL_COMPILE_STATUS &success);
if(!success) {
    glGetShaderInfoLog(vertexShader, 512, null, infoLog);
}
// If success is true, infoLog is still full of garabge, so still don't read from it
```
Related Problems: Scoping  
You might declare the `infoLog`, then precending code may either use or it not use it.  
This is difficult because you have to understand all of the code in between (or even worse if it is multi-threaded)  
So either make it a `const` or limit the scope.  
One counter-argument: This character array has 512 indices. This is false! In the existing design we always create it but don't always use it.   
You should write code that is easy to reaosn about. If it's easy for you to understand, it's easy for the optimizer to understand. Don't prematurely optimize your code - it may be irrelevant / do nothing / inhibit the optimizer  

A better design
``` cpp
void check(GLuint shaderID) {
	GLint success{}; // Initialized to zero
	glGetShaderiv(shaderID, GL_COMPILE_STATUS, &success);
	if(!success) {
		char infoLog[512]{}; // Info log is only created if it's needed to. ELements initialized to zero. This has a performance implication but only appears when something goes wrong anyways
        glGetShaderInfoLog(shaderID, 512, NULL, infoLog);
	    throw std::runtime_error{...};
    }
}
```

C++26 has Erroneous Behaviour  
Accidentally reading from an uninitialized variable is very bad - Undefined Behaviour!
``` cpp
int x; // Erroneous value (gcc, vs studio each of these will give you the same value)
int y = x; // Erroneous behaviour (Implementations encouraged to diagnose)
```
If you really need to, you can opt out `[[indetermiate]]`

Further Refinements
``` cpp
GLint check_status(GLuint shaderID) {
	GLint success{};                                        //
	glGetShaderiv(shaderID, GL_COMPILE_STATUS, &success);   // Minimal scope for mutable variable
	return success;                                         //
}
void check(GLuint shaderID) {
	if(!check_status(shaderID)) {
		…
	}
}
// Notice that int has changed to GLint => GLint is required to be 4 bytes, int may or may not be 4 bytes depending on the specific implementation
// In Glad, GLint is just an alias for int
```
# Continue Refactoring  
Some of the refactoring for duplicated code can have subtle differences  
E.g. `glGetShaderiv` versus `glGetProgramInfoLog` calls are different for Vertex vs Fragment Shader, passed ENUMS are different.  
``` cpp
GLint check_compilation_status(GLuint shaderID) {
	GLint success{};
	glGetShaderiv(shaderID, GL_COMPILE_STATUS, &success); // Differences: glGetShaderiv, GL_COMPILE_STATUS
	return success;
}

GLint check_linking_status(GLuint shaderID) {
	GLint success{};
	glGetProgramiv(shaderID, GL_LINK_STATUS, &success); // Differences: glGetProgramiv, GL_LINK_STATUS
	return success;
}
```

We can propagate state into  
`check_compilation_success(id, GL_COMPILE_STATUS)`  
and  
`check_linking_success(id, G_LINK_STATUS)`  
This leaves with one difference left either `glGetShaderiv` or `glGetProgramiv` calls  

Next is to use Generic Template Classes!  
We want to pass in a `Fn` which could be one of two things. We template on this class function with this arbitrary type. This is a function type.
``` cpp
template<class Fn>
GLint check_status(GLuint shaderID, GLenum status, Fn getStatus) { // Functions have a type! wtf
    GLint success{};
    getStatus(shaderID, status, &success);
    return success;
}
```

`Fn` is an arbitrary type!  
`check_status(id, GL_COMPILE_STATUS, glGetShaderiv);`  
`check_status(idm GL_LINK_STATUS, glGetProgramiv);`  

Constrain the Type  
We know this FN can only be invoked with GLuint, GLenum, GLint*
``` cpp
template<std::invocable<GLuint, GLenum, GLint*> Fn>
GLint check_status(GLuint shaderID, GLenum status, Fn getStatus) {
	GLint success{};
	getStatus(shaderID, status, &success);
	return success;
}
```
If you try to pass in something inappropriate, the compiler will fail fast
`check_status(id, GL_COMPILE_STATUS, "Hello world!");`




# Coding Time
branch: start_lecture_4  
Introduced a dependency on glad  
Pulled code from LearnOpenGL to render a triangle  

## 1. Refactor Generalize `check_compilation_success(GLuint shaderID)`  
Move all the compilation and error checking into `check_compilation_success()`  
Change parameter to add `GLuint shaderID` to parse in to check compilation  
Change `success` to `GLint`, and initialize it to default value with `success{};`  
Change `infoLog` to `GLchar` and move it into the proper scope when there is an error  

Refine by generalizing into `check_compilation_status(GLuint shaderID)`  
Therefore you check_compilation_success -> then this calls check_compilation_success  
I feel like this is a little redundant.  

The error message is either SHADER/FRAGMENT COMPILATION_FAILED.   
This needs to be included as an argument. Add `std::string_view shaderType` as another argument to use in the error messages  
This isn't great but will returned to later.  


# 2.  Refactor Generalize Linking Functions  
This is a little more subtle because the functions are either `glGetProgramiv` or `glGetProgramInfo` but the rest is the same  
Create the generalized functions  `GLint check_linking_status(GLuint id) { ... ` and `GLint check_linking_success(GLuint shaderID)`  


# 3. Templating the Generalizations!  
Now we have some almost identical functions that can be templated  
`GLint check_compilation_status(GLuint shaderID) `  
`GLint check_linkning_status(GLuint id)`  
and  
`GLint check_compilation_success(GLuint shaderID, std::string_view shaderType)`  
`GLint check_linking_success(GLuint shaderID)`  

Templating function!  
``` cpp
template<class GetStatus>
GLint check_status(GLuint id, GLenum Status, GetStatus getStatus) {
    GLint success{};
    getStatus(id, status, &success); // this is the templated function call
    return success;
}
/*
Now you should be able to call 
check_status(shaderID, GL_COMPILE_STATUS, glGetShaderiv);
and
check_status(id, GL_LINK_STATUS, getProgramiv);

*/
```

Generalizing the compilation / linking success  
One takes a `shaderType` as params for error message, the other doesn't as it is linking failure. We can stil include it  
There is another difference in error message either `COMPILATION_FAILURE` or `LINKING_FAILURE`, we can include it as `buildStage`  
They also need their `GLenum status` as params to pass into their respective functions  

``` cpp
template<class GetStatus, class GetInfoLog>
void check_success(GLuint id, std::string_view name, std::string_view buildStage, GLenum status, GetStatus getStatus, GetInfoLog getInfoLog) {
    if(!check_status(id, status, getStatus)) {
        GLchar infoLog[512]{};
        getInfoLog(id, 512, NULL, infoLog);
        throw std::runtime_error{std::format("ERROR::SHADER::{}::{}_FAILED\n{}\n",name,buildStage,infoLog)};
    }
}

// Now your compilation / linking success functions are now merely
void check_compilation_success(GLuint shaderID, std::string_view shaderType) {
    check_success(shaderID, shaderType, "COMPILATION", GL_COMPILE_STATUS, glGetShaderiv, glGetShaderInfoLog);
}

void check_linking_success(GLuint shaderID) {
    check_success(shaderID, "PROGRAM", "LINKING", GL_LINK_STATUS, glGetProgramiv, glGetProgramInfoLog);
}
```
As a matter of fact, you can now actually delete the two individual `check_compilation_status()` and `check_linking_status()` functions as EVERYTHING has been generalized.


Overall:
Now the actual `DemoMain.cpp` main is a lot cleaner - Duplicated code is now gone. All the differences have been localised in the smallest scope.