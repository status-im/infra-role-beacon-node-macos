# infra-role-beacon-node-macos

## Directory structure

We use the following directory structure in the users home directory:

```
.
`-- beacon-node-mainnet-unstable
    |-- build.sh
    |-- data
    |-- logs
    |-- repo
    `-- rpc.sh
```

The naming pattern is `beacon-node-{network}-{branch}` e.g.
`beacon-node-mainnet-unstable` for a beacon node build from the `unstable`
branch and running in the `mainnet` network.

The following files and directories are located in the main directory:

- `build.sh`: script that is executed to create a new build. The resulting
  binaries can be found in `repo/build/`
- `data/`: data created by the beacon node process
- `logs/`: node and build logs. The logs are rotated according to the
  `templates/logrotate.conf.j2` file
- `repo/`: clone of the [nimbus-eth2](https://github.com/status-im/nimbus-eth2) repo. The branch is set in `beacon_node_repo_branch`.
- `rpc.sh`: script to make it easier to interact with the beacon node json rpc api

User must be `beacon_node_user` (`nimbus` by default).

## Beacon Node

Starting or stopping a beacon node (called `load/unload` in launchd):

```
cd /Library/LaunchDaemons
sudo launchctl unload status.beacon-node-mainnet-unstable.plist
sudo launchctl load status.beacon-node-mainnet-unstable.plist
```

There is no restart command in launchd. To restart the node run `unload` then `load`.

## Logs

Logs are saved in the `logs` directory. There are two log files: `node.log`
which shows the output of the beacon node process and `build.log` which shows
the output of the build script.

To see the beacon node logs:

```
tail -f beacon-node-mainnet-unstable/logs/node.log
```

## Builds

### Automated Builds

Automated builds are done once a day at a time specified with
`beacon_node_build_start_time`. The builds are started from a launchd Daemon.
The launchd config file is located in `/Library/LaunchDaemons` and named after
the network and branch followed by a `-build` suffix e.g.
`status.beacon-node-prater-unstable-build.plist`.

To stop the timer from running in the future you need to unload the file:

```
cd /Library/LaunchDaemons
sudo launchctl unload status.beacon-node-prater-unstable-build.plist
```

### Manual Builds

For manual builds run the `build.sh` script. It's important that the working
directory is the same as the build script location.

To create a new build:

```
cd beacon-node-mainnet-unstable/
./build.sh
```

Note that the build script will not run if the user is not `nimbus`. It will
also not build the same git commit twice, if this is needed pass the `--force`
flag.

To see the logs of previous builds:

```
tail -f logs/build.log
```

When installing a beacon node for the first time, Ansible will run a build as
part of the playbook. If there are any errors during the build the `wait_for`
task will wait until a `timeout` (10min) is reached even though the build script
exited after a few seconds. Make sure to check the build logs and abort the
ansible run when an error occurred to avoid unnecessary waiting time.

## Uninstall

When removing a deployment make sure to stop and delete the launchd daemons:

```
# stop the beacon node
cd /Library/LaunchDaemons
sudo launchctl unload status.beacon-node-mainnet-unstable.plist
rm status.beacon-node-mainnet-unstable.plist

# unload the periodic build job
sudo launchctl unload status.beacon-node-mainnet-unstable-build.plist
rm status.beacon-node-mainnet-unstable-build.plist

# delete the repo, data and logs
rm -rf ~/beacon-node-mainnet-unstable
```

