---
title: "Collaborative infrastructure for a lab: Forgejo"
date: 2025-03-02T06:00:00+02:00
author:
- Michael Hanke
tags:
- Forgejo
- self-hosted
- Git hosting
- podman
- systemd
- lab infrastructure
cover:
  image: cover.webp
  alt: 
  relative: true
description: >
  The first installment of a mini-series on a self-hosted, collaborative infrastructure:
  Run Forgejo on a small VPS or NAS as a central collaboration platform for a lab or group.
showToc: true
---

For the past 18 years I have been a GitHub user.
It has been an extremely convenient platform for collaborating with many people from all over the world.
What makes GitHub, and other platforms like it, particularly attractive is that they are typically way more accessible than any institutionally provided infrastructure ([even if not without issues of its own](https://en.wikipedia.org/wiki/Censorship_of_GitHub)).
GitHub also provided an extremely reliable and stable infrastructure that encouraged and rewarded building on it.
Again, from my personal experience, much more reliable and stable that lots of institutional infrastructures.

But the times of appreciation were over when things started to be about who can grab more data from any sources to train yet another "AI", violating licenses, ignoring basic ethical behavior -- and GitHub was at the forefront of it.
And I felt trapped, because I couldn't really see an affordable way out, other than switching one poison for another.

This all changed for me in 2024, when we had the first [distribits meeting](https://distribits.live) in April.
In the months after this meeting, it became clear to me that [Forgejo](https://forgejo.org), and [Forgejo-Aneksajo](https://codeberg.org/forgejo-aneksajo/forgejo-aneksajo) could be the basis for a sustainable, self-hosted collaboration infrastructure.
I did explore its utility in blog posts about [hosting it on a Raspberry Pi](/posts/forgejo-aneksajo), or [deployment with podman](/posts/forgejo-aneksajo-podman-deployment/), and took a look at [its implementation of "actions" for automation](/posts/forgejo-runner-podman-deployment/).
For the past six months, I deployed a number of Forgejo instances, and learned a great deal about what works well, and not so well, for particular use cases.
At the same time, [Forgejo moved to a GPL license](https://forgejo.org/2024-08-gpl), asserting its commitment to software freedom.
[Matthias Riße](https://codeberg.org/matrss) kept Forgejo-Aneksajo in-sync with the development of Forgejo proper, integrated and enabled new [git-annex](https://git-annex.branchable.com) features, such as [running its P2P protocol over HTTP](https://git-annex.branchable.com/design/p2p_protocol_over_http/).

Altogether, these development have made it possible for me to initiate a migration off of GitHub, and other US tech giant provided services too. A migration for the work I do personally, but also a migration for the [psychoinformatics group](https://psychoinformatics.de) I am heading at work.
In a series of posts, I will document the setup we arrived at (for now).
While there is some overlap with previous posts, the setup has changed significantly enough (and for the better) to warrant an update. So here we go...

## The centerpiece: Forgejo-Aneksajo

A good infrastructure is first and foremost a reliable one.
Things need to run every day, for everyone involved.
Someone needs to make sure that this is the case, and people's time and attention are the big limiting factors.
Therefore, there is a huge incentive to keep things simple, maintainable, and disentangled.

The vast majority of the tools we need (Git hosting, issue tracker, package distribution, automation for CI/CD) can be provided by [Forgejo](https://forgejo.org).
Combined with [git-annex](https://git-annex.branchable.com) in [Forgejo-Aneksajo](https://codeberg.org/forgejo-aneksajo/forgejo-aneksajo), it also facilitates data-intensive workflows and hosting.
This makes Forgejo-Aneksajo the centerpiece of the infrastructure.
Future posts in this mini-series will describe a few more infrastructure components that provide functionality that Forgejo does not.
But they are all either not mission-critical, or somehow feed into the central Forgejo site -- making it the one piece that needs to run reliably, and have its content backed up and preserved to maintain the value of the group's output.

## Hosting

Forgejo has a very modest resource footprint, enabling a wide range of hosting scenarios.
At home I do run it on a Raspberry Pi, but for our group we needed something more robust.

We host Forgejo on a virtual private server with six cores, 12GB RAM, and 300GB SSD disk space.
It is provided by [Contabo](https://contabo.com/en), and costs about 180 Euro a year.
This is not intended to be an endorsement of this particular company, just a statement of a fact.
What is important to me is that:

- the host machine is physically in Germany, covered by the same laws and regulations as the group itself
- the provider offers API-accessible system snapshotting (and rollback)
- file system snapshots and incremental backups are automated by the provider and require no client-software
  to be installed on the system

Other than this basic service, there is no specific tailoring or dependency to this particular provider.
A similar setup as described here should be possible with self-hosted hardware, such as a [Synology NAS](https://effigies.gitlab.io/posts/forgejo-aneksajo-synology/) (again not an endorsement).
For us, the key aspect is that a future provider switch should be (and stay) easily possible.
Off-site backups to other services will be a topic in a future post.


## Deployment

[Forgejo-Aneksajo](https://codeberg.org/forgejo-aneksajo/forgejo-aneksajo) is deployed as a containerized service behind a reverse proxy ([Caddy](https://caddyserver.com)), as [described previously](/posts/forgejo-aneksajo-podman-deployment). However, there are a few key differences:

- the host system is relatively bare, and the integration of the containers with the rest of the system is kept as minimal as possible
- only administrators have shell access (SSH) to the machine
- there is no SSH-access to Forgejo at all
- all services apart from the webserver run rootless in user space, and each service under its own user account

Taken together, this makes it possible to provision a replacement host in a short amount of time and a few steps, without having to maintain somethings like an [Ansible](https://en.wikipedia.org/wiki/Ansible_(software)) setup.
Any and all configuration and data related to a particular service are fully self-contained in the service user's home directory (incl. container images, systemd service units).
They can be moved with something as simple as `rsync`, whenever a need arises.

All services are provided via HTTP(S)-only, making port 443 (and/or 80) the only exposed port apart from an arbitrarily locked down SSH-access for administrators only.

## Host system

The starting point is a basic installation of [Debian 12](https://www.debian.org/releases/bookworm/).
The following steps describe all additional customizations (performed as `root`), leading to a base installation of under 2GB size.

As a first step, we install [etckeeper](https://etckeeper.branchable.com).
It is not strictly necessary, but it will reliably and automatically track any modification of the base system configuration -- manual and automatic ones -- throughout the years.
It provides a zero-effort log and safety net.
Moreover, it yields a Git repository that can be hosted on Forgejo too.

```bash
apt install --no-install-recommends etckeeper
```

Next, we erect a firewall with [UFW](https://launchpad.net/ufw).
Again, other solutions exist, but I know this one and it is simple enough.

```bash
apt install ufw
ufw allow ssh
ufw allow https
# for unencrypted HTTP if desired
ufw allow www
ufw enable
```

```console
> ufw status
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
443                        ALLOW       Anywhere                  
22/tcp (v6)                ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
443 (v6)                   ALLOW       Anywhere (v6)             
```

A great companion for a firewall is [Fail2Ban](https://github.com/fail2ban/fail2ban).
Even without particular tuning it helps to protect the server against malicious activities.

```bash
apt install fail2ban
```

The essential host system setup is finished with a few more packages:

```bash
# essential helper, not needed here, but later
apt install rsync --no-install-recommends
# webserver/reverse-proxy solution of choice
apt install caddy
# container solution of choice
apt install podman
```

Again, there are other solutions for containers (e.g., docker), and reverse-proxies (e.g., traefik), and directory synchronization (e.g., unison).
But personally find the ones above best and simplest for the job.


I do add a last set of packages to tailor the admin environment with some essential helpers that I have grown accustomed to:

```bash
apt install --no-install-recommends vim htop mc net-tools pipx ncdu tree tig git-annex
```

Lastly, I add a non-root admin user (fictive name `thedude` here) and enable `sudo` for it.

```bash
apt install sudo
adduser thedude
adduser thedude sudo
```

## Root-less Forgejo-Aneksajo service

As for every other coming service, we deploy the Forgejo via a container that is managed by [Podman](https://podman.io) in user space. The pattern for that is always the same:

- create a user, disabling password (and often login)
- enable execution of processes for that user when not logged in
- set up systemd and service units
- perform other service-specific configuration

The target is always to contain all configuration and runtime data in the user's HOME directory.
This facilitates setting and enforcing quotas (if needed), and service migration to other machines.

Using the non-root admin account on the host system we set up Forgejo to run under the `git` user
account with the following steps.

```bash
# create user
sudo adduser git --disabled-password --disabled-login
# enable process execution when not logged in
sudo loginctl enable-linger git
```

We can disable the login for the `git` user, because we will not offer SSH-based access to
Forgejo via [SSH-passthrough](/posts/forgejo-aneksajo-podman-deployment/#ssh-passthrough).
Instead, all write access, including git-annex data transfers will go over HTTPS.
This makes it unnecessary for people that do not already use SSH-keys to create and maintain them.
It also removed complexity from the Forgejo-Aneksajo setup.

Next we prepare the `git` user for `systemd` services in user space.

```bash
# switch to the `git` user
sudo -su git
# enter its HOME
cd
# establish configuration ensuring smooth `systemctl --user` operation
cat << EOT >> .bashrc
if [ -z "\${XDG_RUNTIME_DIR}" ]; then
  XDG_RUNTIME_DIR=/run/user/\$(id -u)
  export XDG_RUNTIME_DIR
fi
EOT

# create destination directory for user systemd service units
mkdir -p .config/systemd/user
```

Next, we create four directories to hold the Forgejo deployment:

- `git` will contain any and all Git repositories hosted on the site as regular bare repositories, including a git-annex if present
- `forgejo` will contain the actual Forgejo installation with its databases, package repositories, etc
- `conf` will contain the site configuration file, and potentially secrets like mail server credentials
- `custom` will contain any site-specific customizations, such as logos and other assets, custom page templates, etc.

```bash
mkdir forgejo git custom conf
```

These four directories could be kept plain, and subject to generic backup strategies.
However, I do like `conf` to be a Git repository that tracks configuration changes,
and `custom` to be a git-annex repository that can be shared to aid management of similar deployments.
The later is a git-annex repository, because custom assets can be quite large and volatile for some use cases.
Both are hosted on the Forgejo site in a dedicated organization.

Now we can drop in a systemd service unit for Forgejo.
Only three things are worth pointing out:

- `hub.datalad.org/forgejo/forgejo-aneksajo:10-rootless-amd64` selects the container image to execute.
  Upgrading Forgejo means changing the major version in the tag.
- `--userns keep-id:uid=1000,gid=1000` sets the ownership of the files on the **host system** that are written by the Forgejo container. `uid` and `gid` must match the output of `id` when executed by the `git` user.
- the `-v` volume mounts compose the four directories we created above in the hierarchy that is expected by Forgejo inside the container. This achieves a simple, flat organization for an admin view, while keeping everything standard inside the container.

```console
> cat << EOT > /home/git/.config/systemd/user/forgejo.service
[Unit]
Description=Podman-managed Forgejo service
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=always
TimeoutStopSec=300
ExecStartPre=/bin/rm \\
        -f %t/%n.ctr-id
ExecStart=/usr/bin/podman container run \\
        --cidfile=%t/%n.ctr-id \\
        --cgroups=no-conmon \\
        --rm \\
        --sdnotify=conmon \\
        -d \\
        --replace \\
        --name forgejo \\
        -p 3000:3000 \\
        -v /home/git/forgejo:/var/lib/gitea:Z \\
        -v /home/git/git:/var/lib/gitea/git:Z \\
        -v /home/git/custom:/var/lib/gitea/custom:Z \\
        -v /home/git/conf:/var/lib/gitea/custom/conf:Z \\
        --userns keep-id:uid=1000,gid=1000 \\
        hub.datalad.org/forgejo/forgejo-aneksajo:10-rootless-amd64
ExecStop=/usr/bin/podman stop \\
        --ignore -t 10 \\
        --cidfile=%t/%n.ctr-id
ExecStopPost=/usr/bin/podman rm \\
        -f \\
        --ignore -t 10 \\
        --cidfile=%t/%n.ctr-id
Type=notify
NotifyAccess=all

[Install]
WantedBy=default.target
EOT
```

With the service unit in place, we can launch Forgejo

```bash
systemctl --user start forgejo
```

On the first launch this will take a moment, because it will pull the necessary container image first.
Once launched successully, we can `enable` this service unit, so it will be restarted when the host machine reboots.

```bash
systemctl --user enable forgejo
```

## Configuration

At this point we can access Forgejo on port `3000` on the host machine.
If the site should be accessible on a custom (sub)domain, it is best to configure the reverse-proxy now, to aid Forgejo's configuration self-detection.
With `caddy`, this is as simple as dropping a short snipped into its configuration file (here shown for out `hub.psychoinformatics.de` site, please adjust accordingly):

```console
# as root
> cat << EOT >> /etc/caddy/Caddyfile
hub.psychoinformatics.de {
    reverse_proxy localhost:3000
}
EOT
```

Now we can reload `caddy`, and it will obtain SSL certificates automatically to enable HTTPS communication. Before reloading caddy, the DNS `CNAME` or `A RECORD` must be set up already.

Now visit the Forgejo installer at port `3000` or the configured (sub)domain.
It makes sense to configure SMTP server for email notification, set up an admin user, and disable LFS.
I have never tried any other database backend then `sqlite` and never had a need to.
Even with dozens of users and thousands of repositories `sqlite` appears to be just fine.
A switch to a proper database server should not be made without proper consideration of the implications on backup strategies to hold filesystem and database content in sync during backup and rollback.
Other than that, the particular choices of configuration made in the installer are not really critical and can be rectified and amended in the next step.

Once the installed has finished, it will have written an `app.ini` file in the `conf` directory.
If desired, this can now be committed to the initialized Git repository (and push to the newly deployed Forgejo site).
Now is also the time to further customize the configuration.
The following `diff` comparison to a default configuration might provide some inspiration:

```diff
--- a/app.ini
+++ b/app.ini
@@ -1,11 +1,22 @@
-APP_NAME = Forgejo
+APP_NAME = Psychoinformatics Hub
 RUN_USER = git
 RUN_MODE = prod
-APP_SLOGAN = Beyond coding. We Forge.
+APP_SLOGAN = Powered by Forgejo-aneksajo.
 WORK_PATH = /var/lib/gitea
 
+[ui.meta]
+AUTHOR = "Psychoinformatics group"
+DESCRIPTION = "Forgejo-aneksajo is Forgejo with git-annex superpowers. It just does the job, but even better."
+KEYWORDS = git,forge,forgejo,forgejo-aneksajo,git-annex
+
 [repository]
 ROOT = /var/lib/gitea/git/repositories
+ENABLE_PUSH_CREATE_USER = true
+ENABLE_PUSH_CREATE_ORG = true
+DEFAULT_PUSH_CREATE_PRIVATE = true
+DEFAULT_PRIVATE = private
+DISABLED_REPO_UNITS = repo.wiki,repo.ext_wiki
+DEFAULT_REPO_UNITS = repo.code,repo.pulls,repo.packages
 
 [repository.local]
 LOCAL_COPY_PATH = /tmp/gitea/local-repo
@@ -18,13 +29,13 @@ APP_DATA_PATH = /var/lib/gitea
 SSH_DOMAIN = hub.psychoinformatics.de
 HTTP_PORT = 3000
 ROOT_URL = https://hub.psychoinformatics.de/
-DISABLE_SSH = false
+DISABLE_SSH = true
 ; In rootless gitea container only internal ssh server is supported
-START_SSH_SERVER = true
-SSH_PORT = 2222
-SSH_LISTEN_PORT = 2222
-BUILTIN_SSH_SERVER_USER = git
-LFS_START_SERVER = true
+START_SSH_SERVER = false
+;SSH_PORT = 2222
+;SSH_LISTEN_PORT = 2222
+;BUILTIN_SSH_SERVER_USER = git
+LFS_START_SERVER = false
 DOMAIN = hub.psychoinformatics.de
 OFFLINE_MODE = true
+
+[annex]
+ENABLED = true
+
+[packages]
+ENABLED = true
```

I won't go into much detail on what this does.
A good place to start reading up on these settings is the [Forgejo configuration cheat sheet](https://forgejo.org/docs/latest/admin/config-cheat-sheet/).
Critical is only

```ini
[annex]
ENABLED = true
```

Without this flag, the git-annex features are not available, and the site will act more-or-less like Forgejo, and not Forgejo-Aneksajo.

Disabling SSH is a choice, but a non-SSH setup is arguably simpler.

Further site customization can be done by adding content to the `custom` directory.
However, be mindful of the recommendations in the Forgejo admin documentation [on interface customization](https://forgejo.org/docs/latest/admin/customization/). For some inspiration, you are welcome to explore the [customizations of `hub.psychoinformatics.de`](https://hub.psychoinformatics.de/hub/custom).


## Conclusions

In this first post of this series, we have established the basic environment for putting together a collaborative infrastructure for a (research) group.

We now have a server deployment with a fairly bare bone configuration, where only admins need shell access.
We also have a Forgejo-Aneksajo deployment that already provides key features of this infrastructure:

- Git/data hosting
- issue tracking and helpers for coordinating work
- managing software releases and artifacts

Most choices documented here are not critical.
Other hosting providers would work equally well, including deployment on own, on-premise hardware.
The choice of Debian 12 as the base system is not without alternative either, but a trusted and reliable foundation for me.

Based on this foundation, future posts will show how this setup can be further tailored and extended to cover more aspects of collaborative work.
Some of the topics will be workflows for review and automated deployment of (project) websites, and shared group calendar and TODO lists.
