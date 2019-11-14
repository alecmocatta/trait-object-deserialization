- Feature Name: `trait_object_deserialization`
- Start Date: 2019-11-14
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Lay the groundwork for safe and sound serialization and deserialization of trait objects on at least tier 1 platforms in a low-impact manner that allows user crates to safely experiment and iterate on this functionality.

# Motivation
[motivation]: #motivation

The ability to serialize and deserialize trait objects is a longstanding issue that has seen recent progressions<sup>1</sup> but remains unsolved in the general case.

The use cases are:

* IPC – for example sending trait objects between forks of a process; and
* distributed computing – for example sending trait objects to processes distributed across a cluster.

While some message kinds can be wrapped in an enum instead of a trait object, this can be inconvenient, and isn't viable for un-nameable types like closures.

See also ["Encodable trait objects"](https://github.com/rust-lang/rfcs/issues/668).

## Serializable closures

Trait object deserialization also unblocks the de/serialization of closures. Multiple distributed computing efforts<sup>2</sup> have benefitted from de/serializable closures, though so far doing this has been unsound and critically unsafe. 

See also ["Serializable Lambda expression"](https://github.com/rust-lang/rfcs/issues/1022).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The approach taken in a [work-in-progress PR](https://github.com/rust-lang/rust/pull/66113) is to create essentially a global array with [appending linkage](https://llvm.org/docs/LangRef.html#linkage-appending) that stores structs comprising the [`type_id`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/struct.TyCtxt.html#method.type_id_hash) of the trait object and a pointer to the vtable, for all materialized vtables. Unfortunately appending linkage isn't really implemented by LLVM yet besides for a couple of special variables, so I've used the old-school approach of emitting static variables into an [orphan section](https://sourceware.org/binutils/docs/ld/Orphan-Sections.html). The start and end of this section can be reliably retreived by user programs with help from the linker.

This array increases the size of libraries by a small amount; for example build/x86_64-apple-darwin/stage2 by ~1.3MiB i.e. ~0.2%. Thanks to `--gc-sections` on Linux and `/OPT:REF` on Windows, it's removed entirely from binaries that don't use it on those platforms. macOS's `-dead_strip` works a little differently, and the best solution I've found adds 16 bytes per used vtable to the resulting binary. This increases the size of binaries by a small amount: hello world grows by 76 bytes i.e. ~0.03%. With a bit more work I think this could probably be made zero-cost similar to Linux and Windows. Android and iOS behave the same as Linux and macOS, and on other platforms this PR is a no-op.

This array allows programs to retrieve a list of all materializable vtable pointers for a given `dyn Trait`. From this, a candidate concrete type for serialization can be selected and safely invoked. Here's a rough sketch of how serialization and deserialization can work:

```rust
use std::raw::TraitObject;
fn vtables() { /* see src/test/ui/vtables/vtable-list.rs */ }

// Process A
let send: Box<T> = send;
let send: Box<dyn Trait> = send;
let type_tag = send.type_tag();
type_tag.serialize();
trait_object.serialize();

// Process B
let trait_type_id = type_id::<dyn Trait>();
let type_tag = deserialize();
let trait_object = vtables().iter().find_map(|record| {
    if record.type_id != trait_type_id {
        return None;
    }
    let trait_object = TraitObject { data: ptr::dangling(), vtable: record.vtable };
    let trait_object = transmute::<TraitObject, *const dyn Trait>();
    (trait_object.type_tag() == type_tag).then(trait_object)
}).unwrap();
let send: Box<dyn Trait> = trait_object.deserialize_erased();
```

`fn type_tag()` here could be `type_id()`, which would work for unnameable types like closures but doesn't work across different builds where the `type_id` has changed due to potentially unrelated changes. It could be provided explicitly by the program (i.e. like [`typetag`](https://github.com/dtolnay/typetag)), which would not work for unnameable types but would work across different builds. It could also be a combination of both – explicit tags where provided, falling back to `type_id`.

<!--
Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.
-->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
-->

# Drawbacks
[drawbacks]: #drawbacks

<!--
Why should we *not* do this?
-->

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!--
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
-->

# Prior art
[prior-art]: #prior-art

## Rust

[`typetag`](https://github.com/dtolnay/typetag).

Deserializing trait objects is possible today, but only in a rather hairy manner. The crate [`serde_traitobject`](https://github.com/alecmocatta/serde_traitobject) for example works and sees usage, but it has two unsoundness vectors:

* a malicious MITM (who could forge ostensibly-valid but unsound data); and
* multiple vtable segments in a single binary that are loaded at inconsistent relative addresses.

The latter requires a very odd linker script and is unlikely to occur in practise; the former is concerning and rules out many use cases. It is however sufficient for other use cases – in fact a really cool one using it was published recently: [`native_spark`](https://github.com/rajasekarv/native_spark).

## JIT compilation / interpreted languages

* Python: pickle
* Java: https://docs.oracle.com/en/java/javase/13/docs/api/java.base/java/lang/invoke/SerializedLambda.html
* Scala: http://erikerlandson.github.io/blog/2015/03/31/hygienic-closures-for-scala-function-serialization/

## AOT compilation languages

* C++: [HPX](http://stellar-group.org/libraries/hpx/) general purpose C++ runtime system for parallel and distributed applications: Serializes functions by registering them in a global map. https://stellar-group.github.io/hpx/docs/sphinx/latest/html/manual/writing_distributed_hpx_applications.html#applying-actions

<!--
Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.
-->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!--
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
-->

# Future possibilities
[future-possibilities]: #future-possibilities

<!--
Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
-->

---

1. [`typetag`](https://github.com/dtolnay/typetag) and [`serde_traitobject`](https://github.com/alecmocatta/serde_traitobject).
2. For example, [`constellation`](https://github.com/constellation-rs/constellation) and [`native_spark`](https://github.com/rajasekarv/native_spark) use [`serde_closure`](https://github.com/alecmocatta/serde_closure) - a proc macro that extracts captured variables from the AST and puts them in a struct that implements Serialize, Deserialize, Debug and other convenient traits – to safely serialize and deserialize closures. As the (unnameable) type isn't known for deserialization, the type is coerced to a trait object and de/serialized with [`serde_traitobject`](https://github.com/alecmocatta/serde_traitobject).
