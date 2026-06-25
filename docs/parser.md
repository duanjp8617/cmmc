The cmmc compiler contains 2 parsers for 2 different languages, the SysY and the SPL. I'll only talk about SysY here. 

# SysY language structure.

Look very much like c language.

Basic types: void int char float double uint(8|16|32|64)_t, int(8|16|32|64)_t.

Support basic struct definition, and nested struct defintion

```c
struct f {
    struct y {
        int i;
    } j;
    int i;
};
```

Support arrays, only inner-most arrays can be set without a size, indicating a pointer

```c
int c[];
int c_2d[1][2];
int pointer_to_array[][3]; // a pointer to int[3] arrays
```

Expressions, statement, function definitions look like C programming language.

# AST structure

The program is just a list of variant types, containing FunctionDefinition, FunctionDeclaration, GlobalVarDefinition, StructDefinition.

Then for each one of the type, cmmc updates the context and emit the IR code. 

## Struct definition

For struct definition, the parser wil first see STRUCT ID. It will  construct TypeRef first, which contains the struct name string, a classifier indicating that this is a struct, and a qualifier (cmmc places const and signess to qualifer). 

Then, if the STRUCT ID is followed by LC DefList RC, then cmmc will insert a struct definition to the program list. 


# Emitting IRs

## Struct definition 

For each named variable list, getting the type of the named variabled first, and then construct `StructField`, then store `StructField` in a vector. 

Note that the qualifier of the named variable is lost, so the signess information about the struct member variable is actually missing. 

Finally, construct the struct type. Insert the struct type to the cache map of the `EmitContext` first. Then add the struct type to the module being emitted. 

# Miscs

## Transforming type name to IR type

Given a type name string, the class of the type (including default and struct, enum is not supported), and the array list, cmmc returns the corresponding IR-level type using:

```shell
const Type* EmitContext::getType(const String& type, TypeLookupSpace space, const ArraySize& arraySize) {
```

If the type's class is default, it translates the type name into the corresponding type. Note that the IR type only differentiates integer bit width, it does not contain signness information (which is stored as a qualifier). 

If the type's class is struct, it looks for the struct type from the cache map using the struct name. Note that struct that is defined early will be processed by EmitIR early and stored in the cache map early. So we will not miss retriving early struct definition. 

After obtaining the base type, EmitIR then transform the base type into array or pointer type. What it does is that it reversely tranverse the array index list, and then construct the array/pointer type accordingly.  


## Array list:

```shell
int a[1][2]
```

The array sizes are recorded in a list. Which is then traversed reversely to construct the array type. 

Therefore, the outmost array index indicates the inner most array dimension (it seems that all array representations are the same, including the pytorch array representation). 

Among all dimensions, only the inner-most index can be avoided, like:

```shell
int a[][2]; // a pointer that pointers to int[2] array
int a[]; // a pointer for int
```

In this case, the array will be treated as a pointer. 

Also, if indexes other than the inner-most one are missing, errors are thrown

```shell
int a[2][]; // EmitIR.cpp detects it and throws error
```

## getRValue