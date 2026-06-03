# A Pixi Tutorial for ROS Users

This repository is the working ground for a tutorial that introduces
[pixi](https://github.com/prefix-dev/pixi) to ROS users, built around a real
robotics project: one workspace that serves a dev workstation with a CUDA GPU,
a CPU-only laptop (Linux or macOS), and an aarch64 robot.

## How to use this tutorial

The tutorial lives in the git history. Every commit is one step, and applying
the steps in order turns the starting project into the finished one. The commit
messages are numbered `<chapter>.<step>`, so the log reads as a table of
contents:

```
0.0  Introduction          (main)
1.0  Introduction          (chapter1)
1.1  Install pixi
...
1.7  Recap
2.0  Introduction          (chapter2)
...
3.3  Recap
4.0  Introduction          (chapter4)
...
4.4  Recap
END  Summary: ROS with Pixi (summary)
```

`main` is the starting point: the unmodified ROS project before any pixi work.
Each branch marks where a chapter begins, and `summary` marks the end:

| Branch     | Points at                  |
|------------|----------------------------|
| `main`     | the starting project       |
| `chapter1` | start of chapter 1         |
| `chapter2` | start of chapter 2         |
| `chapter3` | start of chapter 3         |
| `chapter4` | start of chapter 4         |
| `summary`  | the finished project       |

There are two ways to follow along. Read it like a book by walking the history
one commit at a time -- each diff shows exactly what that step changes:

```bash
git log --oneline main..summary   # list every step
git show <commit>                 # read what one step does
```

Or jump straight to the state at any point and work from there:

```bash
git checkout chapter2   # the project as it stands at the start of chapter 2
```

If you want to try a step yourself, check out the commit before it and redo the
change described in the next commit's message.

## Motivation

Robotics development is multi-machine by nature: you develop and simulate on a
workstation, train models on a GPU, and deploy to a robot with a different CPU
architecture and an older system libc. Two upcoming pixi features make this
setup declarable in a single manifest, and this tutorial showcases both:

- **Rich platforms** -- `workspace.platforms` entries can carry the virtual
  packages the solver should assume on the target machine, instead of a
  global `[system-requirements]` table:

  ```toml
  [workspace]
  platforms = [
    { platform = "linux-64", cuda = "12.8" },
    { name = "robot", platform = "linux-aarch64", glibc = "2.39" },
    "osx-arm64",  # generic entries last: matched in order, first match wins
  ]
  ```

  Each machine gets exactly the packages it can run, and features can bind to
  a named platform (GPU training only on the CUDA machine, headless inference
  on the robot).

- **V3 package features** -- conda packages can expose optional dependency
  groups (`extras`) and variant selectors (`flags`) in their metadata, so
  picking a CUDA build of TensorFlow or pulling in visualization-only
  dependencies no longer requires build-string matching or parallel package
  names:

  ```toml
  [dependencies.tensorflow]
  version = ">=2.18"
  flags = ["cuda"]
  ```

The tutorial also builds the project's ROS packages with the
[pixi-build-ros](https://prefix-dev.github.io/pixi-build-backends/backends/pixi-build-ros/)
backend, which reads `package.xml` directly -- no `colcon` setup required.

## The demo project

The code in [`./code`](./code) is a ROS 2 (Jazzy) course project that teaches
machine-learning-based line following with a simulated TurtleBot3 robot in
Gazebo Harmonic. It progresses from classic OpenCV color-thresholding to a
small CNN: camera images from the simulated robot are collected and labeled
(`forward`, `left`, `right`, `nothing`), a Keras/TensorFlow model is trained
on them, and the trained network then steers the robot along a line in the
simulation. A pre-trained model and the training images are included.

It consists of two ament packages:

- `turtlebot3_mogi` (`ament_cmake`) -- launch files, Gazebo worlds and models,
  RViz configurations
- `turtlebot3_mogi_py` (`ament_python`) -- the Python nodes:
  `line_follower` (OpenCV), `save_training_images`, `train_network`,
  `line_follower_cnn` (CNN inference), plus training data and the
  pre-trained model

It fits the tutorial because the project naturally targets heterogeneous
machines -- GPU training, CPU-only simulation, aarch64 inference -- and its
dependency variants (TensorFlow CPU vs. CUDA builds, simulation and
visualization stacks only needed on dev machines) map directly onto rich
platforms, extras, and flags.

All ROS dependencies are available from the
[robostack-jazzy](https://prefix.dev/channels/robostack-jazzy) channel and all
Python/ML dependencies from conda-forge, on linux-64, linux-aarch64, and
osx-arm64.

### Origin

- Upstream repository: <https://github.com/MOGI-ROS/Week-1-8-Cognitive-robotics>
- Author: David Dudas, part of the MOGI ROS course at the Budapest University
  of Technology and Economics (BME), Faculty of Mechanical Engineering,
  Department of Mechatronics, Optics and Mechanical Engineering Informatics
- Checked out 2026-06-03 from the `main` branch at commit
  [`7a1e23535e2e8d482ee7fae4c37dbba167ad73bc`](https://github.com/MOGI-ROS/Week-1-8-Cognitive-robotics/commit/7a1e23535e2e8d482ee7fae4c37dbba167ad73bc)
  ("add video about different environments")

### License

Apache License 2.0 -- see [`code/LICENSE`](./code/LICENSE).
