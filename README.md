# Description

This role provisions a [Nimbus](https://nimbus.team/) installation on MacOS hosts.

# Introduction

The role will:

* Checkout a branch from the [nimbus-eth2](https://github.com/status-im/nimbus-eth2) repo
* Build it using the [`build.sh`](./templates/build.sh.j2) Bash script
* Schedule regular builds using [Launchd Scheduled Jobs](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/ScheduledJobs.html)
* Start a node by defining a [Launchd Job](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)

# Ports

The service exposes three ports by default:

* `9000` - LibP2P peering port. Must __ALWAYS__ be public.
* `9200` - JSON RPC port. Must __NEVER__ be public.
* `9900` - Prometheus metrics port. Should not be public.

# Installation

Add to your `requirements.yml` file:
```yaml
- name: infra-role-beacon-node-macos
  src: git@github.com:status-im/infra-role-beacon-node-macos.git
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
# WebSocket or HTTP URLs for execution layet Engine API
beacon_node_exec_layer_urls:
  - 'wss://erigon.internal.example.org:8551'
  - 'http://geth.internal.example.org:8552'
```
The order of WebSocket URLs matters. First is the default, the rest are fallbacks.

Most non-sensitive configuration resides in `conf/config.toml` file in service directory.

# Management

## Directories

Each service folder consists of the following contents:

* `build.sh` - Script that is executed to create a new build. The resulting binaries can be found in `repo/build/`
* `rpc.sh` - Script to make it easier to interact with the beacon node JSON RPC API.
* `data/` - Data created by the beacon node process.
* `repo/` - Clone of the [nimbus-eth2](https://github.com/status-im/nimbus-eth2) repo.
* `logs/` - Node and build logs.
* `logs/build.log` - Build logs generated by the Launchd scheduled job.
* `logs/service.log` - Node logs rotated according to the [`logrotate.conf`](./templates/launchd-logrotate.plist.j2) file.

## Service

To see service status use:
```
 > sudo launchctl list status.beacon-node-mainnet-stable
{
	"StandardOutPath" = "/Users/nimbus/beacon-node-mainnet-stable/logs/service.log";
	"LimitLoadToSessionType" = "System";
	"StandardErrorPath" = "/Users/nimbus/beacon-node-mainnet-stable/logs/service.log";
	"Label" = "status.beacon-node-mainnet-stable";
	"OnDemand" = false;
	"LastExitStatus" = 0;
	"PID" = 6347;
	"Program" = "/Users/nimbus/beacon-node-mainnet-stable/repo/build/nimbus_beacon_node";
	"ProgramArguments" = (
		"/Users/nimbus/beacon-node-mainnet-stable/repo/build/nimbus_beacon_node";
		"--network=mainnet";
...
```
To restart the node use:
```sh
sudo launchctl unload /Library/LaunchDaemons/status.beacon-node-mainnet-unstable.plist
sudo launchctl load /Library/LaunchDaemons/status.beacon-node-mainnet-unstable.plist
```
Note that the Launchd config has the `keepalive` value set to `true` and
running `launchctl stop` will immediately restart the node process.

## Builds

Build scheduled with a [Launchd Scheduled Job](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/ScheduledJobs.html) can be also managed using `launchctl`:
```
sudo launchctl start status.build-beacon-node-mainnet-stable
sudo launchctl stop status.build-beacon-node-mainnet-stable
```
The build logs can be seen in the `logs/build.log`.
The job simply runs the [`buils.sh`](./templates/build.sh.j2) Bash script which can be run by hand:
```
 > sudo -u nimbus /Users/nimbus/beacon-node-prater-stable/build.sh 
 >>> Build Start: 2021-10-26T10:37:52Z
 >>> Fetching changes...
HEAD is now at 9e8081e4 Merge branch 'stable' into unstable
 >>> Binary already built. Run with --force flag to override
 >>> SUCCESS
```

## Cleanup

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
