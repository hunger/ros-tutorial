# Chapter 2: One Workspace, Three Machines

Chapter 1 ended with a manifest that says `platforms = ["linux-64"]` -- one
kind of machine. That is not what a robotics lab looks like. A realistic
setup for this project is:

- a **workstation** with an NVIDIA GPU, where the CNN gets trained
- a **laptop** without a GPU, where you develop and run the simulation
- the **robot** itself -- an aarch64 board running a stable Debian, headless,
  doing nothing but inference

In the apt/colcon world these are three different setup documents: CUDA and
cuDNN versions in the training README, a desktop install for the laptop, and
a cross-compilation or on-device build story for the robot. In this chapter
the same information moves into `workspace.platforms`: each entry describes
*what a machine has*, and the solver works out what each machine *gets*.

## Name your machines

A platform entry like `"linux-64"` is shorthand for "this conda subdir, with
no extra promises on features but the minimums pixi defaults to". Written as
a table, an entry can pin those capabilities explicitly -- and carry a name:

```toml title="pixi.toml"
[workspace]
platforms = [
  { name = "workstation", platform = "linux-64", cuda = "12.8" },
  { name = "robot", platform = "linux-aarch64", glibc = "2.36" },
  "linux-64",
]
```

**Order matters.** Pixi matches a machine against the entries in the order
they are written and takes the first one whose promises the machine can keep.
A bare `"linux-64"` promises nothing, so *every* linux-64 host satisfies it --
including the CUDA workstation. If it came first it would swallow the
workstation before the `cuda` entry was ever considered. Generic entries go
last; specific ones first.

The keys map onto the conda *virtual packages* you may know from
`[system-requirements]` or from conda itself: `cuda` becomes `__cuda`,
`glibc` becomes `__glibc`, and `macos`, `windows`, `linux`, and
`archspec` work the same way. Where `[system-requirements]` declared one set
of assumptions for the whole workspace, a rich platform declares them per
machine -- the workstation promises CUDA 12.8, the robot promises nothing
but an aarch64 CPU and glibc 2.36, and the plain `linux-64` entry at the
bottom keeps working for any other laptop.

!!! warning "TODO(pixi#6355): derive these platforms, don't hand-write them"

    This chapter hand-writes the virtual packages on each platform. That is
    error-prone -- you have to know which capabilities matter. Once
    [pixi#6355](https://github.com/prefix-dev/pixi/issues/6355) lands, the
    intended workflow is to *derive* them:

    1. `pixi workspace platform add workstation=auto-detected` -- let pixi
       detect the current machine's virtual packages and record them as a
       named platform.
    2. `pixi install` an environment on that machine.
    3. Inspect the *minimum* platform pixi derives for that environment.
    4. Notice the auto-detected platform is far too specific: it pins every
       virtual package the machine happens to have, not just the ones the
       environment actually needs.
    5. Trim it back to that minimal set -- which is exactly the
       `cuda = "12.8"` shown above.

    **This section must be rewritten once #6355 is released.** It is the only
    place in the tutorial that still hand-writes platform capabilities.

Inspect what is declared:

```shell
pixi workspace platform list
```

The `name` is how the rest of the workspace refers to the entry. If you omit
it, pixi synthesises one from the contents (the workstation entry would be
called `linux-64-cuda-12-8`) -- naming them yourself reads better in
everything that follows.

## Lock once, install anywhere

`pixi lock` resolves *all* declared platforms into the one `pixi.lock` --
including machines you are nowhere near. The robot's dependency set is
solved and pinned from your laptop:

```shell
pixi lock
```

On any of the three machines, `pixi install` picks the matching entry and
installs exactly what the lock file says for it. The install itself stays
per-machine -- the workspace's own ament packages are built natively on each
host -- but you can inspect any machine's slice from wherever you sit:

```shell
pixi list --platform robot
```

This is the rosdep/apt story inverted: instead of every machine resolving
its own dependencies at install time (and drifting apart), resolution
happens once, is reviewed in version control, and every machine merely
*materialises* its slice of it.

## Train where the GPU is

The training script `train_network` has been part of `turtlebot3_mogi_py`
all along. Give it a task, and bind that task to the machine that can run
it -- features tie dependencies and tasks to platforms:

```toml title="pixi.toml"
[feature.training]
platforms = ["workstation"]

[feature.training.tasks]
train = "ros2 run turtlebot3_mogi_py train_network"

[environments]
training = ["training"]
```

```shell
pixi run -e training train
```

Because the `workstation` platform declares `cuda = "12.8"`, the solver is
allowed to pick the CUDA build of TensorFlow for that platform -- the same
`tensorflow` dependency from chapter 1 resolves to the GPU variant on the
workstation and stays CPU-only everywhere else. No second manifest, no
`tensorflow-gpu` fork of the dependency list, no build-string matching.

On the laptop, `pixi run -e training train` refuses: the `training`
environment is not available for plain `linux-64`. The error is the
solver's way of saying what the lab whiteboard used to: *don't train on
your laptop*.

## Keep the robot lean

The robot needs `line_follower_cnn_robot` and its inputs -- not Gazebo, not
RViz. Move the simulation stack into a feature that only exists on the
machines with a screen:

```toml title="pixi.toml"
[feature.simulation]
platforms = ["linux-64", "workstation"]

[feature.simulation.dependencies]
ros-jazzy-turtlebot3-gazebo = { path = "turtlebot3_gazebo" }
ros-jazzy-mogi-trajectory-server = { path = "mogi_trajectory_server" }
ros-jazzy-turtlebot3 = ">=2.3.6,<3"
ros-jazzy-turtlebot3-simulations = ">=2.3.7,<3"
ros-jazzy-ros-gz = ">=1.0.19,<2"
ros-jazzy-rviz2 = ">=14.1.19,<15"

[feature.simulation.tasks]
simulation = "ros2 launch turtlebot3_mogi simulation_bringup_line_follow.launch.py"

[environments]
default = ["simulation"]
robot = []
```

The surprise in that list is `ros-jazzy-turtlebot3`: a metapackage that
looks robot-essential but drags in `turtlebot3-bringup` (which wants RViz)
and `turtlebot3-navigation2` (which wants the whole Nav2-and-Gazebo stack).
When a package shows up in an environment and you do not know why, ask for
the reverse dependency tree:

```shell
pixi tree -e robot --platform robot --invert ros-jazzy-rviz2
```

This is `rosdep` without the guesswork: the tree prints exactly which
declared dependency pulls the offender in, and moving that one line into
the `simulation` feature fixes it.

Check the result from your laptop -- the robot's environment contains no
renderer, no simulator, nothing that wants a display:

```shell
pixi list -e robot --platform robot
```

On the robot itself, install and run the inference node:

```shell
pixi install -e robot
pixi run -e robot ros2 run turtlebot3_mogi_py line_follower_cnn_robot
```

## Recap

The manifest now answers a question chapter 1 could not even ask: *which
machine is this?*

- `workspace.platforms` describes each machine's capabilities -- CUDA on the
  workstation, a glibc floor on the robot -- in the same place that used to
  hold a bare list of subdirs.
- One `pixi lock` pins every machine's environment; each machine installs
  its own slice.
- Features bound to named platforms put the training task where the GPU is
  and keep Gazebo off the robot.

One thing still works by *implication*: the workstation gets CUDA TensorFlow
because the solver happens to prefer GPU builds when `__cuda` is present.
The next chapter makes that choice explicit -- conda packages are growing
`extras` and variant `flags` in their metadata, so the manifest can say
`flags = ["cuda"]` and mean it.
