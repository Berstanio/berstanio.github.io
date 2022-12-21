---
layout: post
title: Swift Runtime
date: 2022-12-21
permalink: /swift/
---

## The Swift Runtime:

This post is intended to illustrate a bit of the challenges of swift and how a java runtime can be implemented. Maybe someone will find this helpful!

#### Introduction Java-Native Runtimes:
Before we go into swift, let’s cover a bit the general approach. The core element here is libffi. Libffi lets you dispatch a function call based on an address. 
The API is pretty straight forward, first we define a “ffi_cif”, that describes the layout and conventions of our function.
This can be done with ```ffi_prep_cif (ffi_cif *cif, ffi_abi abi, unsigned int nargs, ffi_type *rtype, ffi_type **argtypes)```.
After that we can dispatch a function call with a function pointer and that cif ```ffi_call (ffi_cif *cif, void *fn, void *rvalue, void **avalues)```
Seems easy enough, doesn't it?  

But how do we glue this now together with jni?
Also that is fairly simple, we can use the libffi closure API. With that, we can generate a closure that will receive arbitrary arguments and can call a function itself again.
So, we can do something like ```ffi_prep_closure_loc(closure, closureCif, javaToSwiftHandler, info, code);```, where javaToSwiftHandler is a function pointer to a function looking like this ```void javaToSwiftHandler(ffi_cif* cif, void* result, void** args, void* user)```.  
NOTE: With "info" we can transport information about the parameter and return type layout. "code" is the function pointer to our closure.
The javaToSwiftHandler now needs to convert the java objects to native values and than calls "ffi_call" with the parameters and the targeted swift function (the function pointer will be passed in the "info" too).  

Now the closure just needs to be registered as native method, and all should work.  


#### How to get swift function pointer
After I laid out the basic about the jni/native method dispatching, now lets find out how we get the swift function pointer.  
This is also straight forward, we need to know the symbol of the function and can than get the address with `dlsym`.  
Let's start with global functions. Suppose we have
```swift
public func testFunc() {
    print("Heeey")
}
```
We can compile that to create a shared library with ```swiftc swiftTest.swift -emit-library -g```.  
Than we can dump out all symbols with ```nm -g -C --defined-only libswiftTest.dylib```.  
We will fine one symbol `$s9swiftTest8testFuncyyF`. We can double check that symbol with `swift demangle s9swiftTest8testFuncyyF` (note, that the leading $ needs to be removed), and we will see `$s9swiftTest8testFuncyyF ---> swiftTest.testFunc() -> ()`.  
So we got the right symbol. And `dlsym(handle, "$s9swiftTest8testFuncyyF")` will give us the correct function pointer.  

Some code that can be used for demonstration is:
``` c++
int main() {
    void* handle = dlopen("libswiftTest.dylib", RTLD_GLOBAL|RTLD_LAZY);

    ffi_cif* cif = new ffi_cif;

    ffi_prep_cif(cif, FFI_DEFAULT_ABI, 0, &ffi_type_void, NULL);

    ffi_call(cif, (void (*)(void))dlsym(handle, "$s9swiftTest8testFuncyyF"), NULL, NULL);

    return 0;
}
```
For further testing around with libffi this example can be used and extended, but I wont come back to it.  

So, we can now dispatch global functions with arbitrary parameter and return types, yay!  
Structs until a size of 32 are passed by value, anything larger will be passed by a struct pointer (but the struct itself will still be cloned).  

Having global function support, lets move on to object functions.

#### What are objects in swift
Objects in swift are defined by a object pointer. The object pointer points to the information/data of the object

#### What are structs in swift
Structs in swift are similiar to c structs. They are passed by value, don't support inheritance and can have "object" functions.

#### Object methods
Usually in object oriented programming, object functions are just global functions, that get their object to act on as the first implicit parameter.  
NOTE: We will cover constructors later, lets assume for now we have a global functions that creates the objects for us and returns them.  

So in theory, both method should be able to get identical called:
```swift
public class TestClass {
    public var field: Int = 4
    
    public func printField() {
        print("Field: \(self.field)")
    }
    
    public init() {
        
    }
}

public func printFieldGlobal(_ par: TestClass) {
    print("Field: \(par.field)")
}
```
However, we will see that we get a segfault when trying to execute the object method, but the global method works fine. How can this be?  
Let's look at the assembly, whether we find something.  
We can get it with `swiftc -S switTest.swift > assembly.asm`  

