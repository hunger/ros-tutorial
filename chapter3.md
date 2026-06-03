# Chapter 3: Extras -- Packages That Know Their Options

Chapter 2 kept the robot lean by sorting dependencies into workspace
features: simulation here, training there. That works, but some of the
knowledge lives in the wrong place. That scikit-learn, imutils, and
matplotlib are only needed for *training* is a fact about
`turtlebot3_mogi_py` -- yet it is the workspace that encodes it, and every
other workspace that ever consumes the package would have to rediscover and
re-encode it.

Conda packages are growing the fix: optional dependency groups -- *extras* --
declared in the package's own metadata. Python developers know the pattern
from wheels (`pip install requests[security]`); conda extras work the same
way, except a group can carry anything from the conda ecosystem: Python
libraries, C++ libraries, CUDA builds, other ROS packages.

In this chapter the training-only dependencies move out of the workspace
and into a `training` extra owned by the package.

## Teach the package its options

Only `train_network` imports scikit-learn, imutils, and matplotlib; the
inference nodes need none of them. Declare a `training` group in the
package manifest, next to the build section from chapter 1:

```toml title="turtlebot3_mogi_py/pixi.toml"
[package.extra-dependencies.training]
scikit-learn = ">=1.9.0,<2"
imutils = ">=0.5.4,<0.6"
matplotlib-base = ">=3.10.9,<4"
```

```shell
pixi lock
```

The group is now part of the package's metadata: any workspace that depends
on `ros-jazzy-turtlebot3-mogi-py` can request `training` -- and nobody pays
for the group by default.

The `training` group resolves through the same development build backend
chapter 1 already wired up for every package, so extras work without any
extra setup here.

## Request the extra where it is needed

The consumer side is one line: the `training` feature -- already bound to
the `workstation` platform in chapter 2 -- requests the group on its
dependency:

```toml title="pixi.toml"
[feature.training.dependencies]
ros-jazzy-turtlebot3-mogi-py = { path = "turtlebot3_mogi_py", extras = ["training"] }
```

With the package owning the list, the three libraries leave the workspace
dependencies:

```toml title="pixi.toml"
[dependencies]
# scikit-learn, imutils, and matplotlib-base deleted -- the package
# declares them in its `training` extra now.
```

Re-solve and check who gets what:

```shell
pixi lock
pixi list -e training --platform workstation | grep -E "scikit-learn|imutils|matplotlib"
pixi list -e robot --platform robot | grep -cE "scikit-learn|imutils|matplotlib"
```

The training environment carries all three (next to its CUDA TensorFlow);
the robot reports `0`. Same machines as chapter 2 -- but the reason the
training machine gets training libraries is now written in the package that
needs them.

## Recap

- A package declares optional dependency groups in
  `[package.extra-dependencies]`; consumers opt in with
  `extras = [...]` on the dependency -- the conda equivalent of
  `requests[security]`.
- The workspace shrank: it now says *which machines train*, while the
  package says *what training requires*. Each piece of knowledge lives with
  its owner.
- Extras are one half of the new package metadata. The other half is
  variant *flags* -- a package publishing CPU and CUDA builds as flagged
  variants of one name, selected with `flags = ["cuda"]` instead of
  build-string matching. As channels start publishing flagged packages,
  the implicit CUDA choice from chapter 2 becomes an explicit, reviewable
  line in the manifest.
