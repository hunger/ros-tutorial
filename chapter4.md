# Chapter 4: Flags — Selecting a Build Variant

Chapter 3 split *optional* dependencies into extras. Flags are the other half
of the new package metadata, and they answer a different question: when one
package name ships in several builds — a CPU build and a CUDA build, an
OpenBLAS build and an MKL build — which one do you want?

You already solved a version of this in chapter 1, the old way:

```shell
pixi add "pytorch=*=cuda*"   # match the build string
```

Build-string matching is fragile: the pattern (`cuda*`) is a per-feedstock
convention, it is easy to get subtly wrong, and it says *how* a build is
named rather than *what* it is. Flags replace the convention with metadata. A
build records a set of plain-string flags, and a consumer selects on them:

```toml
# the build records what it is
[package.build]
flags = ["cuda"]

# the consumer asks for it
some-package = { version = "*", flags = ["cuda"] }
```

> **Note**: The headline example for flags is TensorFlow — its conda-forge
> builds come in CPU and CUDA variants (`cpu_py312…`, `cuda128py312…`). Those
> builds do not carry flag *metadata* yet; they are still distinguished by
> build string. Until the channel publishes flagged builds,
> `tensorflow = { flags = ["cuda"] }` finds no candidates. So this chapter
> demonstrates the mechanism on the one package whose builds you control: the
> ROS package you build yourself.

## Record a flag on your package

`turtlebot3_mogi_py` is built by the workspace (chapter 1), so its metadata is
yours to write. Record a `cuda` flag on the build, next to the backend and
distro settings:

```toml title="turtlebot3_mogi_py/pixi.toml"
[package.build]
backend = { name = "pixi-build-ros", version = "*", channels = [
  "https://prefix.dev/pixi-build-backends",
  "https://prefix.dev/conda-forge",
] }
flags = ["cuda"]
```

```shell
pixi lock
```

The flag is now part of the built package's metadata. It does not change what
the package *contains* — it is a label the solver can match against, the same
way it matches a version or a build string, but declared rather than parsed.

## Select it from the consumer

The training feature from chapters 2 and 3 already runs only on the
`workstation` platform — the machine with the GPU. Have it ask for the
cuda-flagged build explicitly:

```toml title="pixi.toml"
[feature.training.dependencies]
ros-jazzy-turtlebot3-mogi-py = { path = "turtlebot3_mogi_py", extras = ["training"], flags = ["cuda"] }
```

```shell
pixi lock
```

It resolves: the build provides `cuda`, the consumer asked for `cuda`. The
selection is now stated in the manifest instead of being implied by the
platform's `__cuda` virtual package — the "by accident" of chapter 2 becomes
"by design".

The match is enforced, not cosmetic. Ask for a flag the build does not
declare and the solve fails fast:

```shell
# turtlebot3_mogi_py = { path = "...", flags = ["nonexistent"] }
pixi lock
# × No candidates were found for ros-jazzy-turtlebot3-mogi-py *[flags=[nonexistent]]
```

## Where this is going

For a pure-Python ROS package the flag is mostly a marker: there is one build,
and the flag documents and enforces intent. Flags earn their keep when a
single package name ships *several* builds and you need to pick one without
build-string archaeology — exactly the TensorFlow CPU/CUDA case the opening
note set aside.

When conda-forge publishes those builds with flag metadata, nothing in your
workspace changes except the spec: the implicit CUDA selection chapter 2 got
from the `workstation` platform becomes a reviewable line —

```toml
tensorflow = { version = ">=2.19.1,<3", flags = ["cuda"] }
```

— in the same `[feature.training.dependencies]` table, sitting beside the
package flag you set above.

## Recap

The series introduced three selectors, each answering a different question
and each living with its rightful owner:

| Selector | Question | Declared by |
|---|---|---|
| rich `platforms` | what does this *machine* have? | the workspace |
| `extras` | what *optional features* does the package offer? | the package |
| `flags` | which *build variant* do I want? | the build records, the consumer selects |

Together they let one manifest describe a GPU workstation, a CPU laptop, and
an aarch64 robot — each getting exactly the packages, features, and variants
it should — with every choice written down and resolved the same way on every
machine.
