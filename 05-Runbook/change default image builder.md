
The `forgeops build` command relies internally on a utility script called `skb` (Skaffold Build wrapper) to package your ForgeRock configurations into custom images. By default, Skaffold looks for a running Docker daemon socket (`/var/run/docker.sock`).

To swap this over to use **Podman** seamlessly on your jump server, you have two primary options depending on whether you want a permanent environmental configuration or a one-time command override.

### Option 1: The One-Time Override (Environment Variable)

You can instruct the underlying build engine to use the Podman execution binary instead of Docker on the fly by prefixing your build command with the `BUILD_ENGINE` parameter.

Bash

```
# Execute the forgeops build leveraging Podman directly
BUILD_ENGINE=podman ./bin/forgeops build idm --profile dev
```

### Option 2: The Permanent Method (Recommended for Jump Servers)

If this jump server is primarily using Podman to build your customized ForgeRock AM, DS, IDM, or IG layers, you should configure your shell to always default to Podman.

#### 1. Update your Shell Profile

Add the `BUILD_ENGINE` variable to your profile so it persists across terminal logouts.

Bash

```
# For Bash environments
echo 'export BUILD_ENGINE=podman' >> ~/.bashrc
source ~/.bashrc

# For Zsh environments (if applicable)
echo 'export BUILD_ENGINE=podman' >> ~/.zshrc
source ~/.zshrc
```

#### 2. Enable the Podman System Socket (Crucial for Skaffold compatibility)

Skaffold occasionally relies on a Docker-compatible API socket. Podman provides a built-in service that emulates this behavior perfectly without running a heavy background daemon.

Bash

```
# Start and enable the Podman API service for your current Linux user
systemctl --user enable --now podman.socket

# Verify the socket is active and generating a path
systemctl --user status podman.socket
```

### Deep-Dive Troubleshooting: What to check if it fails

If you trigger `BUILD_ENGINE=podman ./bin/forgeops build` and see execution errors, verify these two platform realities:

- **Skaffold Version Constraints:** Ensure your version of `skaffold` (installed inside your `forgeops/bin` or global path) is at least version **v1.35.0+**. Legacy versions of Skaffold hardcode calls strictly to the `docker` command namespace.
    
- **The Podman Symlink Shortcut:** Certain versions of the ForgeOps build scripts might drop back to a hardcoded `docker` string execution path if they hit an edge-case configuration loop. If this happens, create a user-level alias linking the namespaces:
    
    Bash
    
    ```
    sudo ln -s /usr/bin/podman /usr/local/bin/docker
    ```
    
    _(This tricks the automation script into executing Podman layers cleanly whenever it tries to reference a "docker" command binary)._