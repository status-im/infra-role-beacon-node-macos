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

The naming pattern is `beacon-node-{network}-{branch}`.

- build.sh: The script that is executed to create a new build. The resulting
  binaries are located in repo/build/.
- data: the data created by the beacon node process
- logs: node and build logs
- repo: a clone of the nimbus-eth2 repository
- rpc.sh: script to make it easies to interact with the node json rpc api

User must be `beacon_node_user` (`nimbus` by default).

## Beacon Node

Starting or stopping a note (called `load/unload` in launchd):

```
cd /Library/LaunchDaemons
sudo launchctl unload status.beacon-node-mainnet-unstable.plist
sudo launchctl load status.beacon-node-mainnet-unstable.plist
```

There is no restart command. To restart the node run unload then load.

## Logs

Logs are saved in the `logs` directory. There are two log files: `node.log`
shows the stdout/stderr of the beacon node process. `build.log` shows the output
of the build script.

To see the beacon node logs:

```
tail -f beacon-node-mainnet-unstable/logs/node.log
```

## Builds

Run the `build.sh` script to start a new build. It's important that the working
directory is the same as the build script. 

To create a new build run:

```
cd beacon-node-mainnet-unstable/
./build.sh
```

Note that the build script will not run if the user is not `nimbus`. It will
also not build the same git commit twice, if this is needed pass the `--force`
flag.

Usually builds are started from a launchd service that runs once a day.

To see the logs of previous builds:

```
tail -f logs/build.log
```
