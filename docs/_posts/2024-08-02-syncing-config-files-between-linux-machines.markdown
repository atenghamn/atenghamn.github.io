---
layout: post
title: "Syncing .configuaration files between linux machines"
date: 2024-08-02 15:21:00 +0200
categories: linux
---

So I'm currently on vacation and vacation is a great opportunity to do the things you never have time to do the rest of the year. So instead of fixing the garden or some other stuff that is should REALLY do I decided it was time to sync my configuration files between my different machines.

## The problem
The background is that I have three computers I use on an almost everyday basis. The first one is my work laptop that runs Ubuntu, the second one is my own computer that also runs Ubuntu and the third one is server that runs on top of a rpi4 with [DietPi](https://dietpi.com). Having these different machines for different purposes still means that I have some stuff that I use on all three of them. Some of the things I use like [Nala](https://github.com/volitank/nala) or [Starship](https://starship.rs/) is pretty straight forward to set up and I don't use a lot of customizations or plugins. But other things like [Neovim](https://neovim.io/) or [Tmux](https://github.com/tmux/tmux/wiki) is so heavily customized that I feel like jumping of a cliff when getting a new computer and having to reproduce my previous setup. 

## The solution
  The idea I got was to push every config file that I need to a github [repo](https://github.com/atenghamn/.configs) and then setup a service on every machine that pulls the repo and check if there is any changes from whats on the machine. If the local changes are newer those should be pushed to the repo. Otherwise the local config files should be overwritten by the those from the repo. I didn't want to have it as a script that required me to manually run it (because then I would just forget about it), instead I would like it to run everyime I reboot the machine (approximately once a week). To be honest I'm not shure if I stick with this, It might be that I set it to run at a given time interval instead since I never reboot the server but for now this will do. 

The first step after setting up the github repo was to create a bash script that handles the logic behind the actual file handling.

```bash
#!/bin/bash

# Set variables
REPO_URL="atenghamn/.configs"
LOCAL_REPO_DIR="$HOME/Repo/Configs/.configs/"
TMUX_CONF="$HOME/.config/tmux/tmux.conf"
NVIM_CONF_DIR="$HOME/.config/nvim"
REMOTE_TMUX_CONF="$LOCAL_REPO_DIR/tmux.conf"
REMOTE_NVIM_CONF_DIR="$LOCAL_REPO_DIR/nvim"

# Clone the repository if it doesn't exist, otherwise pull the latest changes
if [ ! -d "$LOCAL_REPO_DIR" ]; then
  gh repo clone "$REPO_URL" "$LOCAL_REPO_DIR"
else
  cd "$LOCAL_REPO_DIR" || exit
  git pull origin main
fi

# Function to check and replace configs
replace_config() {
  local src=$1
  local dest=$2

  if ! diff -q "$src" "$dest" >/dev/null 2>&1; then
    echo "Changes detected. Replacing $dest with $src"
    cp -r "$src" "$(dirname "$dest")"
  else
    echo "No changes detected. Pushing $dest to the repo"
    cp -r "$dest" "$src"
    cd "$LOCAL_REPO_DIR" || exit
    git add .
    git commit -m "Updated configurations from local machine"
    git push origin main
  fi
}

# Check and replace tmux config
replace_config "$REMOTE_TMUX_CONF" "$TMUX_CONF"

# Check and replace neovim config
replace_config "$REMOTE_NVIM_CONF_DIR" "$NVIM_CONF_DIR"
```
As you can see it's a pretty straight forward script. After I created this I ran: 
```
mv sync_configs.sh /usr/local/bin/
sychmod +x /usr/local/bin/sync_configs.sh
```
This moved it to the appropriate folder and made it executable. 

The next step was to create a systemd service. If you are new to systemd processes It's worth to read up on [it](https://www.linux.com/training-tutorials/understanding-and-using-systemd/). 

```bash
sudo systemctl enable sync-configs.service
sudo systemctl start sync-configs.service
sudo systemctl status sync-configs.service
```

So this was actually all that there was to it. Now I have the same setup on all my machines! 
If you want to do the same thing just follow along this guide and replace the paths with your own. 
