- Feature Name: precise_pre_release_cargo_update
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)

# Summary
[summary]: #summary

This RFC proposes extending `cargo update` to allow updates to pre-release versions when requested with `--precise`.
For example, a `cargo` user would be able to call `cargo update -p dep --precise 0.1.1-pre0` as long as the version of `dep` requested by their project and its dependencies are semver compatible with `0.1.1`.
This effectively splits the notion of compatibility in `cargo`.
A pre-release version may considered compatible when `--precise`ly stated but will not be updated to automatically via a basic `cargo update`.

# Motivation
[motivation]: #motivation

Pre-release crates are currently challenging to use in large projects with complex dependency trees.
For example, if a maintainer releases `dep = "0.1.1-pre0"`.
They may ask one of their users to try the new API additions in a large project so that the user can give feedback on the release before the maintainer stabilises the new parts of the API.
Unfortunately, since `dep = "0.1.0"` is a transitive dependency of several dependencies of the large project, `cargo` refuses the upgrade, stating that `0.1.1-pre0` is incompatible with `0.1.0`.
The user is left with no upgrade path to the pre-release unless they are able to convince all of their transitive uses of `dep` to release pre-releases of their own.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When Cargo considers `Cargo.toml` requirements for dependencies it always favours selecting stable versions over pre-release versions.
When the specification is itself a pre-release version, Cargo will always select a pre-release.
Cargo is unable to resolve a project with a `Cargo.toml` specification for a pre-release version if any of its dependencies specify a stable release.

If a user does want to select a pre-release version they are able to do so by explicitly requesting Cargo to update to that version.
This is done by passing the `--precise` flag to Cargo.
Cargo will refuse to select pre-release versions that are "incompatible" with the requirement in the projects `Cargo.toml`.
A pre-release version is considered compatible for a precise upgrade if its major, minor and patch versions, ignoring its pre-release version, are compatible with the requirement.
`x.y.z-pre0` is considered compatible with `a.b.c` when requested `--precise`ly if `x.y.z` is semver compatible with `a.b.c` and `a.b.c` `!=` `x.y.z`.

Consider a `Cargo.toml` with this `[dependencies]` section

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

It is not possible to update to `2.0.0-pre0` because `2.0.0` is not semver compatible with `1.0.0`

```
> cargo update -p example --precise 2.0.0-pre0
    Updating crates.io index
error: failed to select a version for the requirement `example = "^1"`
candidate versions found which didn't match: 2.0.0-pre0
location searched: crates.io index
required by package `tmp-oyyzsf v0.1.0 (/home/ethan/.cache/cargo-temp/tmp-OYyZsF)`
```

Once `1.2.0` is released `cargo update` will warn the user that a stable version is available

```
> cargo update
warning: version `1.2.0` of `example` has been released but `1.2.0-pre0` was previously selected
note: to use the stable version run `cargo update -p example --precise 1.2.0`
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Consider this table where `a.b.c` is compatible with `x.y.z` and `x.y.z > a.b.c`

| Cargo.toml spec | Cargo.lock version | Target version | Selected by cargo update  | Selected by cargo update --precise  |
| --------------- | ------------------ | -------------- | ------------------------- | ----------------------------------- |
| `a.b.c`         | `a.b.c`            | `x.y.z`        | ✅                        | ✅                                  |
| `a.b.c`         | `a.b.c`            | `x.y.z-pre0`   | ❌                        | ✅                                  |
| `a.b.c`         | `x.y.z-pre0`       | `x.y.z-pre1`   | ❌                        | ✅                                  |
| `a.b.c-pre0`    | `a.b.c-pre0`       | `a.b.c-pre1`   | ✅¹                       | ✅                                  |
| `a.b.c`         | `a.b.c`            | `a.b.c-pre0`   | ❌                        | ❌                                  |
| `a.b.c-pre0`    | `a.b.c-pre0`       | `x.y.z`        | ❌²                       | ✅                                  |

✅: Will upgrade

❌: Will not upgrade

¹For backwards compatibility with Cargo's current behaviour

²Emits a warning

# Drawbacks
[drawbacks]: #drawbacks

- Pre-release versions are not easily auditable when they are only specified in the lock file.
  A change that makes use of a pre-release version may not be noticed during code review as reviewers don't always check for changes in the lock file.
- Library crates that require a pre-release version are not well supported (see [future-possibilities])

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main alternative to this would be to accept that pre-release versions are not very usable and discourage their use.
Cargo overrides can be used instead using `[patch]`.
These provide a similar experience to pre-releases, however, they require that the library's code is somehow vendored outside of the registry, usually with git.
This can cause issues particularly in CI where jobs may have permission to fetch from a private registry but not from private git repositories.
Resolving issues around not being able to fetch pre-releases from the registry usually wastes a significant amount of time.

Another alternative would be to resolve pre-release versions in `Cargo.toml`s even when a dependency specifies a stable version.
This is explored in [future-possibilities].
This would require significant changes to the resolver since the latest compatible version would depend on the versions required by other parts of the dependency tree.
For that reason, I consider detailing such a change outside of the scope of this RFC.

# Prior art
[prior-art]: #prior-art

[RFC: Precise Pre-release Deps](https://github.com/rust-lang/rfcs/pull/3263) aims to solve a similar but subtly different issue where `cargo update` opts to upgrade 
pre-release versions to new pre-releases when one is released.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

It would be nice if dependencies could specify their requirements for pre-release versions.
Since a library crates `Cargo.lock` is ignored when used as a dependancy, these versions must be part of the `Cargo.toml`.
These versions could be required to be specified with `=` and considered incompatible with other pre-release versions.

