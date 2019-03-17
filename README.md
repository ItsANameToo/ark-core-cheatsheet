# ARK Core cheatsheet

> A curated list with useful commands, links and answers to common issues / questions for delegates and relay node operators

## Basics

### Updating Server

Install and enable the latest security patches and updates:

```bash
sudo apt-get update -y
sudo apt-get upgrade -y
sudo reboot
```

### Fresh ARK Core 2.2 Install

For a fresh installation of Core 2.2, you can read the instructions in the [official ARK docs](https://docs.ark.io/guidebook/core/cli.html#fresh-installation). In short, it comes down to running the following commands:

```bash
adduser ark
usermod -aG sudo ark
su ark
cd ~
bash <(curl -s https://raw.githubusercontent.com/ArkEcosystem/core/master/install.sh)
```

### SSH Access

Depending on your server provider, you can often configure an SSH key in their web portal. That is the most convenient way, as your SSH key will then be available on any server you create with them. If you don't have this option, or want to set it up manually anyway, read on.

Make your life easier by whitelisting your ssh key so you can login without a password

1. Copy the ssh key from your local machine to your clipboard, this differs per OS.
2. On your server, go into the `~/.ssh` folder (create it if it doesn't exist)
3. `nano authorized_keys` (this will create the file if it didn't exist yet)
4. Paste the key you copied in step 1. in this file
5. Exit the editor by pressing `ctrl + x` followed by `y` to save your changes
6. **Important**: keep your current commandline session open and try connecting to your server in a new screen. You should be able to login by simply typing `ssh <username>@<server ip>` and it should no longer prompt for a password.

### Useful aliases

You can add aliases to your `~/.bashrc` file for commands you often run. In order to do this, simply run `nano ~/.bashrc` and add an alias. Afterwards, save and exit the file and enable the changes by running `source ~/.bashrc`, or by logging out and logging in to your server.

```bash
alias pm2rl="pm2 restart all && pm2 logs" # Restart the core processes and open the logs afterward
alias editpl="nano ~/.config/ark-core/mainnet/plugins.js" # Open the `plugins.js` file where you can find and add plugins
alias editdel="nano ~/.config/ark-core/mainnet/delegates.js" # Open the `delegates.js` file where you can find your delegate passphrase
alias cdsnaps="cd ~/.local/share/ark-core/mainnet/snapshots/" # Navigates to the snapshot folder
```

Note that the above location commands use `mainnet` by default. If you are on `devnet` or `testnet` instead, you need to replace `mainnet` with the network you use.

----

## Security

It's recommended that you setup additional security measurements for your node. Some options are explained in the [official ark docs](https://docs.ark.io/tutorials/node/secure.html), which include:

* Changing SSH port and other config tweaks
* Installing Fail2ban
* Port Knocking
* Cloudflare DDOS protection

This is a good basis to have on your server, but you are free to look for other ways to secure it too.

## Node Configuration

### Plugins / Scripts

Some useful core plugins / scripts that you can install on your server:

* [Vanir](https://github.com/ItsANameToo/vanir) - an easy way to self-forge your delegate payout transactions
* [Hermod](https://github.com/ItsANameToo/hermod) - a monitoring and automatic snapshot tool to keep track of your server and to easily keep snapshots
* [Core Control](https://github.com/geopsllc/core-control) - an alternative to the official ARK Cli, with additional features

There are also a couple options for TBW scripts:

* [Cryptology's TBW](https://github.com/wownmedia/cryptology_tbw)
* [Ploutos](https://github.com/faustbrian/ploutos)
* [Goose's TBW](https://github.com/galperins4/tbw)

### .env File

You can find a `.env` file in `$HOME/.config/ark-core/$network/.env` in which you can make changes regarding the default core plugins.
Alternatively, you can run `ark env:list` to show what is currently set in the `.env` file.

The most interesting environment options will be highlighted here:

* `CORE_LOG_LEVEL=<level>`, you can change this to your preferred log level (e.g. `debug`, `info`, etc.). This can also be achieved through the CLI by running `ark env:set CORE_LOG_LEVEL <level>`
* `CORE_DB_USERNAME=<username>` and `CORE_DB_PASSWORD`, these values are used by core to log in to your local database. If you are having issues with accessing the database, you can change those entries as they default to user `ark` and password `password`, which might not be the actual values used on your server.

----

## Snapshots

It is recommended to keep a list of snapshots in case of issues with your node. As snapshots are no longer provided by the ARK team, it's your own responsibility to create and maintain them. You can easily keep a list of snapshots by installing [Hermod](https://github.com/ItsANameToo/hermod), or check below how you can create them manually.

### Creating a Snapshot

A snapshot can be created with `ark snapshot:dump`. The full list of options can be found in the [official ARK snapshot docs](https://docs.ark.io/guidebook/core/cli.html#usage-26). In most cases, it will suffice to use the following command:

```bash
ark snapshot:dump --network="mainnet"
```

### Restoring from a Snapshot

A snapshot can be restored with `ark snapshot:restore`. The full list of options can be found in the [official ARK snapshot docs](https://docs.ark.io/guidebook/core/cli.html#usage-27). In most cases, it will suffice to use the following command:

```bash
ark snapshot:restore --network="mainnet"
```

### Rollback

In case your node gets stuck, it can in some cases suffice to rollback a couple blocks before the issue surfaced and let it sync with the network from there. This can be done with the following command:

```bash
ark snapshot:rollback --network=NETWORK --height=HEIGHT
```

Here, `HEIGHT` is a number indicating the blockheight to which you want to roll back. If you want to go back to height `7.700.000` on `mainnet`, the command will look like this:

```bash
ark snapshot:rollback --network="mainnet" --height=7700000
```

### Database Dump

You can also make a database dump instead of a snapshot. The following command will dump your `ark_mainnet` database to a file called `snapshot_latest`:

```
pg_dump -Fc ark_mainnet > snapshot_latest
```

To import the database dump, you need to run the following commands:

```bash
dropdb ark_mainnet # or ark_devnet
createdb ark_mainnet # or ark_devnet
wget <snapshot url> # fetch the snapshot, not needed if you already have it somewhere locally
pg_restore -n public -O -j 8 -d ark_mainnet <snapshot> # where <snapshot> points to the location + name of the snapshot file
```

### Sharing a Snapshot

In case you need to share a snapshot between server (e.g. your forger was formatted and you want to get a snapshot from your relay), there is an easy method involving [Http Server](https://www.npmjs.com/package/http-server).

1. Install [Http Server](https://www.npmjs.com/package/http-server) on the server you have your snapshots on: `npm install http-server -g`
2. You can now serve any folder by running `http-server [path] [options]`. In case of snapshots, you can for example do the following:

```bash
cdsnaps # Alias, otherwise use cd ~/.local/share/ark-core/<network>/snapshots/
http-server ./
```
3. Download your snapshot from another server through `wget` or something similar.

Note: the default port used is `8080`, so if you have a firewall running you need to make sure that the port is open!

Protip: you can make it easier to download a snapshot by tar'ing it, as that will combine the snapshot folder into a single file.

```bash
cdsnaps
# Combine a snapshot folder into a single file, make sure to rename mySnapshotFolder to the actual name of the snapshot folder
tar cvf mySnapshotFolder.tar mySnapshotFolder
```

On the receiving server you can then extract the tar by running `tar -xvf mySnapshotFolder.tar`

----

## Misc.

### Core File Locations

Core v2 gets installed globally through `yarn` and can be found in the following location:

```bash
~/.config/yarn/global/node_modules/@arkecosystem/
```

There are multiple locations that are used by core to store specific config or data:

```bash
CORE_PATH_DATA=$HOME/.local/share/ark-core/$network
CORE_PATH_CONFIG=$HOME/.config/ark-core/$network
CORE_PATH_CACHE=$HOME/.cache/ark-core/$network
CORE_PATH_LOG=$HOME/.local/state/ark-core/$network
CORE_PATH_TEMP=/tmp/$USER/ark-core/$network

cache       => $HOME/.cache/ark-core/$network
config      => $HOME/.config/ark-core/$network
database    => $HOME/.local/share/ark-core/$network/database
.env        => $HOME/.config/ark-core/$network/.env
logs        => $HOME/.local/state/ark-core/$network/logs
snapshots   => $HOME/.local/share/ark-core/$network/snapshots
temp        => /tmp/$USER/ark-core/$network
```

In the above paths, `$network` needs to be replaced by `mainnet`, `devnet` or `testnet`, depending on which network you currently are

### Disabling API Ratelimit

If you run into API Ratelimit issues on your node, you can disable it (or whitelist the IP that needs access). Disabling is done by adding `CORE_API_RATE_LIMIT=false` to your `.env` file.

### Whitelisting IPs

TODO

### Getting logs from server

You can fetch logs from your remove server through `scp`. Note that you need to replace `$network` with `mainnet` or `devnet`

Download the full log directory to your current directory:

```bash
scp -r username@server:~/.local/state/ark-core/$network/ .
```

Download a single log file to your current directory

```bash
scp username@server:~/.local/state/ark-core/$network/2019-03-11.log . # Change the filename to the correct date
```

If you use a different ssh port than the default `22`, you can run the above commands with `scp -P $port` instead, where `$port` is the port number you use.

----

## Common Issues / FAQ

- how to remove process from pm2: `pm2 delete ID` or `pm2 delete all`
- pm2 list showing `errored`; check the logs for more information on the actual problem
- issue with plugins; remove it from plugins.js or check if there's an update
- The "..." process has entered an unknown state.: `pm2 update` then `pm2 kill` and restart if it still was a problem

----

## Useful Documentation Links

* [ARK CLI Docs](https://docs.ark.io/guidebook/core/cli.html)
* [Creating Plugins](https://docs.ark.io/guidebook/developer/write-a-plugin.html)
* [ARK Core Docs](https://docs.ark.io/guidebook/core/)
* [Migrating v2.1 to v2.2](https://docs.ark.io/releases/v2.2/migrating_2.1_2.2.html)
* [Installing v2.2 from scratch](https://docs.ark.io/guidebook/core/cli.html#fresh-installation)
* [VoteReport](https://explorer.ark.io/VoteReport.txt)

---

## Suggestions

If you have any suggestions / improvements for this cheatsheet, feel free to open a PR or an issue to get it added!
