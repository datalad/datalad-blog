---
title: 'Putting new git-annex features to use with Nextcloud'
date: 2025-02-20T20:30:00+01:00
author: Michał Szczepanik
draft: true
---

Git-annex continues to evolve. In this post, I want to look at two changes, one big and one small, introduced over the last seven months. Together, they make publishing files through Nextcloud much nicer.

In git-annex's changelog, we find the following:

> - 10.20240531: git-remote-annex: New program which allows pushing a
>   git repo to a git-annex special remote, and cloning from a special
>   remote.  (Based on Michael Hanke's git-remote-datalad-annex.)
> - 10.20250115: Allow enableremote of an existing webdav special remote
>   that has read-only access

The former is the big one, and it is introduced in a git-annex tips page: [storing a git repository on any special remote](https://git-annex.branchable.com/tips/storing_a_git_repository_on_any_special_remote/). In short, the change introduced a Git remote helper---when Git is asked to clone from a specifically crafted URL, it turns to git-annex for help---and git-annex can fetch the repository deposit from places not typically accessible to Git. This means that Git-aware hosting is no longer necessary to clone, push, and pull.

The latter sounds small in comparison, but it opened a way for a read-only password-protected shared Nextcloud folder to become a one-stop-shop for accessing repository with data thanks to git-remote-annex. This is the scenario I will describe here.

## Nextcloud and WebDAV

Why the focus on Nextcloud? I am under the impression that it is currently enjoying a lot of popularity as a self-hosted cloud solution. In a workshop for research data managers last year, I sensed a lot of positive sentiment towards Nextcloud among my colleagues. The [FOSDEM 2025 talk "What's new in Nextcloud?"](https://fosdem.org/2025/schedule/event/fosdem-2025-4515-what-s-new-in-nextcloud-/) (which I had the pleasure to attend, and can attest that the room was packed) gives an idea what Nextcloud can do beyond just storage. Relevant for my work, the German state of North Rhine-Westphalia operates a cloud storage service for research, studying and teaching called [Sciebo](https://hochschulcloud.nrw/en/), which is about to migrate from Owncloud to Nextcloud. And personally, I use a managed solution from one of the commercial providers.

What is important in this context is that Nextcloud provides WebDAV as a way to access files. This means that it can be used with the git-annex [WebDAV special remote](https://git-annex.branchable.com/special_remotes/webdav/).

Folders shared from Nextcloud's web interface can also be accessed via WebDAV. There are some rules which govern the access paths in different scenarios (named users or public links; for details, see [Nextcloud's WebDAV documentation](https://docs.nextcloud.com/server/20/user_manual/en/files/access_webdav.html) and a summary in the Psychoinformatics Knowledge Base item on [Nextcloud URL patterns](https://knowledge-base.psychoinformatics.de/kbi/0028/index.html#nextcloud-url-patterns)). Here, we will focus on a share via a public link which is read-only and password-protected, using an example instance address:

- Owner's WebDAV path to a folder is: `https://nextcloud.example.com/remote.php/dav/files/jdoe/some/folder`;
- The generated share link is: `https://nextcloud.example.com/s/3DQbgnWnTmMGi4z`;
  - The last link component (`3DQbgnWnTmMGi4z`), also called the share token, becomes the recipient's WebDAV username;
  - The password set for the share link becomes the WebDAV share password;
- The public WebDAV path is: `https://nextcloud.example.com/public.php/webdav` (no folder components).


This can be used with git-annex!

## Software requirements

Using the features described here requires at least git-annex version 10.20250115. This is much newer than what is available from the official repositories on many GNU/Linux distributions at the time of writing. Since I am on Debian-stable, I used a git-annex build from [here](https://dl.kyleam.com/git-annex/) (v10.20250115). Standalone git-annex builds are available [here](https://downloads.kitenet.net/).

To focus on git-annex, I will be using Git and git-annex commands in the examples, although naturally they can be interweaved with DataLad commands.

## Special remote and read-only share

In this example, we want to use a WebDAV special remote to store data on a Nextcloud instance. We then want to create a public share link (read-only, password protected) to share the data with selected recipients, also using git-annex. The Git repository is (for now) shared through other means.

First, we create a mock repository and add a file:

```
cd /tmp/ds
git init
git annex init
head -c 256k < /dev/urandom > foo.dat
git annex add foo.dat
git commit -m "Added random data"
```

Then we add a WebDAV special remote. The user-specific WebDAV URL can be found in Nextcloud's web UI, and we add path components as needed.

```
WEBDAV_USERNAME=jdoe WEBDAV_PASSWORD=$MYTOKEN \ 
  git annex initremote nextcloud \
  type=webdav encryption=none \ 
  url=https://nextcloud.example.com/remote.php/dav/files/jdoe/some/folder
```

We push (or copy) the contents to the remote:

```
git annex push nextcloud
```

The remote, as it is, has one problem in terms of sharing: its URL is user-specific. However, git-annex has an answer for that: the `--sameas` parameter declares that remotes use the same underlying storage. Because initremote performs a write test, and we intend for the share link to be read-only, we do it with the same URL first...

```
WEBDAV_USERNAME=jdoe WEBDAV_PASSWORD=$MYTOKEN \
  git annex initremote nextcloud-public --sameas nextcloud \
  type=webdav encryption=none \
  url=https://nextcloud.example.com/remote.php/dav/files/jdoe/some/folder
```

... only to generate the share link and reconfigure the remote afterwards. Here, the changed enableremote behavior becomes crucial.


```
git annex enableremote nextcloud-public \
  url=https://nextcloud.example.com/public.php/webdav
```

In practice, at this point we would publish the Git repository somewhere else. To simulate the consumer's perspective, we clone the repository locally:

```
cd /tmp
git clone ds ds-clone
cd ds-clone
```

The special remote can be enabled with the share token and password as the WebDAV credentials -- it is for this moment that we needed the ability to enable without write permissions:

```
WEBDAV_USERNAME=$SHARETOKEN WEBDAV_PASSWORD=$SHAREPASSWORD \ 
    git annex enableremote nextcloud-public
```

And the file can be obtained:

```
git annex get foo.dat --from nextcloud-public
```


## git-remote-annex

Above, we simulated taking the "traditional" approach, where the git-annex special remote is separate from the Git remote. Here we will see how git-remote-annex can help us do away with Git hosting.

Squeezing everything into Nextcloud has downsides compared to using a Git-aware hosting (no web visualization, no pull request workflows, conflicted pushes done at the same time can overwrite each other), but it is a viable use case when a Git(+annex) platform is not available or needed.

First, back to the owner's perspective.

```
cd /tmp/ds
```

Enabling the git-remote-annex is easy.

```
git annex enableremote nextcloud --with-url
git annex push nextcloud
```

On push, git-annex will report the full remote URL, one that we (as owners) could use to clone:

```
push nextcloud
Full remote url: annex::2cd3b81c-a794-4e8e-9e00-527b46efe2d9?encryption=none&type=webdav&url=https%3A%2F%2Fnextcloud.example.com%2Fremote.php%2Fdav%2Ffiles%2Fjdoe%2Fsome%2Ffolder
To annex::
 * [new branch]      main -> synced/main
 * [new branch]      git-annex -> synced/git-annex
ok
```

If we browse the folder in Nextcloud's web interface, we can see that there are now more files, with key names starting with `GITMANIFEST` or `GITBUNDLE`. For more details, see [git-remote-annex](https://git-annex.branchable.com/git-remote-annex/) manual.

## Clone and get from a read-only share

Now, we can put the two together. We switch the perspective back to the recipient, who only needs to know the WebDAV URL to clone from.
The URL can be crafted by putting together:

- special remote ID (here: `2cd3b81c-a794-4e8e-9e00-527b46efe2d9`),
- special remote parameters (`?encryption=none&type=webdav`),
- Nextcloud instance-specific public WebDAV URL (`&url=https://nextcloud.example.com/public.php/webdav`).

```
PUBLIC_URL="annex::2cd3b81c-a794-4e8e-9e00-527b46efe2d9?encryption=none&type=webdav&url=https://nextcloud.example.com/public.php/webdav"
```

This URL can be cloned from. As previously, share token and password are used as WebDAV credentials.

```
cd /tmp
WEBDAV_USERNAME=$SHARETOKEN WEBDAV_PASSWORD=$SHAREPASSWORD \ 
  git clone $PUBLIC_URL ds-clone-2
```

Because webdav and webdav-public remain technically two separate remotes, getting from the origin at this point would try to use the owner's URL for getting data (even though we cloned through the public URL). The recipient still needs to enable the -public remote after cloning.

```
cd ds-clone-2
WEBDAV_USERNAME=$SHARETOKEN WEBDAV_PASSWORD=$SHAREPASSWORD \ 
  git annex enableremote nextcloud-public
```

And the file can be obtained as previously.

```
git annex get foo.dat --from nextcloud-public
```

## Summary

Since version 10.20240531, a new use-case is possible with git-annex and Nextcloud.
In this use case, annexed data are published together with the Git repository to a folder on a Nextcloud instance, which gets shared with git-annex-using recipients without giving them write access (the share can optionally be password protected).
The folder owner only needs to take care of two additional git-annex configurations: setting up an additional special remote pointing to a public WebDAV URL of their instance, and enabling git-remote-annex.
The data recipients can clone the data from a somewhat lengthy but easily generated URL, and enable the special remote using share-specific credentials obtained from the owner.
