# Using tailscale client on an immutable desktop OS

This guide provides instructions on how to set up and use the Tailscale client on an immutable desktop operating system, such as KDE Linux. Immutable OSes are designed to be stable and secure by preventing changes to the base system, which can pose challenges when installing and configuring software like Tailscale.

## Prerequisites
- An immutable desktop OS installed (e.g., KDE Linux).
- A Tailscale account. You can sign up at [Tailscale](https://tailscale.com/).
- Basic knowledge of terminal commands and system administration.

I have tested following procedure with :
- KDE Linux 2025-12-28.
- Tailscale 1.92.3
- zsh 5.9
## Overview
Since immutable OSes do not allow direct installation of software, we will use a containerized approach to run the Tailscale client. This guide will walk you through setting up a Distrobox container to run Tailscale.


## Installation
We assume the Distrobox is already installed on your system ( as same as KDE Linux is ).

There are three main steps to get Tailscale running in a Distrobox container:
1. Create a Distrobox container.
2. Install Tailscale in the container.
3. Modify shell config to resolve the Tailscale host name.

### Step 1: Create a Distrobox Container
Fist, create a Distrobox container with root privileges and systemd support.

Then, enter the container.
```sh
distrobox create --root --name tailscale\
    --image quay.io/toolbx-images/debian-toolbox:latest\
    --init --additional-packages "systemd"

distrobox enter --root tailscale
```

### Step 2: Install Tailscale in the Container
Install Tailscale using the official installation script, enable the Tailscale daemon, and export the Tailscale binary to the host system.
```sh
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
sudo tailscale set --operator $USER
distrobox-export --bin /bin/tailscale
```
Now, you can run the Tailscale binary from the host system at `~/.local/bin/tailscale`.

### Step 3: Modify shell config to resolve the Tailscale host name
In the KDE Linux, the Tailscale inside the container works fine, but the DNS settings are not applied to the host system automatically.

To fix this, you can create a shell function that wraps the Tailscale command and applies the necessary DNS settings whenever you run `tailscale up` or `tailscale set`.

Add the following function to your shell configuration file (e.g., `~/.bashrc` or `~/.zshrc`):

> [!IMPORTANT]
> Substitute `<your-tailscale-domain-name>` with your actual Tailscale domain name, which is usually in the format `tailxxxxxx.ts.net`.

```sh
# Run tailscale inside container and add appropriate DNS.
tailscale() {
    # Run the exported binary from distrobox.
    command $HOME/.local/bin/tailscale "$@"
    local exit_status=$?

    # Are "up" or "set" avairable in the command line?
    # And then, if the command terminated normally, do it.
    if [[ ($1 == "up" || $1 == "set") && $exit_status -eq 0 ]]; then
        _apply_tailscale_dns
    fi

    return $exit_status
}

# Add 100.100.100.100 as DNS server to the tailscale0
_apply_tailscale_dns() {
    echo "Checking Tailscale interface and DNS..."
    # Wait for the interface appears.
    for i in {1..5}; do
        if ip link show tailscale0 > /dev/null 2>&1; then
            # If already configured, skip.
            if ! resolvectl dns tailscale0 2>/dev/null | grep -q "100.100.100.100"; then
                sudo resolvectl dns tailscale0 100.100.100.100
                # <your-tailscale-domain-name> is `tailxxxxxx.ts.net` where xxxxxx is user specific.
                sudo resolvectl domain tailscale0 "~ts.net" "~tailscale.net" "<your-tailscale-domain-name>"
                echo "✅ Tailscale DNS settings applied to tailscale0."
            else
                echo "ℹ  Tailscale DNS is already configured."
            fi
            return 0
        fi
        sleep 1
    done
    echo "⚠ tailscale0 not found. DNS settings skipped."
}
```

After adding the function, reload your shell configuration:
```sh
source ~/.zshrc  # or source ~/.bashrc
```

## Usage
You can now use the `tailscale` command as you normally would. For example, to start Tailscale, run:
```sh
tailscale up
```

This will also apply the necessary DNS settings to the Tailscale interface on your host system.

## Additional Notes

### Tailscale API socket
To control the Tailscale from non CLI client ( e.g., KTailctl ), you may need to publish the Tailscale API socket from the container to the host system. You can do this by adding the following option when creating the Distrobox container:
```sh
--volume /var/run/tailscale:/var/run/tailscale:rw
```

So the entire command to create the container will be :
```sh
sudo mkdir -p /var/run/tailscale
distrobox create --root --name tailscale\
    --image quay.io/toolbx-images/debian-toolbox:latest\
    --init --additional-packages "systemd"\
    --volume /var/run/tailscale:/var/run/tailscale:rw
```

The /var/run/tailscale directory must be created on the host system before creating the container. 

Note that the /var/run/ directory is usually cleared on reboot, so you may need to recreate it after a system restart.

To recreate the directory automatically on boot, you can create a dropfile for the tmpfiles service on the host system.

```sh
sudo mkdir -p /etc/tmpfiles.d
echo "d /var/run/tailscale 0755 root root -" | sudo tee /etc/tmpfiles.d/tailscale-socket.conf
```

### DNS configuration commands
The actual commands to setup DNS for the Tailscale TUN device are :
```sh
sudo resolvectl dns tailscale0 100.100.100.100
# <your-tailscale-domain-name> is `tailxxxxxx.ts.net` where xxxxxx is user specific.
sudo resolvectl domain tailscale0 "~ts.net" "~tailscale.net" "<your-tailscale-domain-name>"
```

These commands must be run whenever the Tailscale interface is brought up. If you turn on the Tailscale by non-tailscale CLI command ( e.g., KTailctl ), you need to run these commands manually, or think some trick to run them automatically.

## See also
- [Installing Tailscale on immutable Linux distros - DEV Community](https://dev.to/mscharley/tailscale-on-immutable-linux-3j92)
## Licensing
This guide is provided under [CC0 1.0 Universal (CC0 1.0)](LICENSE) license. 