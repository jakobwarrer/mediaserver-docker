# Automated Media Server
---
## About 
This is an automated media server set up in docker containers via docker-compose. This goal of this setup is to automate as much of the installation and configuration as possible.

The end result of this setup is a media server with the following components
- `plex`
- `qBittorrent`
  - with `filebot` binary
  - auto-configured to run `filebot` on torrent completion
- `sonarr`
- `radarr`
- `jackett`

The high-level steps for setup are as follows
1. Install docker and docker-compose
1. `docker-compose up`
1. add an indexer to `jackett`
1. configure `sonarr` / `radarr` to use the `jackett` indexer
1. configure `sonarr` / `radarr` to use `qbittorrent` as their download client
1. add libraries to `plex`

Once these steps have been completed, it is possible to add a TV Show or Movie to `sonarr` / `radarr`, have it automatically download them when available, and then it will be automatically copied into your `plex` libraries. Post-installation, this should be a fully-automated media server (the exception to this is removing completed downloads).

📓 This has only been tested on Ubuntu 18.04 LTS but it should work just fine on other linux distors. MacOS and Windows are unsupported. If you test it on MacOS or Windows and it works, let me know!

## Network
Each service is available on its own ports:
- `qBittorrent` : `8080`
- `sonarr` : `8989`
- `radarr` : `7878`
- `jackett` : `9117`
- `plex` : `32400/web`

The services are all running in `network=host` mode so they can see each other without having to do extra port mapping.

## Installation
#### Install Docker
  - https://docs.docker.com/install/#supported-platforms

#### Install Docker Compose
  - https://docs.docker.com/compose/install/#install-compose

#### Clone this repo
```
git clone https://github.com/ghostserverd/mediaserver-docker.git
cd mediaserver-docker
```

#### Build your `.env` file
```
cp .env_sample .env
id $USER # save the result of this for building your .env file below
```

Modify the `.env` file to specify the following configurations. Note that these directories are for the host machine. They are mapped to various locations inside of their container by the `docker-compose` file.

- `CONFIG_DIR` is where configuration for each of the services will live. You'll end up with multiple directories in here, one for each service. If you move this directory to a different volume with a different instance of the whole media server, it should retain your various configurations.

- `DOWNLOAD_DIR` is where various services will download files to. ⚠️ This should not be on a small partition as it will contain media files.

- `MEDIA_DIR` is where your media will be copied to. ⚠️ This should not be on a small partition as it will contain media files.

- `TV_DIR` is where your TV shows will be placed by `filebot` on download completion. It should be a subdirectory of `MEDIA_DIR`

- `MOVIES_DIR` is where your TV shows will be placed by `filebot` on download completion. It should be a subdirectory of `MEDIA_DIR`

- `QBIT_WEBUI_USER` is the Web UI user for `qBittorrent`. The default is `admin`.

- `QBIT_WEBUI_PASS` is the Web UI password for `qBittorrent`. The default is `adminadmin`.

- `PUID` is the unix `UID` that will be passed to the various services. It can be discovered by running `id $USER` on the host machine as mentioned above.

- `PGID` is the unix `GID` that will be passed to the various services. It can be discovered by running `id $USER` on the host machine as mentioned above.

#### Deploy the service
```
docker-compose up
```
Append `-d` to run in detached mode. The first time you run it, it's probably a good idea to not run in attached mode so you can watch all of the logs for issues.

## Configuration
#### Configure Jackett
`<server-ip>:9117`
- Add an Indexer
  - Click `+ Add Indexer`
  - Search for your desired tracker
  - Click on the 🔧icon next to your desired tracker
  - Sign in with your account information
  - Click `Okay`
- You should see `Successfully configured IPTorrents` and a new entry for your tracker.
- Copy the `API Key` from the top right corner and save it somewhere
- Click the `Copy Torznab Feed` and paste it somewhere to save it

