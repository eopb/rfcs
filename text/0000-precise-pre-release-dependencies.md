- Feature Name: precise_pre_release_dependencies
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

https://rust-lang.zulipchat.com/#narrow/stream/246057-t-cargo/topic/Semver.20compatible.20pre-release.20versions.20.28maybe.20pre-RFC.29

# Language

root crate: the crate whos cargo lock is currently being used to select dependency versions

# Summary
[summary]: #summary

This RFC proposes extending `cargo update` to allow updates to pre-release versions when requested with `--precise`.
For example a `cargo` user would be able to call `cargo update -p dep --precise 0.1.1-pre0` as long as the version of `dep` requested by their project and its dependenies is semver compatible with `0.1.1`.
This effectively splits the notion of compatibility in `cargo` where a version is considered compatible when `--precise`ly stated but will not be updated to automaticly via a basic `cargo update`.


# Motivation
[motivation]: #motivation

Pre-release crates are currenlty challenging to use in large projects with complex dependacy trees.
For example, if a maintainer releases `dep = "0.1.1-pre0"`.
They may then asks one of their users to try the new API additions in a large project to get feedback on the release before stablising the new parts of the API.
Unfortunately, since `dep = "0.1.0"` is a transitive dependacy of serveral dependencies of the large project, `cargo` refuses the upgrade, stating that `0.1.1-pre0` is incompatible with `0.1.0`.
The user is left with no upgrade path to the pre-release unless they are able to convince all of their transitive uses of `dep` to release pre-releases of there own.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

`cargo-update` does not update dependencies to new pre-release versions.

This is becuase that when Cargo considers `Cargo.toml` requirments for dependacies it always favours selecting stable versions over pre-release versions.
When the specified requirment is itself a pre-release version, that version is considered pinned and Cargo will always select that version.

If a user does want to select a pre-release version they are able to do so by explicity requesting Cargo to update to that version.
This is done by passing the `--precise` flag to Cargo.
Cargo will refuses to select pre-release versions that are "incompatible" with the requirment in the projects `Cargo.toml`.
A pre-release version is considered compatible for a precise upgrade if its major, minor and patch versions, ignoring its pre-release version, is compatible with the requirment.
For example, `x.y.z-pre0` is considered compatible with `a.b.c` if `x.y.z` is semver compatible with `a.b.c` when requested `--precise`ly.

With a `Cargo.toml` with this `[dependencies]` section

```
[dependencies]
example = "1.0.0"
  
```

It is possible to update to `1.2.0-pre0` because `1.2.0` is semver compatible with `1.0.0`

```
> cargo update -p example --precise 1.2.0-pre0
    Updating crates.io index
    Updating example v1.0.0 -> v1.2.0-pre0
```

It is not possible to update to `2.0.0-pre0` because `2.0.0` is semver compatible with `1.0.0`

```
> cargo update -p example --precise 2.0.0-pre0
    Updating crates.io index
error: failed to select a version for the requirement `example = "^1"`
candidate versions found which didn't match: 2.0.0-pre0
location searched: crates.io index
required by package `tmp-oyyzsf v0.1.0 (/home/ethan/.cache/cargo-temp/tmp-OYyZsF)`
```



- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Cargo updating off of a prereeleas happens only if the toml request is for a non prereeleas. Requesting a pre release in the toml will effectively pin cargo update

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

Not easily auditibale as reviewers ofthen dont check the content of cargo lock ( was have a policy at work to avoid pre releases in the production environment but this wouldn't be caught)
 - Maybe this could be reduced by producing a warning when pre-releases are being used

No min versions

App -> pre1 -> pre2
App -> pre2
It's impossible to express that pre1 depends on pre2 via Cargo lock as dependacy cargo locks are ignored
So we shouldn't recommend pretag precise upgrades on libraries 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Embrase that pre-releases arn't very usable and discorage there use.

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

> Not provided good support of pre-releases and instead always encorage users to apply a git ovveride (versons need to be the same, source needs to be vendored or git patched, can't include them on the registry, makes the porpualr pre-releases second class)

# Prior art
[prior-art]: #prior-art

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

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Channels of compatible pre versions `beta.1` `beta.2`

Resolving pre release versions in `Cargo.toml`

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
