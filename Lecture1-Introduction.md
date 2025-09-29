# Lecture 1 Introduction

Instancing
- Rendering jupiter's asteroid belt as instanced rocks - > 10,000 individual rocks but they are all the same object just transformed and had 1 draw call on the GPU

Recursive Instancing
- Instanced cubes on a quad as a texture then instanced again

Graph Theory
- For the recursive instancing, firstly you have to render a model (e.g. a backpack - vertices, textures, transform, camera specifications, then shader programs), then if you want to render recursive instances, you have to take that output to re-render. If you abstract that out, then you have two nodes and edges, you can create a dependency directlyed acyclic graph - then you can do things like topological sorting.

Setting up the project:
- Getting TBB issue, the lines aren't updated on the google docs, these lines are the ones menat to be commented out:
```
find_package(TBB REQUIRED)
target_linke_libraries(${target} PUBLIC TBB::tbb)
```
After that there was a CMake error being thrown for compiler_feature cxx_std_26 is not known
Change these both to cx_std_23 or whatever machine version
```
FUNCTION(sequoia_compile_features target)
    if(WIN32)
        target_compile_features(${target} PUBLIC cxx_std_23)
    else()
    target_compile_features(${target} PUBLIC cxx_std_23)
```

Running demo:
```
cmake .
make .
./Demo
```