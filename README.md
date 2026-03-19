# macos-automounter

`macos-automounter` contains `mountctl`, a small macOS helper for:

- forcing the London office DNS server to the front of the resolver list
- mounting reachable London developer machines with SSHFS
- restoring the previous DNS state and unmounting everything cleanly when you leave the office network

The script is intended for a workstation that moves between office and non-office networks. It installs itself as a LaunchAgent and then re-runs periodically so the mount and DNS state follows the machine automatically.

## What `mountctl` does

`mountctl` manages two pieces of state:

1. DNS
   It detects active macOS network services on the configured office subnet prefix, defaulting to `10.20.*`, and moves `10.20.40.200` to the front of the DNS order so `*.office.xyzzy.tools.internal` resolves correctly.
2. SSHFS mounts
   It probes the London developer blades and mounts the reachable ones under `~/mnt`.

When it changes DNS, it first stores the original resolver state in `/tmp/mountctl-dns`. That lets `stop` and `dns-down` restore the previous manual DNS list or automatic DHCP mode.

## Dependencies

`mountctl` expects:

- Homebrew
- Homebrew Bash at `/opt/homebrew/bin/bash`
- macFUSE
- SSHFS from the macFUSE project at `/usr/local/bin/sshfs`

Current install sources:

- Homebrew: `https://brew.sh/`
- Bash: `brew install bash`
- macFUSE: `https://macfuse.io/`
- SSHFS package: `https://macfuse.github.io/2025/11/11/SSHFS-3.7.5.html`

`mountctl install` checks those dependencies before it writes anything to `/usr/local/bin` or `~/Library/LaunchAgents`.

If `mountctl install` detects the deprecated Homebrew `sshfs` at `/opt/homebrew/bin/sshfs`, it will warn but continue. That path is treated as a temporary fallback only; the preferred setup is still the macFUSE SSHFS package at `/usr/local/bin/sshfs`.

## Configuration

`mountctl` supports a small set of environment overrides:

- `HOSTS`: override the managed host list with a space-delimited subset
- `DNSSERVER`: override the office DNS server IP
- `OFFICE_IP_PREFIX`: override the office network IP prefix that marks a service as "in office"
- `REMOTE_PATH`: override the remote path that SSHFS mounts
- `SSHFS_BIN`: override the SSHFS binary path
- `SCRIPT_COLOUR`: set to `off`, `false`, or `0` to disable colour output

Defaults:

- `HOSTS="dev01 dev02 dev03 dev04 dev05 dev06 dev07 dev08"`
- `DNSSERVER=10.20.40.200`
- `OFFICE_IP_PREFIX=10.20.`
- `REMOTE_PATH=""` which mounts the remote login user's home directory
- `SSHFS_BIN=/usr/local/bin/sshfs` when the macFUSE SSHFS package is installed, otherwise `/opt/homebrew/bin/sshfs` if only the deprecated Homebrew SSHFS is present

Install-time overrides are written into the generated LaunchAgent. If you want the background agent to keep using custom `HOSTS`, `DNSSERVER`, `OFFICE_IP_PREFIX`, `REMOTE_PATH`, `SSHFS_BIN`, or `SCRIPT_COLOUR` values, set them when you run `mountctl install`.

## Install

Clone the repo somewhere under your home directory, then run:

```bash
./mountctl install
```

That command:

- installs the script to `/usr/local/bin/mountctl`
- writes `~/Library/LaunchAgents/tools.xyzzy.mountctl.plist`
- bootstraps the LaunchAgent into the current GUI session
- persists any install-time environment overrides into the LaunchAgent

The LaunchAgent is configured with:

- `RunAtLoad = true`
- `StartInterval = 120`
- `KeepAlive -> NetworkState = true`

So after installation it will re-run automatically at login, on network changes, and every two minutes.

## Usage

```bash
mountctl start
mountctl stop
mountctl status
```

Available commands:

- `install`: install or refresh `mountctl` and its LaunchAgent
- `bootstrap`: alias for `install`
- `start`: DNS-up plus mount
- `stop`: DNS-down plus unmount
- `status`: show current DNS settings and managed mounts
- `run`: LaunchAgent mode; auto-apply in the office and auto-clean up elsewhere
- `up`: alias for `start`
- `down`: alias for `stop`
- `dns-up`: only change DNS
- `dns-down`: only restore DNS
- `mount`: only mount reachable hosts
- `unmount`: only unmount managed hosts
- `help`: show command help

## Examples

Use the defaults:

```bash
mountctl start
```

Override the office DNS server IP:

```bash
DNSSERVER=10.20.40.200 mountctl dns-up
```

Override the managed host list:

```bash
HOSTS="dev07 dev08" mountctl mount
```

Override the office network prefix match:

```bash
OFFICE_IP_PREFIX=10.20. mountctl status
```

Override the remote path mounted over SSHFS:

```bash
REMOTE_PATH=/var/tmp mountctl mount
```

Override the SSHFS binary path:

```bash
SSHFS_BIN=/custom/path/sshfs mountctl start
```

Persist a custom host subset and DNS settings into the LaunchAgent:

```bash
HOSTS="dev07 dev08" DNSSERVER=10.20.40.200 OFFICE_IP_PREFIX=10.20. ./mountctl install
```

Combine multiple overrides:

```bash
HOSTS="dev07 dev08" OFFICE_IP_PREFIX=10.20. DNSSERVER=10.20.40.200 REMOTE_PATH=/srv/shared mountctl start
```

## Mount behavior

By default `mountctl` mounts the remote login user's home directory.

You can override that with `REMOTE_PATH`:

```bash
REMOTE_PATH=/some/remote/path mountctl start
```

Mount points are created under:

```text
~/mnt/<short-hostname>
```

For example:

```text
~/mnt/dev07
~/mnt/dev08
```

## DNS behavior

On a London office network, `mountctl` rewrites the active network services whose IPs match `OFFICE_IP_PREFIX`, defaulting to `10.20.*`, so the DNS list begins with:

```text
10.20.40.200
```

That is necessary because the corporate resolver can return `NXDOMAIN` for `office.xyzzy.tools.internal`, and macOS will not reliably fall through to the second DNS server after a negative response.

You can override the default office DNS server IP when needed:

```bash
DNSSERVER=10.20.40.200 mountctl dns-up
```

You can also override the office-network match prefix:

```bash
OFFICE_IP_PREFIX=10.20. DNSSERVER=10.20.40.200 mountctl start
```

When you run `stop`, `dns-down`, or when the LaunchAgent notices the machine is no longer on the office network, `mountctl` restores the previous DNS state from its saved state files.

## Logs

`mountctl` writes logs to:

- `/tmp/mountctl.out`
- `/tmp/mountctl.err`

## License

This project is licensed under the GNU Affero General Public License v3.0 (AGPL-3.0).

- [LICENSE](LICENSE): full license text
- [LICENSE.md](LICENSE.md): short license summary
- [NOTICE.md](NOTICE.md): notices and attributions