We will find the following arm64 assembly (simplified) for the object method:
```asm
_$s9swiftTest0B5ClassC10printFieldyyF:
	.cfi_startproc
	sub	sp, sp, #16
	.cfi_def_cfa_offset 16
	str	xzr, [sp, #8]
	str	x20, [sp, #8]
	add	sp, sp, #16
	ret
```

and for the global function:
```asm
_$s9swiftTest16printFieldGlobalyyAA0B5ClassCF:
	.cfi_startproc
	sub	sp, sp, #16
	.cfi_def_cfa_offset 16
	str	xzr, [sp, #8]
	str	x0, [sp, #8]
	add	sp, sp, #16
	ret
```

Spot the difference? The global functions expects the object pointer in the x0 register (which is usual) and the object method in the x20 register.  
A quick look at the docs confirms this: https://github.com/apple/swift/blob/main/docs/ABI/CallConvSummary.rst  
The implicit object pointer is expected to be in x20. To fix this I applied a hack to libffi, that the first parameter gets put into the x20 register, if a specific flag is set.  
The patch would be the following, please forgive the ugliness, I don't really know assembly:
```diff
diff --git a/src/aarch64/ffi.c b/src/aarch64/ffi.c
--- a/src/aarch64/ffi.c	(revision cda21f6d8fa0045f28fbb37b68dafce628765ce1)
+++ b/src/aarch64/ffi.c	(revision e3f656c4782651a705649b2b307332e995363163)
@@ -570,7 +570,7 @@
 	rsize = rtype_size;
     }
   else if (orig_rvalue == NULL)
-    flags &= AARCH64_FLAG_ARG_V;
+    flags &= (AARCH64_FLAG_ARG_V | AARCH64_FLAG_SWIFT_ARG1_IN_X20);
   else if (flags & AARCH64_RET_NEED_COPY)
     rsize = 16;
 
diff --git a/src/aarch64/internal.h b/src/aarch64/internal.h
--- a/src/aarch64/internal.h	(revision cda21f6d8fa0045f28fbb37b68dafce628765ce1)
+++ b/src/aarch64/internal.h	(revision e3f656c4782651a705649b2b307332e995363163)
@@ -61,6 +61,7 @@
 
 #define AARCH64_FLAG_ARG_V_BIT	7
 #define AARCH64_FLAG_ARG_V	(1 << AARCH64_FLAG_ARG_V_BIT)
+#define AARCH64_FLAG_SWIFT_ARG1_IN_X20	(1 << 8)
 
 #define N_X_ARG_REG		8
 #define N_V_ARG_REG		8
diff --git a/src/aarch64/sysv.S b/src/aarch64/sysv.S
--- a/src/aarch64/sysv.S	(revision cda21f6d8fa0045f28fbb37b68dafce628765ce1)
+++ b/src/aarch64/sysv.S	(revision e3f656c4782651a705649b2b307332e995363163)
@@ -79,6 +79,7 @@
 #ifdef FFI_GO_CLOSURES
 	mov	x18, x5			/* install static chain */
 #endif
+	and x20, x4, #256
 	stp	x3, x4, [x29, #16]	/* save rvalue and flags */
 
 	/* Load the vector argument passing registers, if necessary.  */
@@ -95,6 +96,19 @@
 	ldp     x4, x5, [sp, #16*N_V_ARG_REG + 32]
 	ldp     x6, x7, [sp, #16*N_V_ARG_REG + 48]
 
+	cmp x20, #256
+	b.eq SWIFT
+	b CONTINUE
+
+SWIFT:
+	ldp     x0, x1, [sp, #16*N_V_ARG_REG + 8]
+	ldp     x2, x3, [sp, #16*N_V_ARG_REG + 24]
+	ldp     x4, x5, [sp, #16*N_V_ARG_REG + 40]
+	ldp     x6, x7, [sp, #16*N_V_ARG_REG + 56]
+	ldr x20, [sp, #16*N_V_ARG_REG + 0]
+	b CONTINUE
+
+CONTINUE:
 	/* Deallocate the context, leaving the stacked arguments.  */
 	add	sp, sp, #CALL_CONTEXT_SIZE
```

With this patch object function dispatch works fine!  

However, structs object functions are a bit more difficult. If a struct is smaller than 32, the struct is passed by value as the *last* function argument, not the first.
If the struct is larger or equal to 32, it is passed as a pointer to the struct, but in the x20 register (as swiftself). The struct should be still cloned and therefor indirectly passed by value.

