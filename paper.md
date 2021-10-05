# Destructive move for [library]s

## 0. Paper history

## 1. Abstract

## 2. Background

## 3. Mechanism

### 3.1. Special member functions

We propose two additional special member functions, bringing the total up to the (in Western culture) symbolic number of seven:

* *destructive move constructor*, referred to with strawman name `T::T(T&&, std::destroying_move_t) noexcept(*by default*)` and
* *destructive move assignment operator*, referred to as `T& T::operator=(T&&, std::destroying_move_t) noexcept(*by default*)`.

The *destructive move constructor* has the expected behavior that on completion the object being constructed holds the former value of its xvalue argument,
whose lifetime is ended by the operation; that is, it is nilpotent with respect to object number.
While it is noexcept by default, it may throw in which case the source object is *destroyed* without recourse; thus this is a potentially destructive operation from the point
of view of strong exception safety.

We set out the rules by which the special member functions are defaulted, suppressed, deleted, and trivial.
We provide strawman syntax for declaring, defaulting, deleting and user-defining the special member functions.
We explain the behavior of the compiler-generated (for scalar, array, class and union types) and user-defined versions (for class types) of the special member functions.
We discuss the (limited) scenarios in which the compiler may and/or shall invoke the special member functions (viz. move-delete elision).

## 4. Library support

### 4.1. Magic function

We propose non-ergonomic (for use by library authors only) magic function(s) for invoking destructive move on an object occupying a region of storage, possibly named as follows:

* `template<class T> constexpr T move_and_destroy_at(T*) noexcept(*see below*);`
* `template<class T> constexpr void construct_and_destroy_at(T*, T*) noexcept(*see below*);`

The former constructs its prvalue return value by from its pointer argument indirected and cast to xvalue,
invoking the destructive move constructor of `T` if available (not defaulted as deleted),
otherwise a copy constructor (which may be a move constructor) as selected by overload resolution.

The latter is intended to offer the strong exception-safety guarantee, i.e. given on entry its first argument points to uninitialized storage suitable to hold `T`
and its second argument points to a `T`, on success this state of affirs is reversed, while on failure it is preserved, and the `T` object retains its former value.
If the destructive move constructor of `T` is noexcept that will be invoked;
otherwise the move or copy constructor is invoked, via a temporary object if necessary for strong exception safety.

Additionally, we propose concepts and associated template variables and type traits:

* `template<class T> concept MoveDestructible = 
* `template<class T> concept NoexceptMoveDestructible = MoveDestructible<T> && requires(T* p) { requires noexcept(move_and_destroy_at(p)); };
* `template<class T> inline constexpr bool is_noexcept_move_destructible_v = NoexceptMoveDestructible<T>;
* `template<class T> using is_noexcept_move_destructible = bool_constant<is_noexcept_move_destructible_v<T>>;

### 4.2. Standard library changes

We suggest modifications and additions to the Standard library that may usefully employ this new facility.

## 5. Examples

## 6. Wording
