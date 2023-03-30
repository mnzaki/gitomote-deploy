# Intro
This repo contains a simple docker-compose deployment script that is integrated with [gitolite](https://gitolite.com/) and a utility script `gitomote` to be used from your local machine to adminstrate the deployed services without sshing directly there.

# Installation and Useage

Setup [gitolite](https://gitolite.com/) on your server.

Add the user that gitolite is run as to the `docker` group. Let's suppose you
installed it for the user `infra`
```sh
usermod infra -a -G docker
```

clone your gitolite installation's  gitolite-admin and copy the contents of
`gitolite-deploy/gitolite-admin` to it
```sh
git clone infra@yourdomain.com:gitolite-admin yourdomain-gitolite
cp -r gitolite-deploy/gitolite-admin/* yourdomain-gitolite/
```

edit the `.gitolite.rc` file to uncomment the line that enables `LOCAL_CODE`
loading from the gitolite-admin repo. This enables the `post-receive` hook.
```sh
cd /home/infra
sed -i 's/# LOCAL_CODE.*GL_ADMIN_BASE.*/LOCAL_CODE => "$rc{GL_ADMIN_BASE}\/local",/' .gitolite.rc
```

edit the `.gitolite.rc` file to add the `'gitolite-deploy'` command to the `ENABLE`
list
```sh
cd /home/infra
sed -i "s/\(new commands here.*\)/\1\n'gitolite-deploy',/" .gitolite.rc
```

now copy the `gitomote` utility script to somewhere in your `$PATH` on your
local computer
```sh
# assuming ~/bin is in your $PATH
cp gitolite-deploy/gitomote ~/bin
```

Use the `yourdomain-gitolite/conf/gitolite.conf` to add a new repo, clone the
repo to your machine, and add a `docker-compose.yml` file to it. Commit and push
to your `infra` gitolite. `docker-compose up -d --build` will be run
automatically.

And while in your local clone of any repo that is hosted on `infra` you can use
`gitomote` as a replacement for running docker-compose remotely
```
gitomote logs -f
```