With that we also have field support, because looking at the symbols you'll find:
```
$s9swiftTest0B5ClassC5fieldSivs ---> swiftTest.TestClass.field.setter : Swift.Int
$s9swiftTest0B5ClassC5fieldSivg ---> swiftTest.TestClass.field.getter : Swift.Int
$s9swiftTest0B5ClassC5fieldSivM ---> swiftTest.TestClass.field.modify : Swift.Int
```
Getter and setter are obvious and can be dispatched as normal object functions. And I have no idea what modify is :/  

Now we can move on to, how we can even create/init objects/structs.  

#### Constructor

##### Constructor for objects
The constructor is also just a static function:  
`$s9swiftTest0B5ClassCACycfC ---> swiftTest.TestClass.__allocating_init() -> swiftTest.TestClass`  
NOTE: Don't confuse it with the symbol with the lower case `c` symbol at the end, this is just for initing, not allocation.  
However, there is another trick to it, that is hidden. Looking at the llvm bitcode `swiftc -emit-bc swiftTest.swift` we will find:
```
define swiftcc %T9swiftTest0B5ClassC* @"$s9swiftTest0B5ClassCACycfC"(%swift.type* swiftself %0) #0 {
  %2 = call noalias %swift.refcounted* @swift_allocObject(%swift.type* %0, i64 24, i64 7) #2
  %3 = bitcast %swift.refcounted* %2 to %T9swiftTest0B5ClassC*
  %4 = call swiftcc %T9swiftTest0B5ClassC* @"$s9swiftTest0B5ClassCACycfc"(%T9swiftTest0B5ClassC* swiftself %3)
  ret %T9swiftTest0B5ClassC* %4
}
```
Yep, a hidden parameter, that is also passed in the x20 register (see swiftself).  
What we need to pass is the metadata type of the class. But we can easily get it with the `$s9swiftTest0B5ClassCMa ---> type metadata accessor for swiftTest.TestClass` function, so no big hassle.  

##### Constructor for structs
There are two types of struct constructors. If the struct is smaller than 32, the struct is just returned by the constructor (NOTE: Structs only have a init constructor `$s9swiftTest0B6StructVACycfC ---> swiftTest.TestStruct.init() -> swiftTest.TestStruct`)
If the struct is larger or equals than 32, than we need to pass a pre allocated struct pointer to the constructor as first parameter (normal register, not x20).
TODO: Is this even true? And how does 24/32 works?

#### Inheritance
Now that we can create objects of classes, how can we support inheritance?  
Imagine this code:
```swift
public class TestClass {    
    public func printName() {
        print("TestClass")
    }

    public init() {

    }
}

public class BaseClass {
    public func getClassSpecificNumber() -> Int {
        return 1
    }

    public func onlyBaseClass() -> Int {
        return 4
    }

    public init() {

    }
}

public class SubClass : BaseClass {

    public func onlySubClass() -> Int {
        return 3
    }

    public override func getClassSpecificNumber() -> Int {
        return 2
    }
}
```
Suppose we only have bindings for the `TestClass` class, but we got a `SubClass` somewhere as return. How can we dispatch the overriden function correctly, without knowing the symbol we want to call at build time.  
To achieve this, swift uses virtual dispatch.  
Rough explanation about it:  
In virtual dispatch every object gets a table at creating time, where pointer to every function the object has, are stored.  
So, for every object of BaseClass and it ascendents (SubClass e.g.), the object stores a address to the function that should be called at the same offset. So by knowing the offset at compile time, you can get the correct function at runtime.  
I won't go deeper into the tables, there are way better resources already existing.  

