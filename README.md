# Gitomote

Simple push-to-deploy system as a post-receive hook integrated with
[gitolite](https://gitolite.com/) and a utility script `gitomote` to be used
from your local machine to adminstrate the deployed services without using ssh
directly. Combine with persistent shared ssh connections and you have seamless
remote docker-compose access.

## Installation and Setup

You will need a separate unix user on your server to act as the owner of the
stored bare git repositories, and trigger their deployment. I like to use the
name `infra` so that the deployment repos have a remote address like
`infra@yourserver.com:nginx`

### Clone gitomote-deploy locally

```sh
# clone the gitomote repo on your local machine
git clone git@github.com:mnzaki/gitomote-deploy
cd gitomote-deploy
# Now link gitomote somewhere in your $PATH
# assuming ~/bin is in your $PATH
ln -s gitomote ~/bin
```

### Install [gitolite](https://gitolite.com/) on yourserver.com

But first, you are going to need your personal ssh key handy, copy it to the clipboard

```sh
# with xsel
xsel -b < ~/.ssh/id_rsa.pub
# or with xclip
xclip -selection clipboard -i < ~/.ssh/id_rsa.pub
```

Now ssh to the server, and let's setup gitolite

```sh
ssh yourserver.com

# if you are using Arch Linux
pacman -Sy gitolite

# if you are using a Debian
apt-get install gitolite3

# Add the user that gitolite is run as to the `docker` group. Let's call it
# `infra`. We will also add it to the `docker` group so that it can talk to the
# docker daemon
useradd -m -G infra,docker infra
su infra # switch to user infra
cd # change directory to the user home

echo paste your ssh public key then press Ctrl-D
cat >> yourname.pub
# paste your key with Ctrl-V then press Ctrl-D
```

Let's setup gitolite for this user. You might have to change the command to
`gitolite3` depending on your linux distribution.
```sh
# still inside `su infra` on your server
gitolite setup -pk yourname.pub

# Stream EDit the .gitolite.rc file
## uncomment the line that enables `LOCAL_CODE` loading from the
## `gitolite-admin` repo
sed -i 's/# LOCAL_CODE.*GL_ADMIN_BASE.*/LOCAL_CODE => "$rc{GL_ADMIN_BASE}\/local",/' .gitolite.rc
## add the `gitomote-deploy` command to the list of `ENABLE`d commands
sed -i "s/\(new commands here.*\)/\1\n'gitomote-deploy', 'gitomote-ssh',\n/" .gitolite.rc

## set the AUTH_OPTIONS to allow ssh to grab a PTY
sed -i "s/^\(%RC = .*\)/\1\nAUTH_OPTIONS => 'no-port-forwarding,no-X11-forwarding,no-agent-forwarding',\n/" .gitolite.rc
```

That's all! Now back to our local machine, let's deploy something.

## Usage

To administrate the gitolite installation, you need to clone the automatically
generated `gitolite-admin` repo on your local machine, or just use the
`gitomote setup` command

```sh
# you can use the gitomote-deploy dir to manage your servers
cd gitomote-deploy
mkdir yourserver.com
gitomote setup infra@yourserver.com ./yourserver.com
```

Now let's say we wanna deploy [Zulip](https://zulip.com)
```sh
# Edit the `infra-gitolite/conf/gitolite.conf` to add new repos
cd ./yourserver.com/infra-gitolite
cat >> conf/gitolite.conf<<EOF
repo zulip
    RW+     =   @all
EOF
git add conf
git commit -m "repo: zulip"
git push

# and clone that to use it
git clone infra@yourserver.com:zulip
cd zulip
# it's empty
# add a `docker-compose.yml` file to it
vim docker-compose.yml
gitomote deploy
```

### Remote docker-compose

While in any deployment directory you can use the `gitomote` with the same
arguments as with `docker-compose`; it will run `docker-compose` remotely.

```sh
cd zulip
gitomote ps
gitomote logs -f
```

### Redeploying

Just make changes then `git commit && git push` to deploy! This will run
`docker-compose up -d --build --remove-orphans` remotely.

You can also use `gitomote deploy` which will help you commit changes and update
submodules and push and start tailing logs, all in one go.

## See Also

- Automatic nginx virtualhosts and letsencrypt certs https://github.com/mnzaki/docker-auto-revhttps
- share ssh connections: https://www.linuxjournal.com/content/speed-multiple-ssh-connections-same-server
