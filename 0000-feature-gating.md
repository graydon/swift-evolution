# Feature Gating

* Proposal: [SE-NNNN](NNNN-feature-gating.md)
* Authors: [Graydon Hoare](https://github.com/graydon)
* Review Manager: TBD
* Status: **Awaiting review**

*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)


## Introduction

This proposal suggests adding a new build-configuration dictionary for
named "features" of a module, and/or of the entire Swift language and
platform. The flags in the `feature` dictionary can be used both for
staging the introduction of new features into the language, as well as for
staging, querying, and configuring optional features provided by 3rd party
modules.

Swift-evolution thread: [Discussion thread topic for that proposal](https://lists.swift.org/pipermail/swift-evolution/)


## Motivation

This proposal is motivated by two different, but related, problems:

1. Currently, developers of a module present all of its features in a
   single API surface. If the module has potentially-optional portions, the
   module developer is presented with a difficult choice: putting portions
   of the module behind custom conditional-compilation flags like (`#if
   FEATURE_FOO`) means risking name-collision with other such flags and
   generally hiding the conditional-nature of the feature from visibility
   to tools; but moving the conditional code to a separate module or
   package may require exposing module-private features to the module's
   public API surface, or negatively effect performance.

2. Currently, users have unconditional access to each feature of the
   compiler (or standard library) as soon as it is enabled in the source
   tree, whether or not the feature is mature or stable. This presents
   compiler-developers working on the feature over the lifecycle of a
   release (or several releases) with a difficult choice: enabling the
   feature publicly to experiment and get feedback on its use means
   potentially introducing long-term compatibility requirements, due to
   users writing code that depends on the immature feature; but isolating
   the feature to a separate branch or conditional build of the compiler
   means drastically limiting the number of users likely to experiment with
   the feature. It would be preferable to introduce language features in a
   way that is disabled by default, but can be opted-into using a more
   lightweight mechanism than rebuilding the whole compiler.


## Proposed solution

The proposed solution is to allow per-module "feature flags" -- indicating
language features, library features, or a combination thereof -- to be
explicitly enabled on the command-line, and for code be marked as
conditional on such feature flags.

Features of the Swift language are controlled by the feature flag
dictionary of the `Swift` standard library module, since many language
features are accompanied by library changes, and a single team controls
and coordinates feature-additions to both in any case.

Language features are further subdivided into stable and unstable, where
the latter are only available for enabling on snapshot builds, not beta or
release builds.


### Example: Conditional features in a library

Suppose one is building a database ORM library, and certain code should be
conditionally enabled when compiling "for postgres" but otherwise
omitted. The ORM can be described as having a feature called `postgres`,
with build-configuration statements guarding the relevant code:

~~~~
class DBConnection {
    // ...
}

#if feature(postgres)
import Postgres
class PGConnection : DBConnection {
    //...
}
#endif
~~~~

This feature can be enabled when compiling the module with:

~~~~
$ swiftc -enable-feature postgres MyORM.swift
~~~~

Moreover, clients can query the feature and enable dependent code:

~~~~
import MyORM
func parseConnectionString(_ s: String) -> MyORM.DBConnection? {
    if s.hasPrefix("sqlite://") {
        // ...
    }
#if feature(MyORM.postgres)
    else if s.hasPrefix("postgresql://") {
        return MyORM.PGConnection(s.substring(from:12))
    }
#endif
    return nil
}
~~~~

### Example: staged use of prototype language features

Suppose a library author sees that the new feature called "conditional
conformances" is slated for arrival in a future version of Swift. Knowing
this, the library author can write some new parts of their library that
depend on the feature:

~~~~
#if feature(Swift.conditionalConformance)
extension Array : Equatable where Element : Equatable {
  func ==(lhs: Array<T>, rhs: Array<T>) -> Bool {
    // ...
  }
}
#endif
~~~~

In snapshot builds of the compiler, users can opt-in to the feature and
experiment with it:

~~~~
$ swiftc -enable-feature Swift.conditionalConformance MyLibrary.swift
~~~~

However, since the feature is still marked as unstable in the compiler, the
opt-in will not work on beta or release compilers; this limits the chance
of production code developing accidental dependence on unstable features:

~~~~
$ swiftc -enable-feature Swift.conditionalConformance MyLibrary.swift
<unknown>:0: error: unstable language feature 'Swift.conditionalConformance' is unavailable in this build
~~~~


## Detailed design

The proposed solution is to add a new `feature` dictionary _for each
module_, along with a build-configuration function `feature(...)` that can
be used to query the feature dictionary of a module in build-configuration
statements. Entries are added to this dictionary by the compiler, both
using a built-in list of language features, and from command-line argument
`-enable-feature`.

### Build-configuration function

The build-configuration grammar is extended with a new feature-testing
function, taking either a single identifier or a module-qualified
identifier:

~~~~
build-configuration --> feature-testing-function
feature-testing-function --> "feature" "(" feature-name ")"
feature-name --> identifier | identifier "." identifier
~~~~

### Command-line option

A command-line option is added called `-enable-feature`, taking an
optionally-module-qualified feature name, such as `-enable-feature foo`
or `-enable-feature Swift.bar`. The only permitted module qualifier names
are the current module and the built-in standard library name `Swift`;
the latter attempts to enable a language feature in the compiler itself.

### Language features

Language features (in module `Swift`) are divided in two groups: stable
features and unstable features. Stable features are enabled in all
compilers (and cannot be disabled). Unstable features are disabled by
default in all compilers, and can _only_ be enabled through the
`-enable-feature` command-line option in snapshot compilers. The ability to
enable unstable features is, itself, a build-time configuration setting on
the compiler, which is disabled on beta or release compilers.

### Compilation artifacts

Feature-flags enabled for a given compilation of a module become part of
the module's compiled, public signature. They are emitted into the serial
form of the module (and possibly the module's metadata), and reloaded when
the compiler imports a module for use in client code.

### Feature name qualification

Feature names queried in build-configuration statements may also be
module-qualified or unqualified. If a feature name is unqualified, it is
looked-up in the current module first, then the system module `Swift`.

### Conditional parsing

If a known feature name is used in a conditional feature-testing-function,
and the feature is disabled, the body of the (disabled) conditional code is
treated differently depending on where the feature is defined.

  - If the feature is defined as part of the standard `Swift` module, the
    body of the disabled conditional code is tokenized and discarded, not
    parsed. This is similar to the way language-version-testing functions
    modify the parser, for forward-compatibility.

  - Otherwise the disabled conditional code is parsed and discarded.

### Unknown feature names

As a further forward-compatibility measure, _unknown_ feature names can be
used in conditional feature-testing-functions. A warning is generated,
and the unknown feature is considered disabled. As with known features,
unknown feature names that are unqualified, or qualified by the `Swift`
module name, are treated as potential language features and the associated
body of disabled conditionally-compiled code is only tokenized, not parsed.


## Impact on existing code

Existing source code should not be affected. Existing modules will be
interpreted as having empty feature-flag dictionaries. Feature flags are
encoded in serialized modules as a new type of block record within the
module's options block.

New code that wishes to use feature-testing-functions will not compile on
older compilers; such code will need to wait until feature-gating, itself,
is stable (or further conditionalize checks on a language-version-testing
function).


## Alternatives considered

### Using existing custom conditional-compilation flags (`-D`)

The existing custom conditional-compilation flags ("CCCF") system, has
several limitations, relative to feature flags:


  - All CCCFs in a project share a single namespace and can collide;
    feature flags are explicitly module-scoped.

  - CCCFs are not obviously public or private, and tools cannot
    differentiate those meant for public manipulation; feature flags are
    explicitly part of the public signature of a module.

  - Since CCCFs are not specific to a module, they are also not stored
    in the compiled artifact and cannot be queried by client code;
    feature flags are stored in the compiled artifact(s) and can be queried
    by client code.

  - Code that is conditional on a CCCF is currently parsed in its entirety,
    with invalid code causing a parse-error; code that is conditional on a
    language feature flag will only be tokenized, not parsed (as with code
    conditional on Swift language-version-testing functions).

### Unqualified lookup alternative

Unqualified feature names could potentially scan all imported modules.
This was decided against because feature flags are resolved early during
parsing, at a time when loading too many additional modules could slow
tools that are only interested in a parse. Moreover, deciding which modules
to scan would, itself, involve running more of the frontend than seems
strictly necessary.

### Integration with the @available system

Availability checking is a runtime facility, and does not seem to quite
address the same set of concerns.