So, where exactly is the vtable stored in swift?  
If you dereference the object pointer, you will get a metadata pointer (remember, the one we needed for creating the object in the first place with a constructor). The metadata is class specific.  
In assembly it will look like this:  
BaseClass:
```asm
_$s9swiftTest9BaseClassCMf:
	.quad	_$s9swiftTest9BaseClassCfD
	.quad	_$sBoWV
	.quad	_$s9swiftTest9BaseClassCMm
	.quad	_OBJC_CLASS_$__TtCs12_SwiftObject
	.quad	__objc_empty_cache
	.quad	0
	.quad	__DATA__TtC9swiftTest9BaseClass+2
	.long	2
	.long	0
	.long	16
	.short	7
	.short	0
	.long	120
	.long	16
	.quad	_$s9swiftTest9BaseClassCMn
	.quad	0
	.quad	_$s9swiftTest9BaseClassC03getD14SpecificNumberSiyF <------ VTable start
	.quad	_$s9swiftTest9BaseClassC04onlycD0SiyF
	.quad	_$s9swiftTest9BaseClassCACycfC
```
SubClass:
```asm
_$s9swiftTest8SubClassCMf:
	.quad	_$s9swiftTest8SubClassCfD
	.quad	_$sBoWV
	.quad	_$s9swiftTest8SubClassCMm
	.quad	_$s9swiftTest9BaseClassCMf+16 <----- Pointer to the superclass metadata, relevant for finding suiting binding (later)
	.quad	__objc_empty_cache
	.quad	0
	.quad	__DATA__TtC9swiftTest8SubClass+2
	.long	2
	.long	0
	.long	16
	.short	7
	.short	0
	.long	128
	.long	16
	.quad	_$s9swiftTest8SubClassCMn
	.quad	0
	.quad	_$s9swiftTest8SubClassC03getD14SpecificNumberSiyF <------ VTable start
	.quad	_$s9swiftTest9BaseClassC04onlycD0SiyF
	.quad	_$s9swiftTest8SubClassCACycfC
	.quad	_$s9swiftTest8SubClassC04onlycD0SiyF
```
As you can see, the function pointer are ordered after definiton order, from BaseClass -> SubClass.  
So by offsetting the metadata pointer by e.g. 88, we get the object dependend function pointer to the overriden method, without knowing, that the class was overriden in the first place.  
This is the whole magic for inheritance.  

But the questions is now, how can we find the best fitting java binding for swift object.

#### How to find the best binding 
Suppose function A declares a return of type B, but it actually returns C, what we have a binding for.  
Since every object has a metadata type pointer, we can just use a table, mapping metadata pointer to binding classes.  
And struct don't have inheritance, so we know we already have the highest binding.  
For classes we can find the highest possible binding, since we can traverse the metadata type hierachy.  
At the +8 offset on the metadata is a pointer to the metadata of the super class.  

#### Protocol
So, now about protocols. Protocols are like interfaces in swift, to which classes **and structs** can conform to.  
What makes protocols complicated to deal with is, the fact, that also structs can conform to that.  
E.g. we want a array of protocol types. structs don't have a consistent size, unlike objects. So having a array of possibly different sized structs + objects is not possible.  
So, how does swift solves this? They put it into a "existential container".  This is basically a fixed size container for the structs and classes.  
The layout is the following:  

3 x word-sized values  
metadata pointer  
protocol witness table pointer  

The three values store either the object pointer (for classes), the whole struct if it fits or a pointer to a heap allocated struct.  
The pointer to the heap allocated struct seems to be weird, since the actual struct just starts at a offset of 16. It's up to investigation, why.  
// https://developer.apple.com/videos/play/wwdc2016/416/?time=1472 38:13 maybe?  
The metadata pointer points to the metadata.  
And the pwt stores pointer to functions, similiar to the vtable of classes.  

So, this is difficult. How do we wanna deal with it, when we get a protocol type returned?
We need to differenciate:
1. We have a struct/class and we have bindings for that (we can know that over the metadata pointer)  
    Than we can just unbox the EC (existential container) and we have a struct/class binding downcasted to a interface in java.  
	For structs we know the layout of the EC, because we know the size of it. Important to consider for structs is probably also the management.   
	So we will *probably* need to make a copy of the struct in every case.  
    Method dispatching will work perfectly with static/virtual dispatch for structs/classes.
2. We have a struct/class and we don't have bindings  
    Than we need to create a proxy for that protocol java interface, and dispatch methods with the pwt. The pwt offset for the protocol is also known at compile time and is than just simple virtual dispatch.  
	Also here is the lifetime of the EC important, it may be that it needs to be cloned too.  
	However, for classes it is good enough if we find the first conforming class in the inheritance hirachy. And after that, we can easily use virtual dispatch.  

What is, when we want to call a function with a protocol parameter:
1. We have bindings for what we want to pass, so we need to box them in a EC ourselve.  
    How to properly box it, will be explained later.
2. We pass a already boxed proxy back, so we just need to pass it back.  
NOTE: Important is here also, that the EC is passed by value, not by reference.

What is also complicated is the fact, that a function can expect a parameter, that conforms to protocol A and B. In java we sadly don't have a clean way to represent this.

#### How to box a java binding in a EC
While I first thought this would be easy, it was suprisingly complicated.  
So what we need is: The metadata pointer and the protocol witness table.  
The metadata pointer is easy to obtain, for classes we can just dereference the class pointer. For structs we need to create a mapping on startup for class -> metadatapointer.  
The witness table was complicated:  
The swift runtimes has a method called "swift_conformsToProtocol", which retrieves a metadata pointer and a protocol descriptor pointer and returns the pointer to the witness table.  
Every protocol has it's own descriptor, and we can know the symbol of it at compile time. So we just need to resolve the symbol at runtime.  
Know we have everything we need and create the EC. It seems to be just a unaligned array of word sized values, so easy enough to create.  
Important to consider is also, the earlier mentioned struct offset of 16 for heap allocated structs.  

