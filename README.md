# zvm

This script helps you manage multiple versions of Zig on your system.

## Installation

Download the zvm script and make it executable:

```
curl -o zvm https://raw.githubusercontent.com/highercomve/zvm/refs/heads/master/zvm
chmod +x zvm
```

For Linux systems, move the script to `/usr/local/bin`:

```
sudo mv zvm /usr/local/bin/zvm
```

For macOS systems, you can also move the script to `/usr/local/bin`:

```
sudo mv zvm /usr/local/bin/zvm
```

Alternatively, on macOS, you can place the script in `/opt/local/bin` if you are using MacPorts:

```
sudo mv zvm /opt/local/bin/zvm
```

Or in `/usr/local/bin` if you are using Homebrew:

```
sudo mv zvm /usr/local/bin/zvm
```

Add the zvm path to your system's PATH environment variable to ensure that the installed Zig versions can be easily accessed from the command line. This step is crucial because it allows your terminal to locate the Zig binaries managed by zvm, enabling you to use the `zig` command without specifying the full path to the executable. By adding the zvm path to your PATH, you streamline your workflow and ensure that the correct version of Zig is used consistently across your projects.

For `.bashrc`:
```
export PATH="$PATH:$HOME/.zvm/current"
```

For `.zshrc`:
```
export PATH="$PATH:$HOME/.zvm/current"
```

For `.config/fish/config.fish`:
```
set -x PATH $PATH $HOME/.zvm/current
```

## Usage

The zvm script provides several subcommands to manage Zig versions. To see the available subcommands, run:

```
zvm
```

### zvm list

The list subcommand lists all the installed Zig versions.

```
zvm list
```

Example output:
```
Installed versions:
  0.11.0
  0.12.0
* 0.13.0
```

### zvm remove

The remove subcommand removes an installed Zig version.

```
zvm remove <version>
```

### zvm install
The install subcommand installs a new Zig version. If no version and architecture are used as arguments, the script will list the available versions of Zig and let you select which version and architecture you want to install.

```
zvm install [<version>] [<architecture>]
```

If you only pass the version, zvm will install it for your current architecture. This is useful if you know the version you want but don't want to specify the architecture every time.

Example:
```
zvm install 0.13.0
```

Example:
```
zvm install
```

Example output
```
Fetching Zig versions...
Available versions:
1) master    3) 0.12.0	 5) 0.10.1   7) 0.9.1	 9) 0.8.1   11) 0.7.1	13) 0.6.0   15) 0.4.0	17) 0.2.0
2) 0.13.0    4) 0.11.0	 6) 0.10.0   8) 0.9.0	10) 0.8.0   12) 0.7.0	14) 0.5.0   16) 0.3.0	18) 0.1.1
#? 2
Selected version: 0.13.0
Available architectures for version 0.13.0:
1) x86_64-freebsd       4) x86_64-linux	       7) riscv64-linux	     10) x86_64-windows
2) x86_64-macos	        5) aarch64-linux       8) powerpc64le-linux  11) aarch64-windows
3) aarch64-macos        6) armv7a-linux	       9) x86-linux	     12) x86-windows
#? 4
Selected architecture: x86_64-linux
Downloading x86_64-linux from https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 44.9M  100 44.9M    0     0  10.4M      0  0:00:04  0:00:04 --:--:-- 10.4M
Download complete.
Verifying signature for zig.tar.gz...
Signature verification successful.
Extracting zig.tar.gz...
Extraction complete.
Installed version 0.13.0.
```

### zvm use

The use subcommand sets the default Zig version to use.

```
zvm use <version>
```

### zvm version

The versions subcommand lists all the available Zig versions.

```
zvm versions
```

```
Available versions:
  0.9.0
  0.10.0
  0.11.0
```

#### Configuration

The zvm script uses the $ZVM_DIR environment variable to specify the directory where Zig versions are stored. By default, it is set to $HOME/.zvm. You can change this by modifying the script or setting the environment variable before running the script

add