#### Configure Sonarr
`<server-ip>:8989`
- Add an Indexer
  - Click on the `Settings` button at the top
  - Click on the `Indexers` tab
  - Click the big `+` symbol
  - Click the `Custom` button in the `Torznab` section
  - Configure your `Torznab` feed
    - `Name` : the name of this indexer (doesn't matter, just name it the name of your tracker)
    - `URL` : the `Copy Torznab Feed` url from `Jackett` that you saved earlier
      - I would suggest replacing the IP with `localhost`
        - http://localhost:9117/api/v2.0/indexers/<indexer>/results/torznab/ instead of 
        - http://192.168.1.11:9117/api/v2.0/indexers/<indexer>/results/torznab/
    - `API Key` : the `API Key` from `Jackett` that you saved earlier
  - Click `Test` to verify that it is configured properly
  - Click `Save`
  
- Add a Download Client
  - Click on the `Settings` button at the top
  - Click on the `Download Client` tab
  - Under `Completed Download Handling` toggle `Enable` from Yes to No
    - We'll be using `filebot` to handle our completed downloads
  - Click the `Save` button at the top right
  - Click the large `+` button
  - Click on `qBittorrent`
  - Configure your Download Client
    - `Name` : whatever you want; probably `qBittorrent`
    - `Host` : `localhost`
    - `Port` : `8080`
    - `Username` : `admin` or whatever you have set for `QBIT_WEBUI_USER` in your `.env` file
    - `Password` : `adminadmin` or whatever you have set for `QBIT_WEBUI_PASS` in your `.env` file
  - Click `Test` to verify that it is configured properly
  - Click `Save`
  
- Add Some TV Shows
  - Click on the `Series` button at the top
  - Click `+ Add Series`
  - Start typing in the search bar. It will search automatically when you stop typing
  - Configure the download path (you should only have to do this on the first show you add)
    - Click the dropdown that says `Select Path`
    - Click `Add a different path`
    - Click on the 📁button on the right of the modal
    - Click on `data` > `completed` > `tv`
    - Click on `Ok`
    - Click on the green ✔️that is now visible
    - Click on the `+` sign
- There are many other configuration options for `Sonarr` that are not covered here. `Sonarr`'s webpage is [here](https://sonarr.tv/)

#### Configure Radarr
`<server-ip>:7878`

#### Configure qBittorrent
`<server-ip>:8080`
- This is mostly already configured. It automatically has configuration for `filebot` download completion handling, the username / password you set in your `.env`, and sets your download directory to `/downloads/`. If you want other `.env` configurations to be available for `qBittorrent`, open an issue here.

#### Configure Plex
`<server-ip>:32400/web`
- Add some libraries
  - TV Shows will be at `/data/TV Shows` assuming you followed the `/media/TV Shows` convention for `TV_DIR`
  - Movies will be at `/data/Movies` assuming you followed the `/media/Movies` convention for `MOVIES_DIR`
- ⚠️ Set up your media agents to not use local files (hopefully this will be fixed in the future)
  - There is a problem that I have not been able to fix yet where local TV Series art is not available. It appears the artwork downloaded by `filebot` is not readable by `plex` for an unkown reason (I don't believe it's permissions related, but if you have ideas, please open an issue).
  - To get around this, uncheck `Local Media Assets` for all Agents under `Settings` > `Server` > `Agents`. Artwork will be downloaded by plex and accessible.
- Kill and restart the containers after logging in to Plex if you have Plexpass
```
docker-compose down
docker-compose up
```

## Thank You
#### Linuxserver
[linuxserverurl]: https://linuxserver.io
[linuxserverforumurl]: https://forum.linuxserver.io
[ircurl]: https://www.linuxserver.io/irc/
[podcasturl]: https://www.linuxserver.io/podcast/
[linuxserverdonate]: https://www.linuxserver.io/donate/

Most of these containers are config wrappers around [LinuxServer.io][linuxserverurl] containers. Without their amazing linuxserver containers, none of this would have been possible. If you find this automated media server useful, go donate to them!

* [forum.linuxserver.io][linuxserverforumurl]
* [IRC][ircurl] on freenode at `#linuxserver.io`
* [Podcast][podcasturl] covers everything to do with getting the most from your Linux Server plus a focus on all things Docker and containerisation!
* [Donate][linuxserverdonate]

#### Filebot
[fileboturl]: https://www.filebot.net/
[filebotforumurl]: https://www.filebot.net/forums/
[filebotpurchaseurl]: https://www.filebot.net/purchase.html

This would also not be possible without filebot. This is currently using the free linux version `4.7.7`, but this may change to use the paid `4.8.2` version in the future. If it does so, there will be a separate tag to stay on `4.7.7`, but it will likely become unsupported.

* [Filebot][fileboturl]
* [Forum][filebotforumurl]
* [Purchase][filebotpurchaseurl]
