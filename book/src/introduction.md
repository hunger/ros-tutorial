# Introduction

You know ROS: you install packages with `apt`, resolve dependencies with
`rosdep`, build with `colcon`, and source `setup.bash` files in the right
order. This book rebuilds a real ROS 2 project with [pixi](https://pixi.sh)
instead — one tool, driven by a single manifest that lives next to your code,
that replaces all of those jobs at once.

The running example is a machine-learning line follower for TurtleBot3: a
Gazebo Harmonic simulation, a TensorFlow CNN that steers the robot, and the
ament packages that tie them together. Over four chapters it goes from a page
of `apt`/`pip`/`git clone`/`colcon` setup to a `git clone` and a couple of
`pixi run`s that work on any machine with pixi installed — no Ubuntu, no
system Python, no C++ compiler, no `sudo`.

## What you will learn

Robotics development is multi-machine by nature: you develop and simulate on a
workstation, train models on a GPU, and deploy to a robot with a different CPU
architecture and an older system libc. Three pixi features make that setup
declarable in a single manifest, and each gets a chapter:

- **Rich platforms** — `workspace.platforms` entries carry the virtual
  packages the solver should assume on each target machine (CUDA on the
  workstation, a glibc floor on the robot), instead of a global
  `[system-requirements]` table.
- **Extras** — conda packages expose optional dependency groups in their own
  metadata, so "training needs scikit-learn" is a fact the package owns rather
  than something every workspace re-encodes.
- **Flags** — a package build records variant markers (`cuda`, `blas:*`) that
  consumers select on, replacing fragile build-string matching.

Chapter 1 starts from the boring baseline — pixi as a drop-in replacement for
apt/rosdep/colcon — so the later chapters have something real to build on.

## The four chapters

1. **From apt and colcon to a Pixi Workspace** — install the whole project,
   ROS and ML stack included, from one manifest; build the ament packages
   without colcon.
2. **One Workspace, Three Machines** — describe a GPU workstation, a CPU
   laptop, and an aarch64 robot as rich platforms, and let the solver give
   each machine what it can run.
3. **Extras: Packages That Know Their Options** — move training-only
   dependencies out of the workspace and into an extra owned by the package.
4. **Flags: Selecting a Build Variant** — record and select a build variant
   with flags instead of build-string archaeology.

## How to follow along

The project lives in the [`code/`](https://github.com/hunger/ros-tutorial/tree/main/code)
directory of the repository. Each chapter changes only the manifest, never the
project code — so you can read straight through, or check out the repository
and apply each step yourself. Every command shown has been run against a real
pixi build.

The line follower is adapted from the
[MOGI-ROS cognitive robotics course](https://github.com/MOGI-ROS/Week-1-8-Cognitive-robotics)
(Apache-2.0).