#### Enums


#### Error Handling


#### Implementing java side native objects
So far I have only covered pure binding classes. However, the Runtime should also provide capabilities to store states on the java side and should be able to override methods.  
NOTE: We should follow swifts contract, and don't allow overriding of structs.  
Implementing the first one should be easy, we just need a direct mapping from one object pointer to a java object. This way, we can always restore the same state for the same swift object.  
The second one is however tricky. How can we let the native side dispatch a method call to our java method?  
Basically, we need a custom metatdata or pwt, which we will attach to our custom object. On that metadata/pwt we can patch the function pointer to libffi closures. It's basically the same procedure
as how the jni methods are bridged, just the other way around.  
We can do that on runtime. Basically, we just need to get the metadata pointer from the parent binding class, clone that and patch a few fields. If we want to implement a protocol, we can create a dummy struct metadata pointer. We can't override structs, so no need to deal with that.  
Besides the nominal type descriptor, creating the metadata at runtime is quite straight forward. I put a reversed engeneerd layout for a class below, on what needs to be patched:
```asm
_$s15InheritanceTest013DummyClassForA0CMf:
	.quad	_$s15InheritanceTest013DummyClassForA0CfD // deinit
	.quad	_$sBoWV
	.quad	_$s15InheritanceTest013DummyClassForA0CMm // meta class -> start metadata* (see below)
	.quad	_$s15InheritanceTest9BaseClassCMf+16 // fill in here parent class metadata*
	.quad	__objc_empty_cache
	.quad	0
	.quad	__DATA__TtC15InheritanceTest24DummyClassForInheritance+2 // Some random data (see below)
	.long	2
	.long	0
	.long	24
	.short	7
	.short	0
	.long	152
	.long	16
	.quad	_$s15InheritanceTest013DummyClassForA0CMn // nominal type descriptor. Compleeetly different and no real docs about the layout....
	.quad	0
	.quad	16
	.quad	_$s15InheritanceTest9BaseClassC03getD14SpecificNumberSiyF
	.quad	_$s15InheritanceTest9BaseClassC9baseFieldSivg
	.quad	_$s15InheritanceTest9BaseClassC9baseFieldSivs
	.quad	_$s15InheritanceTest9BaseClassC9baseFieldSivM
	.quad	_$s15InheritanceTest9BaseClassC03getC5FieldSiyF
	.quad	_$s15InheritanceTest013DummyClassForA0CACycfC // init

_$s15InheritanceTest013DummyClassForA0CMm:
	.quad	_OBJC_METACLASS_$__TtCs12_SwiftObject
	.quad	_$s15InheritanceTest9BaseClassCMm // whats this? Metaclass of the parent? How does it behave on multiple inheritance?
	.quad	__objc_empty_cache
	.quad	0
	.quad	__METACLASS_DATA__TtC15InheritanceTest24DummyClassForInheritance // Whats this? But doesn't matter, since it seems identical 🥳


__DATA__TtC15InheritanceTest24DummyClassForInheritance:
	.long	128
	.long	24
	.long	24
	.long	0
	.quad	0
	.quad	l_.str.10 // pointer to ascii class name
	.quad	0
	.quad	0
	.quad	0 // __IVARS__TtC15InheritanceTest9BaseClass -> no clue, probably object variable info
	.quad	0
	.quad	0
```
TODO: Nominal type  
The layout of a struct is also fairly simple:
```asm
_$s8EnumTest0B6StructVMf:
	.quad	_$sytWV
	.quad	512
	.quad	_$s8EnumTest0B6StructVMn // Nominal type
```
TODO: Nominal type for structs too

#### Memory Managment
Since swift also uses ARC, the same way ObjC does, I will follow this nice article: https://www.noisyfox.io/natj-memory-management.html 
https://clang.llvm.org/docs/AutomaticReferenceCounting.html
HOWEVER, Swift seems to not support hooking in on release calls, which is veeery unfortunate. Soo, I guess we need to implement something like a little GC ourselves for inherited objects?  
Since we have a object* -> java object map, we can just scan through the map, check whether a object* has a ref count of 1, and if yes put it into a weak ref map. But thats probably a heavy operation :/  
