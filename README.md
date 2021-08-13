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

## Launchd services

The following launchd configs will be registered:

```
# beacon node process
status.beacon-node-mainnet-unstable
# scheduled build
status.beacon-node-mainnet-unstable-build
```

## Beacon Node

Start/Stop (called `load/unload` in launchd):

```
cd /Library/LaunchDaemons

sudo launchctl load status.beacon-node-mainnet-unstable.plist
sudo launchctl unload status.beacon-node-mainnet-unstable.plist
```

Show Logs:

```
tail -f beacon-node-mainnet-unstable/logs/node.log
```

## Builds

Create a new build:

```
# Important to change the working directory
cd beacon-node-mainnet-unstable
./build.sh

# If the commit was build previously we need to pass --force
./build.sh --force
```

Show build logs:

```
tail -f logs/build.log
```
