 * Name: `union_types_v2`
 * Date: 2019-09-02
 * Author: Nikita Popov <nikic@php.net>
 * Proposed Version: PHP 8.0
 * RFC PR: [php-src/rfcs#0000](https://github.com/php-src/rfcs/pull/0000)

# Introduction

A "union type" accepts values of multiple different types, rather than a single one. PHP already supports two special union types:

 * `Type` or `null`, using the special `?Type` syntax.
 * `array` or `Traversable`, using the special `iterable` type.

However, arbitrary union types are currently not supported by the language. Instead, phpdoc annotations have to be used, such as in the following example:

```php
class Number {
    /**
     * @var int|float $number
     */
    private $number;

    /**
     * @param int|float $number
     */
    public function setNumber($number) {
        $this->number = $number;
    }

    /**
     * @return int|float
     */
    public function getNumber() {
        return $this->number;
    }
}
```

The [statistics section](#statistics) shows that the use of union types is indeed pervasive in the open-source ecosystem, as well as PHP's own standard library.

Supporting union types in the language allows us to move more type information from phpdoc into function signatures, with the usual advantages this brings:

 * Types are actually enforced, so mistakes can be caught early.
 * Because they are enforced, type information is less likely to become outdated or miss edge-cases.
 * Types are checked during inheritance, enforcing the Liskov Substitution Principle.
 * Types are available through Reflection.
 * The syntax is a lot less boilerplate-y than phpdoc.

After generics, union types are currently the largest "hole" in our type declaration system.

# Proposal

Union types are specified using the syntax `T1|T2|...` and can be used in all positions where types are currently accepted:

```php
class Number {
    private int|float $number;

    public function setNumber(int|float $number): void {
        $this->number = $number;
    }

    public function getNumber(): int|float {
        return $this->number;
    }
}
```

## Supported Types

Union types support all types currently supported by PHP, with some caveats outlined in the following.

### `void` type

The `void` type can never be part of a union. As such, types like `T|void` are illegal in all positions, including return types.

The `void` type indicates that the function has no return value, and enforces that argument-less `return;` is used to return from the function. It is fundamentally incompatible with non-void return types.

What is likely intended instead is `?T`, which allows returning either `T` or `null`.

### `false` pseudo-type

While we nowadays encourage the use of `null` over `false` as an error or absence return value, for historical reasons many internal functions continue to use `false` instead. As shown in the [statistics section](#statistics), the vast majority of union return types for internal functions include `false`.

A classical example is the `strpos()` family of functions, which returns `int|false`.

While it would be possible to model this less accurately as `int|bool`, this gives the false impression that the function can also return a `true` value, which makes this type information significantly less useful to humans and static analyzers both.

For this reason, support for the `false` pseudo-type is included in this proposal. A `true` pseudo-type is *not* part of the proposal, because similar historical reasons for its necessity do not exist.

The `false` pseudo-type cannot be used as a standalone type, it can only be used as part of a union.

### Nullable union types

Nullable types in PHP use the syntax `?T`. Nullable union types are required to be written as `?(T1|T2)`, while variations such as `?T1|T2`, `T1|?T2` or even `?T1|?T2` are disallowed. This avoids issues of "symmetry" where the nullability is syntactically associated with a specific type, while in reality it could equally apply to all.

An alternative would be to allow an explicit `null` type for use in unions only, such that the above example could be written `T1|T2|null`, and the existing syntax `?T` could be written `T|null`. While this might have been a good option if union types were introduced earlier, I think it is preferable not to introduce a second type nullability syntax at this point in time.

### Duplicate types

Each literal type may only occur once inside a union. As such, types like `int|string|INT` are not permitted. Additionally, only one of `false` and `bool` may be used.

This is a purely syntactical restriction that is intended to catch simple bugs in the type specification. It does not ensure that the union type is in some sense "minimal".

For example, if `A` and `B` are class aliases, then `A|B` remains a legal union type, even though it could be reduced to either `A` or `B`. Similarly, if `class B extends A {}`, then `A|B` is also a legal union type, even though it could be reduced to just `A`. Detecting these cases would require loading all types at the point of declaration.

### Type grammar

Excluding the special `void` type, PHP's type syntax may now be described by the following grammar:

```
type: simple_type
    | union_type
    | "?" simple_type
    | "?" "(" union_type ")"
    ;

union_type: simple_type "|" simple_type
          | union_type "|" simple_type
          ;

simple_type: "false"          # only legal in unions
           | "bool"
           | "int"
           | "float"
           | "string"
           | "array"
           | "object"
           | "iterable"
           | "callable"       # not legal in property types
           | "self"
           | "parent"
           | namespaced_name
           ;
```

At this point in time, parentheses in types are only allowed in the one case where they are necessary, which is the `?(T1|T2|...)` syntax. With further extensions to the type system (such as intersection types) it may make sense to allow parentheses in more arbitrary positions.

## Variance

Union types follow the existing variance rules:

 * Return types are covariant (child must be subtype).
 * Parameter types are contravariant (child must be supertype).
 * Property types are invariant (child must be subtype and supertype).

The only change is in how union types interact with subtyping, with three additional rules:

 * A union `U_1|...|U_n` is a subtype of `V_1|...|V_n` if for each `U_i` there exists a `V_j` such that `U_i` is a subtype of `V_j`.
 * The `iterable` type is considered to be the same (i.e. both subtype and supertype) as `array|Traversable`.
 * The `false` pseudo-type is considered a subtype of `bool`.

In the following, some examples of what is allowed and what isn't are given.

### Property types

Property types are invariant, which means that types must stay the same during inheritance. However, the "same" type may be expressed in different ways. Prior to union types, one such possibility was to have two aliased classes `A` and `B`, in which case a property type may legally change from `A` to `B` or vice versa.

Union types expand the possibilities in this area: For example `int|string` and `string|int` represent the same type. The following example shows a more complex case:

```php
class A {}
class B extends A {}

class Test {
    public A|B $prop;
}
class Test2 extends Test {
    public A $prop;
}
```

In this example, the union `A|B` actually represents the same type as just `A`, and this inheritance is legal, despite the type not being syntactically the same.

Formally, we arrive at this result as follows: First, `A` is a subtype of `A|B`, because it is a subtype of `A`. Second, `A|B` is a subtype of `A`, because `A` is a subtype of `A` and `B` is a subtype of `A`.

### Adding and removing union types

It is legal to remove union types in return position and add union types in parameter position:

```php
class Test {
    public function param1(int $param) {}
    public function param2(int|float $param) {}

    public function return1(): int|float {}
    public function return2(): int {}
}

class Test2 extends Test {
    public function param1(int|float $param) {} // Allowed: Adding extra param type
    public function param2(int $param) {}       // FORBIDDEN: Removing param type

    public function return1(): int {}           // Allowed: Removing return type
    public function return2(): int|float {}     // FORBIDDEN: Adding extra return type
}
```

### Variance of individual union members

Similarly, it is possible to restrict a union member in return position, or widen a union member in parameter position:

```php
class A {}
class B extends A {}

class Test {
    public function param1(B|string $param) {}
    public function param2(A|string $param) {}

    public function return1(): A|string {}
    public function return2(): B|string {}
}

class Test2 extends Test {
    public function param1(A|string $param) {} // Allowed: Widening union member B -> A
    public function param2(B|string $param) {} // FORBIDDEN: Restricting union member A -> B

    public function return1(): B|string {}     // Allowed: Restricting union member A -> B
    public function return2(): A|string {}     // FORBIDDEN: Widening union member B -> A
}
```

Of course, the same can also be done with multiple union members at a time, and be combined with the addition/removal of types mentioned previously.

## Coercive typing mode

When `strict_types` is not enabled, scalar type declarations are subject to limited implicit type coercions. These are problematic in conjunction with union types, because it is not always obvious which type the input should be converted to. For example, when passing a boolean to an `int|string` argument, both `0` and `""` would be viable coercion candidates.

If the exact type of the value is not part of the union, then the target type is chosen in the following order of preference:

1. `int`
2. `float`
3. `string`
4. `bool`

If the type both exists in the union, and the value can be coerced to the type under PHPs existing type checking semantics, then the type is chosen. Otherwise the next type is tried.

As an exception, if the value is a string and both `int` and `float` are part of the union, the preferred type is determined by the existing "numeric string" semantics. For example, for `"42"` we choose `int`, while for `"42.0"` we choose `float`.

### Examples

```php
// int|string
42    --> 42          // exact type
"42"  --> "42"        // exact type
new ObjectWithToString --> "Result of __toString()"
                      // object never compatible with int, fall back to string
42.0  --> 42          // float compatible with int
42.1  --> 42          // float compatible with int
1e100 --> "1.0E+100"  // float too large for int type, fall back to string
INF   --> "INF"       // float too large for int type, fall back to string
true  --> 1           // bool compatible with int
[]    --> TypeError   // array not compatible with int or string

// int|float|bool
"45"    --> 45        // int numeric string
"45.0"  --> 45.0      // float numeric string
"45X"   --> 45 + Notice: Non well formed numeric string
                      // int numeric string
""      --> false     // not numeric string, fall back to bool
"X"     --> true      // not numeric string, fall back to bool
[]      --> TypeError // array not compatible with int, float or bool
```

### Alternatives

There are two main alternatives to the preference-based approach used by this proposal:

The first is to specify that union types *always* use strict typing, thus avoiding any complicated coercion semantics altogether. Apart from the inconsistency this introduces in the language, this has two main disadvantages: First going from a type like `int` to `int|float` would actually *reduce* the number of valid inputs, which is highly unintuitive. Second, it breaks the variance model for union types, because we can no longer say that `int` is a subtype of `int|float`.

The second is to perform the coercions based on the order of types. This would mean that `int|string` and `string|int` are distinct types, where the former would favor integers and the latter strings. Depending on whether exact type matches are still prioritized, the string type would *always* be used for the latter case. Once again, this is unintuitive and has very unclear implications for the subtyping relationship on which variance is based.

## Property types and references

References to typed properties with union types follow the semantics outlined in the [typed properties RFC](https://wiki.php.net/rfc/typed_properties_v2#general_semantics):

> If typed properties are part of the reference set, then the value is checked against each property type. If a type check fails, a TypeError is generated and the value of the reference remains unchanged.
>
> There is one additional caveat: If a type check requires a coercion of the assigned value, it may happen that all type checks succeed, but result in different coerced values. As a reference can only have a single value, this situation also leads to a TypeError.

The [interaction with union types](https://wiki.php.net/rfc/typed_properties_v2#future_interaction_with_union_types) was already considered at the time, because it impacts the detailed reference semantics. Repeating the example given there:

```php
class Test {
    public int|string $x;
    public float|string $y;
}
$test = new Test;
$r = "foobar";
$test->x =& $r;
$test->y =& $r;

// Reference set: { $r, $test->x, $test->y }
// Types: { mixed, int|string, float|string }

$r = 42; // TypeError
```

The basic issue is that the final assigned value (after type coercions have been performed) must be compatible with all types that are part of the reference set. However, in this case the coerced value will be `int(42)` for property `Test::$x`, while it will be `float(42.0)` for property `Test::$y`. Because these values are not the same, this is considered illegal and a `TypeError` is thrown.

An alternative approach would be to cast the value to the only common type `string` instead, with the major disadvantage that this matches *neither* of the values you would get from a direct property assignment.

## Reflection

To support union types, a new class `ReflectionUnionType` is added:

```php
class ReflectionUnionType extends ReflectionType {
    /** @return ReflectionType[] */
    public function getTypes();

    /** @return bool */
    public function allowsNull();

    /** @return string */
    public function __toString();
}
```

The `getTypes()` method returns an array of `ReflectionType`s that are part of the union. The types may be returned in an arbitrary order that does not match the original type declaration. The types may also be subject to equivalence transformations.

For example, the type `int|string` may return types in the order `["string", "int"]` instead. The type `iterable|array|string` might be canonicalized to `iterable|string` or `Traversable|array|string`. The only requirement on the Reflection API is that the ultimately represented type is equivalent.

The `allowsNull()` method returns whether the union additionally contains the type null. A possible alternative would be to introduce a separate `ReflectionNullableType` to represent the `?T` wrapper explicitly. I would prefer this, but we would probably not be able to use it for non-union types for backwards compatibility reasons.

The `__toString()` method returns a string representation of the type that constitutes a valid code representation of the type in a non-namespaced context. It is not necessarily the same as what was used in the original code. Notably this *will* contain the leading `?` for nullable types and not be bug-compatible with `ReflectionNamedType`.

# Backwards Incompatible Changes

This RFC does not contain any backwards incompatible changes. However, existing ReflectionType based code will have to be adjusted in order to support processing of code that uses union types.

# Future Scope

## Intersection Types

Intersection types are logically conjugated with union types. Instead of requiring that (at least) a single type constraints is satisfied, all of them must be.

For example `Traversable|Countable` requires that the passed value is either `Traversable` or `Countable`, while `Traversable&Countable` requires that it is both.

## Mixed Type

The `mixed` type allows to explicitly annotate that any value is acceptable. While specifying no type has the same behavior on the surface, it does not make clear whether the type is simply missing (because nobody bothered adding it yet, or because it can't be added for backwards compatibility reasons), or whether genuinely any value is acceptable.

We've held off on adding a `mixed` type out of fear that it would be used in cases where a more specific union could have been specified. Once union types are supported, it would probably also make sense to add the `mixed` type.

## Literal Types

The `false` pseudo-type introduced in this RFC is a special case of a "literal type", such as supported by [TypeScript](https://www.typescriptlang.org/docs/handbook/advanced-types.html#string-literal-types). They allow specifying enum-like types, which are limited to specific values.

```php
type ArrayFilterFlags = 0|ARRAY_FILTER_USE_KEY|ARRAY_FILTER_USE_BOTH;
array_filter(array $array, callable $callback, ArrayFilterFlags $flag): array;
```

Proper enums are likely a better solution to this problem space, though depending on the implementation they may not be retrofitted to existing functions for backwards-compatibility reasons.

## Type Aliases

As types become increasingly complex, it may be worthwhile to allow reusing type declarations. There are two general ways in which this could work. One is a local alias, such as:

```php
use int|float as number;

function foo(number $x) {}
```

In this case `number` is a symbol that is only visible locally and will be resolved to the original `int|float` type during compilation.

The second possibility is an exported typedef:

```php
namespace Foo;
type number = int|float;

// Usable as \Foo\number from elsewhere
```

# Proposed Voting Choices

Simple yes/no vote.

# Statistics

To illustrate the use of union types in the wild, the use of union types in `@param` and `@return` annotations in phpdoc comments has been analyzed.

In the top two thousand composer packages there are:

  * 25k parameter union types: [Full JSON data](https://gist.github.com/nikic/64ff90c5038522606643eac1259a9dae#file-param_union_types-json)
  * 14k return union types: [Full JSON data](https://gist.github.com/nikic/64ff90c5038522606643eac1259a9dae#file-return_union_types-json)

In the PHP stubs for internal functions (these are incomplete right now, so the actual numbers should be at least twice as large) there are:

  * 336 union return types
  * of which 312 include `false` as a value

This illustrates that the `false` pseudo-type in unions is necessary to express the return type of many existing internal functions.

