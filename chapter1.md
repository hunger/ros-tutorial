# Chapter 1: From apt and colcon to a Pixi Workspace

You know ROS: you install packages with `apt`, resolve dependencies with
`rosdep`, build with `colcon`, and source `setup.bash` files in the right
order. This chapter sets up the same line-follower project with
[pixi](https://pixi.sh) instead -- one tool that handles all of those jobs,
driven by a single manifest file that lives next to your code.

The project in [`code/`](./code) is an ML line follower for TurtleBot3: a
Gazebo Harmonic simulation, a TensorFlow CNN that steers the robot, and the
ament packages that tie them together. Upstream it is set up the classic
way -- a page of `sudo apt install` lines that assumes Ubuntu 24.04, `pip
install` for the ML stack, half a dozen `git clone`s, environment variables
exported in `.bashrc`, and a `colcon build`. By the end of this chapter the
same project installs and runs on a fresh machine with a `git clone` and two
`pixi run`s.

That fresh machine needs exactly one thing: pixi. Not Ubuntu -- any Linux
distribution. No ROS installation, no system Python, no C++ compiler, no
`sudo`. Even the compilers that build the workspace's ament packages come
from the manifest, installed into a user-owned, per-project environment. If
a colleague has pixi, they have the whole lab.

## Install pixi

```shell
curl -fsSL https://pixi.sh/install.sh | sh
```

Nothing else is installed globally. ROS, Gazebo, TensorFlow -- everything
below lands in a per-project environment under `code/.pixi/`, owned by your
user. No `/opt/ros`, no system Python, no sudo.

## Create the workspace

```shell
cd code
pixi init . \
  --channel https://prefix.dev/robostack-jazzy \
  --channel https://prefix.dev/conda-forge
```

This creates `pixi.toml`, the manifest for the workspace. Channels are
package repositories -- the equivalent of an apt source list. The
[robostack-jazzy](https://prefix.dev/channels/robostack-jazzy) channel
contains the complete ROS 2 Jazzy distribution built as conda packages, for
Linux (x86-64 and aarch64), macOS, and Windows; conda-forge supplies
everything else (Python, TensorFlow, OpenCV, ...).

## Add ROS and the ML dependencies

Package names follow the same scheme you know from apt
(`ros-jazzy-<package>`):

```shell
pixi add ros-jazzy-turtlebot3 ros-jazzy-turtlebot3-msgs \
         ros-jazzy-turtlebot3-simulations ros-jazzy-ros-gz \
         ros-jazzy-cv-bridge ros-jazzy-rviz2 \
         ros-jazzy-ros2launch ros-jazzy-ros2run
pixi add tensorflow py-opencv scikit-learn imutils matplotlib-base h5py
pixi add "keras==3.7.0"
```

The exact Keras pin is no accident: the project bundles a pre-trained model
and its CNN node refuses to run if the installed Keras version differs from
the one the model was saved with. Where that requirement would otherwise
live in a README ("make sure you have Keras 3.7.0!"), here the manifest
enforces it on every machine.

Two entries you might not expect: `ros-jazzy-ros2launch` and
`ros-jazzy-ros2run`. RoboStack packages are fine-grained -- each `ros2`
CLI verb is its own package, and nothing else in this list pulls in
`ros2 launch` or `ros2 run` -- declare them explicitly.

Each `pixi add` does three things: records the dependency in `pixi.toml`,
solves the full dependency set, and writes the exact result -- every package,
version, and checksum -- to `pixi.lock`. Commit both files: anyone who clones
the repository gets a bit-for-bit identical environment from the lock file.
This replaces `rosdep install` and `pip install`, and the lock file does what
no `requirements.txt` can: it covers ROS, system libraries, and Python
packages together.

Note what is *not* here: the upstream course clones four forked
`turtlebot3` repositories because Ubuntu's binaries are too old for Gazebo
Harmonic. RoboStack's binaries are current, so the forks are unnecessary.

## Build the ament packages with pixi

The simulation needs one more ROS package that has no binary on any channel:
[`mogi_trajectory_server`](https://github.com/MOGI-ROS/mogi_trajectory_server),
which the launch file starts to display the robot's path in RViz. In the
apt/colcon world you would clone it into your workspace `src/` directory --
here it is the same, minus the overlay:

```shell
mkdir mogi_trajectory_server
curl -L https://github.com/MOGI-ROS/mogi_trajectory_server/tarball/main \
  | tar xz -C mogi_trajectory_server --strip-components=1
```

A second package needs special treatment for a different reason:
`turtlebot3_gazebo` *does* have a binary, but the stock Jazzy version is not
what this course needs -- its burger model carries no camera, and it bridges
`cmd_vel` as `TwistStamped` while the course nodes publish plain `Twist`.
The robot would sit blind and deaf in a perfectly healthy simulation. The
course maintains a fork with a camera-equipped model and a `Twist` bridge,
so we vendor that fork's package as well:

```shell
curl -L https://github.com/MOGI-ROS/turtlebot3_simulations/tarball/new_gazebo \
  | tar xz --wildcards "*/turtlebot3_gazebo/*" --strip-components=1
```

Our packages -- `turtlebot3_mogi`, `turtlebot3_mogi_py`, and the freshly
vendored `mogi_trajectory_server` and `turtlebot3_gazebo` -- are ordinary
ament packages with a `package.xml`. Instead of `colcon build` and overlay
sourcing, pixi builds them as packages of the workspace, using the
[pixi-build-ros](https://prefix-dev.github.io/pixi-build-backends/backends/pixi-build-ros/)
backend. The backend reads `package.xml` -- your existing files stay
untouched.

Building from source is a preview feature, so it is enabled explicitly in
`pixi.toml`:

```toml
[workspace]
preview = ["pixi-build"]
```

Each package gets a small `pixi.toml` next to its `package.xml` that tells
pixi how to build it:

```toml title="turtlebot3_mogi/pixi.toml"
[package.build]
backend = { name = "pixi-build-ros", version = "*", channels = [
  "https://prefix.dev/pixi-build-backends",
  "https://prefix.dev/conda-forge",
] }

[package.build.config]
distro = "jazzy"
```

The `channels` line points at the build backend's development channel.
`pixi-build` is a preview feature and the `pixi-build-ros` backend moves
faster than its conda-forge releases; the version this pixi needs is not on
conda-forge yet. Until it is, every package that this workspace builds takes
its backend from the `pixi-build-backends` channel. (When the backend
releases catch up, drop the `channels` line and the backend resolves from
conda-forge like any other dependency.)

Then the workspace depends on them like on any other package, by path:

```toml title="pixi.toml"
[dependencies]
ros-jazzy-turtlebot3-mogi = { path = "turtlebot3_mogi" }
ros-jazzy-turtlebot3-mogi-py = { path = "turtlebot3_mogi_py" }
ros-jazzy-mogi-trajectory-server = { path = "mogi_trajectory_server" }
ros-jazzy-turtlebot3-gazebo = { path = "turtlebot3_gazebo" }
```

The dependency names carry the `ros-jazzy-` prefix because that is what the
backend names the conda packages it produces -- the same convention RoboStack
uses for the binaries.

Note what the last line does: `ros-jazzy-turtlebot3-simulations` from step
two still pulls in the stock `ros-jazzy-turtlebot3-gazebo` binary -- but a
workspace source package of the same name takes precedence. The fork
replaces the binary for everything that depends on it, the same way a
colcon overlay would shadow an underlay, except resolved by the solver
instead of by environment-sourcing order.

`pixi install` now builds all four packages and puts them into the environment --
name resolution, dependency ordering, and rebuilds on change included. There
is no underlay/overlay distinction and nothing to source.

One wart: building an `ament_python` package currently leaves a stray
`files.txt` (a setuptools record file) in the package source directory.
Until that is fixed in the build backend, add it to `.gitignore`:

```shell
echo "files.txt" >> .gitignore
```

## One platform quirk

On Linux distributions with a recent hardened kernel (e.g. current Fedora),
importing TensorFlow fails with `cannot enable executable stack as shared
object requires: Invalid argument` -- conda-forge's `libtensorflow_cc` marks
its stack executable and the kernel refuses. The manifest can ship the
workaround, an environment variable that every `pixi run` and `pixi shell`
sets automatically:

```toml title="pixi.toml"
[activation.env]
GLIBC_TUNABLES = "glibc.rtld.execstack=2"
```

This is the pixi pattern for all "export this before running" setup that ROS
users normally keep in `.bashrc`: declare it once in the manifest, and
everyone who clones the workspace gets it.

## Run the simulation

The TurtleBot3 launch files select the robot variant through the
`TURTLEBOT3_MODEL` environment variable -- upstream tells you to `export` it
in your `.bashrc`. It goes into the manifest instead, next to the tunable
from the previous section:

```toml title="pixi.toml"
[activation.env]
TURTLEBOT3_MODEL = "burger"
```

Tasks are named commands stored in the manifest -- like a launch alias that
ships with the repository and runs inside the environment:

```shell
pixi task add simulation \
  "ros2 launch turtlebot3_mogi simulation_bringup_line_follow.launch.py"
pixi task add line-follower "ros2 run turtlebot3_mogi_py line_follower_cnn"
```

Start the simulation:

```shell
pixi run simulation
```

Gazebo opens with the TurtleBot3 on a line course, RViz shows the camera
feed. In a second terminal, start the CNN line follower:

```shell
pixi run line-follower
```

A small OpenCV debug window opens showing the downscaled frame the network
sees and its predicted direction -- and the robot follows the line using the
pre-trained network bundled with the package.

For interactive work -- `ros2 topic echo`, `rqt`, introspection -- open a
shell inside the environment instead of sourcing a `setup.bash`:

```shell
pixi shell
ros2 topic list
```

## Recap

The complete setup for a new machine is now:

```shell
git clone <this-repository>
cd <this-repository>/code
pixi run simulation
```

and, once Gazebo is up, in a second terminal:

```shell
pixi run line-follower
```

`pixi run` installs the environment from `pixi.lock` on first use -- there is
no separate setup step to forget. The manifest (`pixi.toml`) declares what
the project needs; the lock file (`pixi.lock`) records exactly what that
resolves to; tasks document how to run it.

So far the manifest assumes one kind of machine. The next chapter declares
*which* machines this workspace supports -- a CUDA workstation for training,
CPU-only laptops for simulation, and the robot itself -- and lets the solver
sort out who gets what.
