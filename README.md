# Dockerized V Rising dedicated server in an Ubuntu 22.04 container with Wine

[![GitHub Actions](https://github.com/AndrewSav/vrising-docker/actions/workflows/main.yml/badge.svg)](https://github.com/AndrewSav/vrising-docker/actions)
[![Docker Image Version (latest semver)](https://img.shields.io/docker/v/andrewsav/vrising?sort=semver)](https://hub.docker.com/r/andrewsav/vrising/tags)

I strongly suggest to start with [official V Rising dedicate server instructions](https://github.com/StunlockStudios/vrising-dedicated-server-instructions), they list and explain the server settings, their different source and precedence, I will assume you are already familiar with these below. The environment variables mentioned there you can directly use with this docker container.

## Environment variables


| Variable    | Description                                                  |
| ----------- | ------------------------------------------------------------ |
| ENABLE_MODS | if provided, mods will be enabled for the server (see below about the mods support) |

## Ports


| Exposed Container port | Type | Default |
| ------------------------ | ------ | --------- |
| 9876                   | UDP  | ✔️    |
| 9877                   | UDP  | ✔️    |

*Note: it has been reported that in order to be listed in the servers list, the ports must be in the steam range **27015-27050**, and the default ports won't work for this purpose*

## Volumes


| Volume             | Container path              | Description                             |
| -------------------- | ----------------------------- | ----------------------------------------- |
| Steam install path | /mnt/vrising/server         | the server files are downloaded into this directory on the first start |
| Saves & settings | /mnt/vrising/persistentdata | server configuration and saves |
| Mods | /mnt/vrising/mods | mods will be copied from this directory to Steam install path on start up, old mods on the Steam install path are removed |

## Server list

1. In order for the server to appear in the server list, it appears that you need to enable both `"ListOnSteam": true,` and `"ListOnEOS": true`  settings, it appears that you need both, even your game is on Steam.
2. The ports should be in 27015-27050 Steam range
3. Your router/firewall should be configured correctly

## Server configuration

When the container starts for the first time it will copy the default server settings to the Saves & settings volume, in the  `Settings` subdirectory. Edit `ServerHostSettings.json` file there if you want to change the ports, descriptions etc., please refer to the very first link in this readme for more details. You will have to restart the container for the changes to be picked up.

## Mods support

> [!NOTE]  
> As of the time of writing (03 May 2025) the mods for V Rising in general (not just for this docker image) are broken by the game 1.1 Oakveil release. The community is working on making nescessary changes to make the modding framework working with the update. You can follow [V Rising Mod Discord](https://discord.com/invite/QG2FmueAG9) for the updates.

When the container starts, first, before starting the server, the container removes old mods from the Steam install path, and then, if `ENABLE_MODS` environment variable is enabled, it copies the mods from the Mods volume. The following files and directories are removed and then copied. Those files and directories are expected to appear in the mods volume if `ENABLE_MODS` is set.

- BepInEx (directory)
- dotnet (directory)
- doorstop_config.ini (file)
- winhttp.dll (file)

This directory structure is based on <https://thunderstore.io/c/v-rising/p/BepInEx/BepInExPack_V_Rising/>, which I tested this setup with.

One suggested workflow for getting the mods running is the following:

- Use [r2modman](https://github.com/ebkr/r2modmanPlus) to select and install those mods, you want on the server locally
- In the Settings => Profile section select "Export profile as file", this will allow you to export an `.rdz` file
- Transfer this file onto your server where you are running this docker container
- Use [r2modman-headless](https://github.com/mpawlowski/r2modman-headless) to install the mods into the mods directory that you mapped to the docker volume

```bash
r2modman-headless --install-dir=./mods \
  --profile-zip ~/vrising-server.r2z \
  --thunderstore-metadata-url=https://thunderstore.io/c/v-rising/api/v1/package/ \
  --work-dir /tmp
```

Here `vrising-server.r2z` is the file you exported on a previous step and `./mods` is the path to the docker volume mapped mods directory.

- **Important:** In `mods/BepInEx/config/BepInEx.cfg`, under `[Logging.Console]` change `Enabled` to `false`
- Set the `ENABLE_MODS` environment variable and start your docker container. If you are using docker compose I suggest running `docker compose up -d --force-recreate` to restart it.
- Once the server is up and running with mods (I noticed it takes considerably more time to start with mods enabled), most of the mods will create configuration files under the Steam install path in `BepInEx/config` directory. You will want to copy all those files over to your mods directory, since they will be lost on then server restart otherwise. Make the desired changes, if any, in those copied configs, and restart the container again

Of course this workflow is just a suggestion, you can use any method of managing mods you want

## Docker CLI

```bash
docker run -d --name='vrising' \
--net='bridge' \
--init \
--restart=unless-stopped \
-e SERVERNAME="My V Rising Server" \
-v '/path/on/host/server':'/mnt/vrising/server' \
-v '/path/on/host/persistentdata':'/mnt/vrising/persistentdata' \
-v '/path/on/host/mods':'/mnt/vrising/mods' \
-p 9876:9876/udp \
-p 9877:9877/udp \
'andrewsav/vrising'
```

## RCON <small>- Optional</small>

To enable RCON edit `ServerHostSettings.json` and paste following lines after `QueryPort`. To communicate using RCON protocol use the [RCON CLI](https://github.com/gorcon/rcon-cli). You will also need to expose the port via docker compose or docker CLI.

```json
"Rcon": {
  "Enabled": true,
  "Password": "change me",
  "Port": 25575
},
```

## Differences with TrueOsiris server

This repository is based of the [TrueOsiris](https://github.com/TrueOsiris/docker-vrising) work. If the original container works well for you, you should stick with it. The reason I forked it, is because I wanted to add mod support, but as I was working on it I was doing more and more changes, that were unlikely to be merged into the original repository: I value simplicity over "more features", so basically I removed quite a bit of things, that some people might find valueable.

The mod support hopefully will be added to TureOsiris image, and at that stage there will be few reasons to use this one over the original one. Changes are:

- Uses winehq for wine, version 9 as for the time of writing. The original one uses version 7, however, there is a winehq label on the original docker repo
- Docker logs and V Rising sever logs are not mixed up interleaved in the docker log. V Rising server logs are in a separate file. Incidentally I also removed old logs clean up on start up, because, IMO if it's done it should be done on schedule and not by the container itself
- I removed all the custom environment variables because they duplicate the ones providing by V Rising server itself and thus are redundant
- I removed "graceful termination",  because I believe that's something that should be handled properly by the docker itself. I added `exec` to invoke wine to make it top level process and added `init` for signals propagation / reaping. I believe this should be sufficient for graceful termination, while being simpler

## Credits

- https://github.com/TrueOsiris/docker-vrising
