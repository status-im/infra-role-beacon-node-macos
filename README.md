# Description

This role provisions a [Nimbus](https://nimbus.status.im/) installation on MacOS hosts.

# Introduction

The role will checkout a branch from the
[nimbus-eth2](https://github.com/status-im/nimbus-eth2) repo, build and run it.

Each host can run multiple beacon nodes. Each node can be built from a different
branch (stable, unstable, testing, etc.) and will be run with systemd.

A timer is installed that will periodically pull changes from git and rebuild
the binaries.

# Ports

The service exposes three ports by default:

* `9000` - LibP2P peering port. Must __ALWAYS__ be public.
* `9200` - JSON RPC port. Must __NEVER__ be public.
* `9900` - Prometheus metrics port. Should not be public.

# Installation

Add to your `requirements.yml` file:
```yaml
- name: infra-role-beacon-node-linux
  src: git+git@github.com:status-im/infra-role-beacon-node-linux.git
  scm: git
```

# Configuration

The crucial settings are:
```yaml
# branch which should be built
beacon_node_repo_branch: 'stable'
# ethereum network to connect to
beacon_node_network: 'mainnet'
# optional setting for debug mode
beacon_node_log_level: 'DEBUG'
# Infura WebSocket URLs
beacon_node_web3_urls: ['wss://mainnet.infura.io/ws/v3/123qwe123qwe123qwe']
```
The order of WebSocket URLs matters. First is the default, the rest are fallbacks.

# Directories

We use the following directory structure in the users home directory:

```
admin@macos-01.ms-eu-dublin.nimbus.prater:/Users/nimbus/beacon-node-prater-stable % ls -l
total 16
-rwxrwxr-x   1 nimbus  staff  2329 Sep  2 11:53 build.sh
drwxr-x---   4 nimbus  staff   128 Sep  2 12:03 data
drwxr--r--  76 nimbus  staff  2432 Sep 29 14:02 logs
drwxr-xr-x  48 nimbus  staff  1536 Sep  2 12:00 repo
-rwxr-x---   1 nimbus  staff   648 Sep  2 11:49 rpc.sh
```

The naming pattern is `beacon-node-{network}-{branch}` e.g.
`beacon-node-mainnet-unstable` for a beacon node build from the `unstable`
branch and running in the `mainnet` network.

The following files and directories are located in the main directory:

- `data/` - Data created by the beacon node process.
- `logs/` - Node and build logs. The logs are rotated according to the `templates/logrotate.conf.j2` file.
- `repo/` - Clone of the [nimbus-eth2](https://github.com/status-im/nimbus-eth2) repo. The branch is set in `beacon_node_repo_branch`.
- `build.sh` - Script that is executed to create a new build. The resulting binaries can be found in `repo/build/`
- `rpc.sh` - Script to make it easier to interact with the beacon node JSON RPC API.

User must be `beacon_node_user` (`nimbus` by default).

# Usage

Starting or stopping a beacon node:
```
cd /Library/LaunchDaemons
sudo launchctl unload status.beacon-node-mainnet-unstable.plist
sudo launchctl load status.beacon-node-mainnet-unstable.plist
```
Note that the launchd config has the `keepalive` value set to `true` and
running `launchctl stop` will immediately restart the node process.

There is no restart command in launchd. To restart the node run `unload` then `load`.

# Logs

Logs are saved in the `logs` directory. There are two log files: `node.log`
which shows the output of the beacon node process and `build.log` which
shows the output of the build script.

To see the beacon node logs:
```
tail -f beacon-node-mainnet-unstable/logs/node.log
```

# Builds

## Automated Builds

Automated builds are done once a day at a time specified with
`beacon_node_build_start_time`. The builds are started from a launchd daemon.
The launchd config file is located in `/Library/LaunchDaemons` and named after the service with a `build-` prefix.

To manually manage the launchd build job use `launchctl`:
```
sudo launchctl start status.build-beacon-node-prater-stable
sudo launchctl stop status.build-beacon-node-prater-stable
```
To stop the timer from running in the future you need to unload the file:
```
cd /Library/LaunchDaemons
sudo launchctl unload status.build-beacon-node-prater-unstable.plist
```

## Manual Builds

For manual builds run the `build.sh` script. It's important that the working
directory is the same as the build script location.

To create a new build:
```
cd beacon-node-mainnet-unstable/
./build.sh
```
Note that the build script will not run if the user is not `nimbus`.
It will also not build the same git commit twice, if this is needed pass the `--force` flag.

To see the logs of previous builds:
```
tail -f logs/build.log
```
When installing a beacon node for the first time, Ansible will run a build as
part of the playbook. If there are any errors during the build the `wait_for`
task will wait until a `timeout` (10min) is reached even though the build
script exited after a few seconds. Make sure to check the build logs and abort
the ansible run when an error occurred to avoid unnecessary waiting time.

# Uninstall

When removing a deployment make sure to stop and delete the launchd daemons:
```
# stop the beacon node
cd /Library/LaunchDaemons
sudo launchctl unload status.beacon-node-mainnet-unstable.plist
rm status.beacon-node-mainnet-unstable.plist

# unload the periodic build job
sudo launchctl unload status.build-beacon-node-mainnet-unstable.plist
rm status.build-beacon-node-mainnet-unstable.plist

# delete the repo, data and logs
rm -rf ~/beacon-node-mainnet-unstable

# delete logrotate config
rm /opt/homebrew/etc/logrotate.d/beacon-node-mainnet-unstable.conf
```

# Known Issues

The first time a repo is checked out and built it fails with:
```
CMake not installed. Aborting.
make[1]: *** [install/usr/lib/libunwind.a] Error 1
make: *** [libbacktrace] Error 2
make: *** Waiting for unfinished jobs....
```
