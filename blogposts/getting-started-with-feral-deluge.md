```
author: @ekilah
created_at: 2020-05-16
updated_at: 2020-05-17
```

# Getting Started with Feral and Deluge

Getting a new seedbox up and running on Feralhosting with the Deluge client is mostly easy.

Feralhosting is a cheap option for a hosted seedbox. They provide some click-to-install software options for common use cases, and their storage and data rates seem fairly competitive. My limited contact with their support team has also been pleasant.


## Installing dependencies

### On the seedbox
Clicking around on the Feral dashboard, you can install the Deluge client from their Software page. Feral will provide a URL and login information shortly after choosing to install it. The client you install on Feral will run 24/7, and the web UI is good enough to get started. 

Later, we'll also (optionally) set up a local Deluge client that can control the daemon running on your seedbox, so you don't have to use the web UI if you prefer a native client.

- SSH to your Feral box: `ssh username@boxname.feralhosting.com`. Your login details are displayed on the Software page of Feral's dashboard.
- Use this command to get info about your Deluge daemon, so you can connect to it from your local client. This will display the host, username, port, and password you'll enter into the local client soon:

```bash
printf "$(hostname -f)\n$(whoami)\n$(sed -rn 's/(.*)"daemon_port": (.*),/\2/p' ~/.config/deluge/core.conf)\n$(sed -rn "s/$(whoami):(.*):(.*)/\1/p" ~/.config/deluge/auth)\n"
```


### On your local machine

Follow these instructions to get a local Deluge client running, if you don't prefer the web client. It's my understanding that most plugins for Deluge will require using something other than the web client, and managing the UI is just a bit nicer when you can do things like right-click (imo).

- Install [Deluge](https://dev.deluge-torrent.org/wiki/Download).
	- NB: You must be on the same major version of Deluge as Feral has installed for you on your seedbox. At the time of writing, Feral is installing 1.x for you, so don't use the 2.x client locally - the 2.x client can't connect to the 1.x daemon (I found this out the hard way).	
- Start the local Deluge client and set it to "thin client" mode. This "thin" mode will enable you to control the daemon running on your seedbox, to e.g. add/manage/remove files. Then, enter the details from the `printf` in the last step into the Deluge Connection Manager's "New Connection" wizard, and connect to the daemon.

> If it doesn't let you click "Connect", the port Feral allocated to you for incomming connectiosn might be conficting with another user on the same box. Contact their support if this happens; they may have to change your config for you.
> 
> You can tell if there is an issue by looking at `~/.config/deluge/deluged.log` on your Feral box, and look for this error: `Couldn't listen on any:XXXXX: [Errno 98] Address already in use.` (Xs are the port number from the `printf` above) ([source](https://www.feralhosting.com/wiki/software/deluge#troubleshooting)).

### From either client

These steps will work no matter which way you choose to connect to the daemon on the seedbox - via the web UI, the local thin client, or even with the `deluge-client` CLI.

- Configure your Feral daemons's up/down limits and other settings before getting started.
	- Some setups or third-party services may come with recommendations about certain settings to enable/disable in your client.


### Optional stuff:

- Set up an SSH alias to make SSHing to your Feral box easier (so you can type `ssh ssh_alias_here` instead of `ssh my_usnername_here@my_boxname_here.feralhosting.com`:
	- Create or append to `~/.ssh/config`, replacing all the `*.here`s with your own stuff:
	
	```
	Host ssh_alias_here
		User my_ssh_username_here
		HostName my_boxname_here.feralhosting.com
		IdentityFile ~/.ssh/id_rsa
	```

- Copy your public key to your server (example Mac instructions [here](https://apple.stackexchange.com/a/210133)).
    - This avoids having to enter your password on every SSH login. 

And that's it! You've got a cheap seedbox with great bandwidth and a simple configuration ready to go.

