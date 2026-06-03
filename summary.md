# Summary: ROS with Pixi

You came in knowing the apt/colcon workflow. This is what the series replaced
it with, in one place. The running example was an ML line follower for
TurtleBot3 (Gazebo Harmonic + a TensorFlow CNN + ament packages), taken from a
page of `apt`/`pip`/`git clone`/`colcon` setup to a `git clone` and one
`pixi run`.

## The one idea

A single manifest (`pixi.toml`) next to your code declares what the project
needs; a lock file (`pixi.lock`) records exactly what that resolved to. One
tool -- pixi -- then does the jobs you used to split across apt, rosdep,
colcon, and `setup.bash` sourcing. Everything lands in a per-project,
user-owned environment under `code/.pixi/`: no `/opt/ros`, no system Python,
no `sudo`.

## What maps to what

| ROS / apt world | Pixi world |
|---|---|
| `apt`, `pip`, `rosdep install` | `pixi add`, resolved into `pixi.lock` |
| apt source list | channels (`robostack-jazzy`, `conda-forge`) |
| `ros-<distro>-<pkg>` apt names | `ros-jazzy-<pkg>` conda names (same scheme) |
| `requirements.txt` (Python only) | `pixi.lock` (ROS + system libs + Python together) |
| `colcon build` + overlay sourcing | `pixi install` builds workspace packages |
| `source install/setup.bash` | `pixi run <task>` / `pixi shell` |
| `export FOO=...` in `.bashrc` | `[activation.env]` in the manifest |
| forked repos for newer binaries | current RoboStack binaries (forks unneeded) |
| `[system-requirements]` (whole workspace) | rich `platforms` entries (per machine) |
| `tensorflow-gpu` fork of the dep list | solver picks the CUDA build where `__cuda` exists |
| "training needs sklearn" in a README | a `training` extra owned by the package |
| `pkg=*=cuda*` build-string matching | `flags = ["cuda"]` variant selection |

## Command cheat-sheet

```shell
pixi init . --channel <ch1> --channel <ch2>   # create pixi.toml
pixi add <pkg> ...                            # add deps, update lock
pixi lock                                     # resolve all platforms
pixi install [-e <env>]                       # materialise an environment
pixi run [-e <env>] <task>                    # run a task (installs on first use)
pixi shell [-e <env>]                         # interactive shell in the env
pixi task add <name> "<command>"              # define a task
pixi workspace platform list                  # list declared platforms
pixi list [-e <env>] [--platform <p>]          # what a machine gets
pixi tree [-e <env>] [--platform <p>] --invert <pkg>  # reverse deps
```

## The mental shift

In the apt/colcon world, knowledge is scattered: versions in a README, exports
in `.bashrc`, CUDA setup in a training doc, per-machine resolution at install
time. With pixi it converges into the manifest and the lock file, lives next to
the code, is reviewed in version control, and resolves the same way on every
machine. The manifest declares intent; the lock file is the reproducible
record; tasks document how to run it.
