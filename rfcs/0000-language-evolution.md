 * Name: `language_evolution`
 * Date: 2020-02-18
 * Author: Nikita Popov <nikic@php.net>
 * Proposed Version: PHP 8.0
 * RFC PR: [php/php-rfcs#0002](https://github.com/php/php-rfcs/pull/2)

# Introduction

In recent years, there has been an increasing tension in the PHP community on how to handle backwards-incompatible language changes. The PHP programming language has evolved somewhat haphazardly and exhibits many behaviors that are considered undesirable from a contemporary position.

Fixing these issues benefits development by making behavior more consistent, more predictable and less bugprone. On the other hand, every backwards-incompatible change to the PHP language may require adjustments in hundreds of millions of lines of existing code. This delays the migration to new PHP versions.

The general solution to this problem is to allow different libraries and applications to keep up with language changes at their own pace, while remaining interoperable. There are quite a few ways in which this general goal can be achieved, which will be discussed in the following.

# Examples of possible backwards-incompatible changes

To keep the following discussion grounded in real proposals, this section gives a few examples of possible backwards-incompatible changes based on existing RFCs. These are intended as examples only, this proposal does not endorse all of them.

## Strict types

The [scalar type declarations RFC](https://wiki.php.net/rfc/scalar_type_hints_v5) introduced the `strict_types` declare, which allows controlling the behavior of scalar type declarations. If it is enabled, passed types must match the declared type exactly (modulo details). If it is disabled, certain type coercions are permitted.

While this is an already existing feature, it is worth giving it some consideration in this context, because it is an existing example of opting into a backwards-incompatible change. To provide some historical context: At the time, internal functions already accepted scalar types and validated them according to coercive semantics. To maintain consistency with internal type checks, userland types would have to follow the same, often undesirable, semantics. The `strict_types` declare allows keeping the behavior between userland/internal consistent, while still providing an opt-in to strict type checking.

I think that overall, the introduction of scalar types together with a `strict_types` directive worked out fairly well. Apart from the common complaint (at least in the early days) of having to specify this option in every single file, the main technical issue people encountered is around the treatment of callbacks:

Callbacks invoked by internal functions like `array_map()` always have coercive argument semantics, even if invoked from a strictly typed file. Conversely, callbacks invoked in a strictly typed file but coming from a weakly typed one, will use strict argument semantics. This is a shortcoming in the original design of `strict_types`: Callbacks should have been special-cased to use the typing mode at the declaration site, not the call-site. I am mentioning this issue here primarily to illustrate that the idea of making code seamlessly interoperate based on opt-ins may not always work out perfectly in edge-cases.

## Explicit pass-by-reference

The [explicit call-site pass-by-reference](https://wiki.php.net/rfc/explicit_send_by_ref) RFC proposes to allow marking arguments that are passed by reference using a `&` at the call-site, next to the existing marker at the declaration-site. The reasons for why this is desirable are laid out in the motivation section of that proposal.

However, only *allowing* the use of `&` at the call-site does not give us many benefits: The use of a call-site marker has to be *required* in order to reap the full benefits for readability, static analysis and performance.

Unfortunately, requiring the marker results in the worst possible type of backwards-compatibility break: It becomes very hard to write code that is compatible both with PHP 8 (where the call-site marker is required) and PHP 7 (where the call-site marker is forbidden). Such a change could only be rolled out over a very long time frame, by first allowing an optional marker, and only making it required many, many years later. This is the kind of change that benefits most from an opt-in mechanism.

## Forbidding dynamic object properties

One of the motivating cases mentioned in the [namespace-scoped declares](https://wiki.php.net/rfc/namespace_scoped_declares) RFC, and (in a different form) the [locked classes](https://wiki.php.net/rfc/locked-classes) RFC. Most code nowadays expects that all properties are declared in classes (apart from specific exceptions like `stdClass` and of course excluding magic like `__set()`). Setting a property that has not been declared is very likely a typo, not an intentional act. Unfortunately PHP silently allows it, and there is no good way to disable this behavior (a common workaround is to include a trait with throwing magic methods).

Having the option to make undeclared property accesses (modulo mentioned exceptions) Error exceptions would be benefitial for modern code. However, there is also a lot of old code that does not declare properties, so this needs to be opt-in.

## Strict operators and friends

While `strict_types` can be used to disable type coercions for function arguments, there currently is no way to disable the same for basic language constructors, such as arithmetic operators. The [strict operators](https://wiki.php.net/rfc/strict_operators) RFC proposes a `strict_operators` opt-in declare to forbid most type coercions.

This RFC is interesting in that it not only adds new errors, but also changes behavior in some places. For example, the `switch` statement will use strict comparison (modulo details), while normally `switch` uses weak comparison (`==`). This is an important distinction: If a declare only adds errors, you can always produce valid code by assuming the option is enabled (regardless of whether it actually is). If the declare carries a behavior change, then it is truly important whether the option is enabled or not.

## Name resolution changes

A [`function_and_const_lookup='global'`](https://wiki.php.net/rfc/use_global_elements) declare has been proposed and declined recently. While this particular proposal was declined, we may still wish to consider other name resolution changes that bring the rules for functions/constants and classes in line.

## String interpolation changes

The [arbitrary expression interpolation](https://wiki.php.net/rfc/arbitrary_expression_interpolation) RFC proposes to introduce the `"foo #{1 + 1} bar"` syntax for interpolating arbitrary expressions. This poses a minor backwards-compatibility break, because currently `#{}` is allowed inside strings and does not have special meaning. A backwards-compatibility break could be avoided by making the syntax opt-in. It would even be possible to use the `"foo {1 + 1} bar"` syntax, though whether that is a good idea is a different question.

This example is interesting, because it involves a change to the PHP lexer, while all the previous examples involved changes to the compiler or runtime.

# Approaches

Three general approaches (more in terms of philosophy than technical detail) have been discussed in the past and are summarized here.

## New language with common implementation (codename P++)

This approach has been suggested by Zeev, and there's some existing discussion of this in the [P++ FAQ](https://wiki.php.net/pplusplus/faq) and [P++ Concerns](https://wiki.php.net/pplusplus/concerns). A PHP internals [unofficial straw poll](https://wiki.php.net/rfc/p-plus-plus) was unanimously opposed to the idea.

The idea behind P++ is that a new language is introduced, which shares an implementation, and is interoperable with PHP. However, P++ could have major differences in syntax and behavior, and could pursue different design goals. As the name suggests, this is similar to the situation of C and C++, which are usually both supported by the same compiler and are nominally interoperable.

I think there are a number of problems with this approach, and they all essentially come down to P++ being "one big change":

 * There is only one chance to introduce backwards-incompatible changes. Once P++ is released, we are back to square one. Assuming that we will not manage to create the perfect language on the first try, it would be better to introduce a more sustainable mechanism.
 * A one-time major change places a high upgrade burden. Depending on scope, it might be more akin to switching languages than upgrading the language version. This makes it less likely that old code will switch to P++.
 * If the divergence is too large, then it may be hard to ensure interoperability between PHP and P++. For example, while C and C++ are nominally compatible, in practice this requires exporting C++ code through a C-compatible FFI interface. This makes it fairly easy to integrate C in C++, but not the other way around. Hypothetically, if P++ introduced generics but PHP did not, it's not clear how they would interoperate.
 * Frankly, we just don't have the development resources to pull this off. This needs a concerted multi-year effort from a larger development team than we have right now; our resources are best invested elsewhere.

On the positive side, P++ would allow us to make some more radical changes than the approaches discussed below. People sometimes bring up ideas like "drop the `$` from variable names", and we generally just brush this off as the craziness that it is, but P++ would make such changes at least principally feasible.

## Editions

"Editions" are a concept popularized by the Rust programming language. See the [edition guide](https://doc.rust-lang.org/edition-guide/editions/index.html) and the [epoch RFC](https://github.com/rust-lang/rfcs/blob/master/text/2052-epochs.md) for more information.

Editions are specified at the package (in Rust: crate) level, and opt-in to a bundled set of backwards-incompatible changes. Different packages using different editions remain compatible. Editions in Rust are intended to be supported forever, though we may adopt a different [support timeline](#maintenance-burden-and-support-timeline). There are also some limitations on what kind of changes are permitted in editions (one of the significant limitations is that no standard library changes are allowed, something we would likely want to adopt as well).

Editions were also intended to serve as a rallying point from a marketing perspective, though I believe that this coupling between a purely technical mechanism and the marketing angle was found to be confusing and detrimental in hindsight.

## Fine-grained declares

An alternative to the "editions" approach is to introduce more fine-grained declare directives for individual changes. This is inspired by the existing `strict_types` directive, and has been brought up in quite a few of the recent proposals in the preceding examples section.

Some considerations regarding the differences between editions and fine-grained declares:

 * The main advantage of "editions" is that changes are grouped and hierarchical. This reduces the number of language "dialects" that are available at a given time to the number of editions. Fine-grained declares on the other hand create `2^N` different dialects, where N is the number of boolean declares.
 * The main advantage of fine-grained declares is that it allows updating code more gradually (by handling changes one by one), and by allowing to opt out of specific changes in parts of the codebase. For example, if there was a hypothetical `no_dynamic_properties` declare, then one may wish to enable it for most code, but disable it in one particular file where one interacts with a legacy library that requires the use of dynamic properties.
 * Relatedly, fine-grained declares allow handling cases where we want to leave people with a choice as to whether they want a certain change or not. "Editions" carry a strong implication that you *should* be using the new edition and the changes it entails. For example, would the current `strict_types` option be enabled as part of a new edition?

# Technical realization of "per-package" options

Independently of how "fine-grained" the opt-ins are, there are also multiple ways in which they could be specified on a technical level. These will be discussed in the following.

## Status quo: Declares at top of file

We already use `declare(strict_types=1);` at the top of the file to enable strict typing, so it would be natural to continue relying on this mechanism.

This approach has two big advantages: First, it already works, is familiar and does not require the introduction of any new language facilities. Second, the file remains self contained: The used language dialect can be determined without looking at additional files, which is especially helpful for tooling.

It also has some disadvantages: First, it doesn't scale. This approach is only compatible with the "editions" approach, where only a single `declare(edition=2020);` line is needed. It is not feasibly to use this with the "fine-grained declares" approach.

Second, packages will very likely want to use one language dialect for the entire package, not mix and match across different files. While having the declare in every single file is nominally more explicit, in practice the programmer will model this as "I'm working on a PHP 2020 project" in their mind, and will not double-check the edition whenever they open a new file. This may lead to surprises if a file forgets to specify the edition by accident.

## New opening tag

A minor variation on the preceding variant: Instead of using declares, a new opening tag can be introduced. Once again this only works for P++ `<?p++` or editions `<?php2020`. This has essentially the same characteristics as per-file declares. It is slightly more compact, but introduces new syntax.

A suggestion that has been made during the discussion is that a new file extension could be used as well. This is only compatible with P++, otherwise all files in a project would have to be renamed on each edition upgrade. Using a file extension also has other disadvantages: It would not work in cases where the file extension is not known, for example if a script is piped into PHP, read from stdin, or coming from a non-filesystem stream. A new extension increases the chance of code-leaks on servers not aware of the new configuration.

## Namespace-scoped declares

This variant is explored in more detail in the [namespace-scoped declares](https://wiki.php.net/rfc/namespace_scoped_declares) RFC. This proposal allows specifying declares for a whole namespace (including sub-namespaces):

```php
namespace_declare('Vendor\Lib', [
    'strict_types' => 1,
    'no_dynamic_properties' => 1,
    // ...
]);
```

The intention is that these will be specified in `composer.json`, and Composer will take care of registering the declares with PHP.

This approach avoids the disadvantages of specifying declares in each file: It can scale with an arbitrary number of declares, and the programmer can assume that declares hold for the whole project, unless they get explicitly overridden.

One disadvantage of this approach (and all the other "package-based" approaches dicussed in the following) is that you can no longer tell the used language dialect by looking at a single file. While this should not be a problem for humans (for whom a package-oriented approach is more useful), it may be an issue for tooling, as it may not be possible to correctly process files without having a larger context.

Unlike the two package-based approaches discussed in the following, namespace-scoped declares are based on the existing, well-established and well-understood "namespace" feature. This is both an advantage (it does not introduce any new concepts or need for additional code) and a disadvantage:

While namespaces commonly map directly to a package, they don't always do so. For example, the main amphp package uses the `Amp\` namespace, while other amphp packages use `Amp\FooBar\`. Here, the `Amp\` namespace cannot really be treated as a single package. There are additional issues (e.g. due to the ability to have multiple namespaces in a single file), but these can be resolved, see the linked RFC for more detailed discussion.

## Explicit package declaration

There is no proper RFC for this variant, but a prototype [pull request](https://github.com/php/php-src/pull/4490) is available. This introduces a new "package" concept that is orthogonal to namespaces. The package would have to be declared in each file, next to the namespace:

```php
<?php

package "nikic/php-parser";

namespace PhpParser\Node;

// ...
```

Declares could then be bound to a particular package, rather than a particular namespace. Additionally, the same feature could be reused for other package-based features, such as package-private symbols.

The advantage of this approach is that it resolves the ambiguities that exist around using namespaces for this purpose. On the other hand, it introduces a whole new concept to the language, and may caused confusion in how it relates to namespaces. Unless this concept is also used for other purposes, it may not be worthwhile to introduce it.

## Filesystem based packages

An alternative to explicit package declarations are filesystem based packages. A package is defined by placing a special file, say `_package.php`, inside a directory, which can then also contain per-package configuration.

Relative to the previous variant, this has the advantage of not needing an explicit package declaration in every single file. Additionally, unlike both namespace-scoped declares and explicit package declarations, it provides a well-defined place where to look for package-related declares (the `_package.php` file). The previous two variants would place the declares inside `composer.json` by convention, but `namespace_declare()` or `package_declare()` could also be called from other places, which makes tooling support more complicated.

The disadvantage of this approach is the filesystem coupling, which introduces a number of problems in PHP:

* Cache invalidation: Within one request, PHP would presumably cache both which directories do not contain a `_package.php` file, and the contents of the ones which do. Changes to either of those would be ignored during the request. The more problematic case is how these would interact with opcache. If `validate_timestamps` is enabled, we would have to check all the directories (and parent directories) for added, removed or changed `_package.php` files as well, which may carry an additional performance penalty.
* Path canonicalization: In order to determine whether a file is part of a package, canonicalized paths need to be available. For example, symlinks must be resolved, and case-(in)sensitivity of the filesystem must be handled correctly. While this functionality is available on the filesystem layer through `realpath()`, this functionality currently does not exist for general PHP streams (not even phars support this properly). This may be solvable with additional stream wrapper hooks.
* Directory traversal: In order to find package files, we need to go "upwards" in a given path to find package files. Once again this operation is not supported by stream wrappers. In fact, many stream wrappers do not have a meaningful concept of a "directory".
* Generally, it may be hard to make this feature work with arbitrary streams. I have a package that replaces the `file` stream wrapper, in order to intercept all included files and replace them with preprocessed files stored in a temporary directory. I have no idea just how this would interact with a directory-based package system.

# General considerations

## Maintenance burden and support timeline 

Each new edition/declare adds additional maintenance burden, because it requires supporting two different behaviors in the same implementation. In Rust the concept of editions includes a commitment to support them indefinitely (though in practice, backwards-compatibility breaks do sometimes get backported to earlier editions, just later).

We may or may not want to support old editions indefinitely. Even if an edition only has a finite life-time, it is still a useful tool for managing version migrations, because it eliminates dependencies between projects. Each project can update to a new edition independently, without requiring updates to its dependencies first, or forcing an upgrade on its reverse-dependencies.

A related question is how often new editions are released. With the fine-grained declare approach it would probably be fine to add them in any minor version. With editions, it's unclear if we would want to also create a new edition for each minor version, even if there are few changes. Should these only be created for each major version instead? Or should they be created on demand, as we accumulate relevant changes?

## Scope and eligible changes

It should be emaphasized that even if we introduce a mechanism like "editions", that does not imply that all backwards-incompatible changes will be handled through editions, and it also doesn't imply that any kind of backwards-compatibilty break is fine as long as it's based on editions.

There are a few considerations in play here:

 * Technical feasibility: Some changes cannot be made through editions/declares, because the effect of the change cannot be contained to code using the edition. For example, removing name mangling from HTTP query parameters will affect the contents of `$_GET` in all code and cannot be limited to one edition. This change has to happen globally or not at all.
 * Mental burden: While it should always be fine to introduce new error conditions in new editions, we need to be more careful about changes in behavior. If only errors are introduced, the programmer can always program against the newest edition, and the code will work even if an older one is actually used. I don't think that changes in behavior should be outright forbidden, but there should be a higher bar for them.
 * Maintenance overhead: As mentioned before, every change introduced through the edition/declare mechanism comes with additional maintenance burden, because two different behaviors have to be supported (and their interaction with other features). As such, the edition/declare mechanism should not be used for minor backwards-compatibility breaks. The mechanism should be seen as a means of making changes we could not make before: If a change can feasibly go through a normal deprecation+removal cycle instead, it should.
 * Library changes: As a particular case, I believe that the edition/declare mechanism is not a good fit for standard library changes. Those are better handled through new functions, or flags.

## Automatic upgrades

Rust provides mostly automated tooling for performing edition upgrades. Due to its dynamic nature, reliable automated upgrades are not always possible in PHP. However, we may still want to provide official best-effort tooling for this purpose.

# Conclusion

My personal conclusion on where we should move: Editions with per-file declares.

While having fine-grained declares has its advantages, and is the option I advocated for initially, I think that concerns about the proliferation of declares, and exponential explosion in language variants are very much real. Editions, even if new ones are introduced regularly, significantly reduce the number of language dialects, thus reducing mental burden for developers and maintenance burden on our side.

Once we go with editions, there is not a lot of benefit to investing into a new mechanism for package-scoped declares, all of which have their own issues. The per-file declare does have its own advantages, in particular that it keeps things self-contained, and that it allows partial upgrades of a codebase. The ergonomics of per-file declares are of course worse, but as people would essentially be replacing `declare(strict_types=1)` with `declare(edition=2020)`, it's not worse than the current situation. (Furthermore some kind of package-scoped declare mechanism can still be introduced at a later time.)

I think we need to prioritize the introduction of an "edition" in PHP 8 as soon as possible, so we still have time to make use of the opportunity.
