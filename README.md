# ROSFlight Distrobox Setup

This repository contains two related setup bundles:

1. **`foo/`** — a generic Ubuntu 24.04 Distrobox wrapper for creating, entering, and resetting a project container.
2. **`rosflight/`** — a ROSFlight add-on that installs ROS 2 Jazzy tooling and builds a ROSFlight workspace inside that Distrobox.

The intended workflow is:

```text
host machine
└── create Ubuntu 24.04 Distrobox with foo/scripts
    └── inside the container, run rosflight/scripts
        └── install ROS 2 Jazzy, import ROSFlight repos, install deps, build workspace
```

## Repository layout

```text
.
├── foo/
│   ├── README.md
│   └── scripts/
│       ├── 00_destroy_distrobox.zsh
│       ├── 01_create_distrobox.zsh
│       ├── 02_enter_distrobox.zsh
│       └── 03_bootstrap_dotfiles.zsh
│
└── rosflight/
    ├── README.md
    ├── rosflight.repos
    ├── scripts/
    │   ├── _common.zsh
    │   ├── 00_install_ros2_jazzy_tools.zsh
    │   ├── 01_import_rosflight_repos.zsh
    │   ├── 02_rosdep_install.zsh
    │   ├── 03_build_workspace.zsh
    │   ├── run_all.zsh
    │   └── source_rosflight_env.zsh
    └── workspace/
```

## What this does

### Generic Distrobox wrapper

The `foo/` scripts create an isolated Ubuntu 24.04 Distrobox container for the project. By default, the container name is the name of the project directory.

The container is created with:

- Ubuntu 24.04 by default
- a project-local Distrobox home under `.distrobox-home/`
- the project directory mounted into the container
- privileged/device access suitable for robotics and embedded development workflows
- optional dotfiles bootstrapping from `git@github.com:Derekbenj/dotfiles.git`

### ROSFlight add-on

The `rosflight/` scripts are designed to be run **inside** the Distrobox. They install ROS 2 Jazzy and related tools, import ROSFlight repositories, install ROS dependencies, and build the workspace with `colcon`.

The ROSFlight workspace imports:

- `rosflight/rosflight_ros_pkgs`
- `rosflight/roscopter`
- `rosflight/rosplane`

## Requirements

On the host machine:

- Linux host with `zsh`
- `distrobox`
- Docker or Podman backend for Distrobox
- network access for package installation and Git repository cloning

Inside the container, the scripts expect:

- Ubuntu 24.04 / Noble
- `sudo` access
- `zsh`

The ROSFlight add-on installs ROS 2 Jazzy, `colcon`, `vcstool`, `rosdep`, ARM embedded tools, OpenOCD, and related dependencies.

## Quick start

### 1. Create the Distrobox from the host

From the repository root on the host:

```zsh
cd foo
./scripts/01_create_distrobox.zsh
```

To enter the container:

```zsh
./scripts/02_enter_distrobox.zsh
```

### 2. Optional: bootstrap dotfiles

From the host, after creating the Distrobox:

```zsh
cd foo
./scripts/03_bootstrap_dotfiles.zsh
```

By default this uses:

```zsh
DOTFILES_REPO=git@github.com:Derekbenj/dotfiles.git
DOTFILES_BRANCH=main
```

You can override those values:

```zsh
DOTFILES_REPO=https://github.com/<user>/<repo>.git \
DOTFILES_BRANCH=main \
./scripts/03_bootstrap_dotfiles.zsh
```

### 3. Install and build ROSFlight inside the container

Once inside the Distrobox, run:

```zsh
cd ~/rosflight
./scripts/run_all.zsh
```

This runs the full ROSFlight setup:

```zsh
./scripts/00_install_ros2_jazzy_tools.zsh
./scripts/01_import_rosflight_repos.zsh
./scripts/02_rosdep_install.zsh
./scripts/03_build_workspace.zsh
```

After the build completes, source the ROSFlight environment:

```zsh
source ~/rosflight/scripts/source_rosflight_env.zsh
```

If your shell setup sources `~/.zshrc.local`, new shells should load the ROSFlight environment automatically after `run_all.zsh` updates that file.

