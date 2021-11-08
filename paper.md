# Destructive member functions

## 0. Paper history

* r0: initial draft
* r1: change to destructive *member functions*

## 1. Abstract

We propose a mechanism to designate member functions (including conversion operators) as *destructive*; that is, when called on a prvalue object they are *assumed* to take responsibility for destructing that object; plus additional special member functions, enabling *destructive move* operations that are efficient, safe and composable.

## 2. Motivation

Binding prvalues to rvalue references for move construction and move assignment falls short in three distinct but related cases:

* *trivially relocatable* types, canonically `std::unique_ptr`;
* *never-empty* types, such as `gsl::non_null`;
* *expensive-empty* types such as `std::list` in some implementations.

In the first case, the issue is that while the move constructor and destructor of `unique_ptr` are non-trivial, the code they execute is unnecessary in the case where the destructor of the source object is called immediately after. This inhibits optimizations such as passing in registers (cf. trivial_abi) and the use of memmove when reallocating or inserting `vector`.

In the second case, the requirement that move construction leave the source object in a valid state directly conflicts with the intent of `non_null`: when the default constructor of the wrapped type produces a null value the only result will be that the source object is left in a *singular* state where it can be destructed but otherwise does not fulfil the invariants of the class.

In the third case, the empty state requires memory allocation or some other expensive operation, meaning that the move constructor is non-`noexcept` and results in a pointless operation that is often immediately undone by the destructor of the source object (even if it may occasionally be elided). Again there is a temptation to leave the source object in a singular state.

Previous proposals have attempted to resolve (parts of) this problem but have failed to be adopted for various reasons.

We submit that a solution must (a) cover all 3 cases mentioned above; (b) be (as far as practicable) safe, making it easy to avoid leaks, double-frees and other errors; and (c) be composable, in that user code should benefit, with a minimum and ideally zero (Rule of Zero) code change required.

Any solution should be aware of differences between platform ABIs, in particular callee-destroy (Windows) vs. caller-destroy (Itanium).

It should not require unreasonable changes to compilers such as tracking variable lifetimes through functions.

Finally, such a solution should work with other library facilities, such as `optional` and `tuple`; for example, it should be possible to provide a `make_from_tuple` that works with *never-empty* argument types.

## 3. Mechanism

### 3.1. Overview

The proposal is that a member function (possibly a conversion operator) may be designated *destructive* by the `~` sigil, appearing in place of cvref qualification:

    int S::destructure() ~;
    S::operator int() ~;

A destructive member function has the following properties:

* it may *only* be called on prvalues, either explicitly via class member access or in a conversion;
* it is *assumed* to destruct its operand, and thus suppresses emission of the call to destructor on its prvalue temporary.
* it overloads with other member functions of the same name, and is preferred on prvalue temporaries.

Building on this, a prvalue conversion operator (to the same class type) and prvalue assignment operator are added as (sixth and seventh) special member functions:

    S::operator S() ~;
    S& S::operator=(S);

(The latter is possible to write today; the change is to make it special.)

As usual, these special member functions may be declared or defined as defaulted or deleted, and traits are provided to detect when they are available and trivial.

### 3.2. Usage

It may fairly be asked why the prvalue conversion operator is necessary; after all, whenever a prvalue is used to initialize a variable it will initialize that variable directly without any need for an additional constructor call. The answer is that it can be invoked when an *xvalue* is to be moved and immediately destroyed, either by the compiler (optionally, as elision), or by library or user code, where that code can ensure the xvalue is not double-destructed.

The elision rule is that when an id-expression appears in a return statement, in addition to the option of calling its move constructor, the compiler may instead call its prvalue conversion operator, eliding its destructor.

Usage on xvalues in user code is via a new library function, set out below.

### 3.3. Destructive member functions

Any destructive member function is `noexcept` by default, but may be marked `noexcept(false)`. In this case, if it throws it is required to leave its object argument in a destrucible state, since it is considered to have failed.

A destructive member function is expected to ensure that every base class and data member is destructed, either by destructor call syntax or by destructive move via a library function (see below). 

Alternatively if this is considered excessively unsafe, we suggest a syntax allowing the user to indicate which bases and members it expects to destruct within the body, the compiler destructing the remainder:

    struct D : B, C {
        int x, y;
        long destructure() ~ : ~B, ~x { x.~int(); B::~B(); return 0l; } // compiler destructs y, C
    };

### 3.4. Defaulted special member functions

The special member functions are declared as defaulted or deleted as with the move constructor and move assignment operator; similarly for triviality.

When defined or declared as defaulted, the default implementation for class types is memberwise. Since the default prvalue conversion operator is noexcept, there is no need to handle cleanup; `noexcept(true)` is ignored (or possibly ill-formed?).

For scalars, the builtin behavior is to return or assign the source value respectively; the source value is not altered.

## 4. Library support

### 4.1. Magic function

We propose non-ergonomic (for use by library authors only) magic function(s) for invoking destructive move on an object occupying a region of storage, possibly named as follows:

* `template<class T> constexpr T move_and_destroy_at(T*) noexcept(*see below*);`
* `template<class T> constexpr void construct_and_destroy_at(T*, T*) noexcept(*see below*);`

The former constructs its prvalue return value by from its pointer argument indirected, invoking the prvalue conversion operator of `T` if available (not defaulted as deleted), otherwise a copy constructor (which may be a move constructor) as selected by overload resolution.

The latter is intended to offer the strong exception-safety guarantee, i.e. given on entry its first argument points to uninitialized storage suitable to hold `T` and its second argument points to a `T`, on success this state of affirs is reversed, while on failure it is preserved, and the `T` object retains its former value. If the prvalue conversion operator of `T` is noexcept that will be invoked; otherwise the move or copy constructor is invoked, via a temporary object if necessary for strong exception safety.

Additionally, we propose concepts and associated template variables and type traits:

* `template<class T> concept MoveDestructible = 
* `template<class T> concept NoexceptMoveDestructible = MoveDestructible<T> && requires(T* p) { requires noexcept(move_and_destroy_at(p)); };
* `template<class T> inline constexpr bool is_noexcept_move_destructible_v = NoexceptMoveDestructible<T>;
* `template<class T> using is_noexcept_move_destructible = bool_constant<is_noexcept_move_destructible_v<T>>;

### 4.2. Standard library changes

We suggest modifications and additions to the Standard library that may usefully employ this new facility.

## 5. Examples

## 6. Wording