## Step-by-step usage

### Reset the Distrobox

From the host:

```zsh
cd foo
./scripts/00_destroy_distrobox.zsh
```

This removes the Distrobox container and also tries to remove matching Docker/Podman containers with the same name.

### Create the Distrobox

```zsh
cd foo
./scripts/01_create_distrobox.zsh
```

Optional environment variables:

```zsh
PROJECT_NAME=my-container-name
DISTROBOX_IMAGE=ubuntu:24.04
DISTROBOX_HOME=/path/to/container/home
```

Example:

```zsh
PROJECT_NAME=rosflight-dev ./scripts/01_create_distrobox.zsh
```

### Enter the Distrobox

```zsh
cd foo
./scripts/02_enter_distrobox.zsh
```

If `/usr/bin/zsh` exists in the container, the script enters a login `zsh` shell. Otherwise, it falls back to the default Distrobox shell.

### Install ROS 2 Jazzy tooling

Inside the Distrobox:

```zsh
cd ~/rosflight
./scripts/00_install_ros2_jazzy_tools.zsh
```

This script checks that the container is Ubuntu 24.04 / Noble, configures the ROS 2 apt repository, installs ROS 2 Jazzy Desktop and development tools, initializes `rosdep`, and adds a ROSFlight environment hook to `~/.zshrc.local`.

### Import ROSFlight repositories

```zsh
./scripts/01_import_rosflight_repos.zsh
```

This imports the repositories listed in `rosflight.repos` into:

```text
~/rosflight/workspace/src
```

### Install workspace dependencies

```zsh
./scripts/02_rosdep_install.zsh
```

This runs `rosdep install` against the packages in `workspace/src`.

### Build the workspace

```zsh
./scripts/03_build_workspace.zsh
```

This builds the workspace with:

```zsh
colcon build --symlink-install --event-handlers console_direct+
```

## Environment variables

After sourcing `source_rosflight_env.zsh`, these variables are exported:

```zsh
ROS_DISTRO=jazzy
ROS_PYTHON_VERSION=3
ROSFLIGHT_HOME=~/rosflight
ROSFLIGHT_WS=~/rosflight/workspace
```

## Notes

- The generic Distrobox setup defaults to Ubuntu 24.04 because ROS 2 Jazzy targets Ubuntu Noble.
- The ROSFlight scripts are guarded so they must be run inside a container.
- The ROSFlight add-on does not install Rust, Cargo, or `uv`; those are expected to be handled by the generic environment/bootstrap workflow.
- The `workspace/` directory is intentionally present as the build workspace root. Generated build artifacts such as `build/`, `install/`, and `log/` should usually be ignored by Git.

## Suggested `.gitignore`

If this is going into a GitHub repository, consider adding:

```gitignore
# Distrobox local home
foo/.distrobox-home/
.distrobox-home/

# ROS / colcon build outputs
rosflight/workspace/build/
rosflight/workspace/install/
rosflight/workspace/log/

# Python/cache/editor noise
__pycache__/
*.pyc
.cache/
.vscode/
.idea/
.DS_Store
```

## Troubleshooting

### `This must be run inside the Distrobox container.`

The ROSFlight scripts are being run on the host instead of inside the container. Enter the Distrobox first:

```zsh
cd foo
./scripts/02_enter_distrobox.zsh
```

Then run the ROSFlight scripts from inside the container.

### `/opt/ros/jazzy/setup.zsh not found`

ROS 2 Jazzy has not been installed yet. Run:

```zsh
cd ~/rosflight
./scripts/00_install_ros2_jazzy_tools.zsh
```

### `ROS 2 Jazzy packages are expected on Ubuntu 24.04 / noble`

The container image is not Ubuntu 24.04. Recreate the Distrobox with the default image or explicitly set:

```zsh
DISTROBOX_IMAGE=ubuntu:24.04 ./scripts/01_create_distrobox.zsh
```

### `vcs not found`

Install the ROS 2 tooling first:

```zsh
./scripts/00_install_ros2_jazzy_tools.zsh
```

## License

No license file was included in the uploaded files. Add a `LICENSE` file before publishing if you want to make the repository open source under a specific license.
