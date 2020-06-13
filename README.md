# `consumable-coos`  <!-- omit in toc -->

I want to locally play with [Google's `coos`](https://cloud.google.com/container-optimized-os) in both `vagrant` and containers but they don't publish an image ([AWS does](https://hub.docker.com/_/amazonlinux) and [Azure builds on top of Ubuntu](https://github.com/Azure/AKS/tree/master/vhd-notes/aks-ubuntu); Google builds their own OS from scratch).

## Contents <!-- omit in toc -->
- [Prerequisites](#prerequisites)
  - [Dependencies](#dependencies)
    - [System](#system)
    - [`depot_tools`](#depot_tools)
    - [`(git-)repo`](#git-repo)
  - [Potentially Useful Local Setup](#potentially-useful-local-setup)
- [Working with COOS](#working-with-coos)
  - [Pulling the Source](#pulling-the-source)
  - [Building the Image](#building-the-image)
  - [Licenses and Attribution](#licenses-and-attribution)
    - [Background](#background)
    - [Do the Needful](#do-the-needful)
    - [Creating the Attribution File](#creating-the-attribution-file)
  - [Running the Image with KVM](#running-the-image-with-kvm)
- [Near Future](#near-future)
  - [Things to Track Down](#things-to-track-down)
  - [Things to Test](#things-to-test)
- [Notes](#notes)
- [License](#license)

## Prerequisites

### Dependencies

#### System

Find and install these with your local package manager.

* `git`
* `curl`
* `libvirt` and `qemu`/`kvm`

#### `depot_tools`

You'll need to [install and configure `depot_tools`](http://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html) to work with the Chromium OS source.

I don't like global installs for dev tools; things like `npm install --global` and `sudo pip install` just create dependency hell. Here's how I choose to set things up on Linux (it's similar to [how `pip install --user` works](https://docs.python.org/3/library/site.html#site.USER_SITE) minus the `bin` exports).

```shell
mkdir -p ~/.local/lib && cd $_
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

I use [the `zshenv` file](http://zsh.sourceforge.net/Intro/intro_3.html) for `PATH` changes; drop this where you set yours.

```
echo 'export PATH="$HOME/.local/lib/depot_tools:$PATH"' >> ~/.zshenv
```

Once you've done that and refreshed your env, you can check it by running `gclient`; it will hang initially as it bootstraps itself.

AFAIK, `gclient` requires `python` to be on the `PATH`. If you're running a newer distro that ships with `python3`, not `python` (v2), **don't symlink `python3` to `python`**. Just bite the bullet and install `python` (v2). If you install an app that expects `python2` as `python` but you provide `python3`, you can really break stuff. On older distros, this will break system calls and can totally ruin your installation.

This doesn't work for Windows. You have to use `cmd.exe` which is gnarly. [Follow the docs](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_windows).

#### `(git-)repo`

[The `repo` cli](https://gerrit.googlesource.com/git-repo) is sort of included in `depot_tools` but you essentially reinstall it with every giant mess of related Google repos that constitute any of Chromium VCS (also any Google VCS). Frustratingly, you can't run `repo help <command>` (except for `repo help init`) until you've run a `repo init` somewhere. The `repo` shipped with `depot_tools` is [a version from Android without many features](https://source.android.com/setup/develop/repo). I highly recommend pulling in newer/more featureful versions as you `init` via the `--repo-url` and `--repo-rev` flags ("repo" is overloaded in this context and refers to the tool `repo`, not the VCS repo you're `init`'ing).

[The COOS build page](https://cloud.google.com/container-optimized-os/docs/how-to/building-from-open-source#obtaining_the_source_code) uses [a Chromium fork](https://chromium.googlesource.com/external/repo.git) instead of the Gerrit base. I don't know if there are any breaking API changes but it's simple enough to just use that version out of the three completely distinct versions of the same tool thrown at you in the prereqs to building COOS so just run with it (so much for DRY). I went a step further and used the `stable` ref because it's called "stable."

```shell
repo init \
    --manifest-url <some url> \
    ...
    --repo-url https://chromium.googlesource.com/external/repo.git \
    --repo-ref stable \
    ...
```

### Potentially Useful Local Setup

* [Use `protocol.version=2`](https://git-scm.com/docs/git-config#Documentation/git-config.txt-protocolversion). There doesn't seem to be any issue to set and forget this; it appears to revert gracefully. If the remote can handle it, [it reduces overhead](https://git-scm.com/docs/protocol-v2) and that's always a good thing.

    ```shell
    git config --global protocol.version 2
    ```

* Ensure you've got a `~/.gitconfig` file. [`repo` will potentially write to it](https://bugs.chromium.org/p/gerrit/issues/detail?id=12912&q=component%3Arepo) even if you're using other files. I'm a huge fan of [the XDG Base Dir Spec](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html) and keep my config in `XDG_CONFIG_HOME="$HOME/.config"` to reduce clutter in `$HOME`. Lots of things don't follow the spec, though, so I've got symlinks pointing to my tidy `~/.config` directories.

    ```shell
    ln -sf ~/.config/git/config ~/.gitconfig
    ```

    If you don't have any `git` config at all, you'll need to [set some globals](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_bootstrapping_configuration).

## Working with COOS

### Pulling the Source

1. Create a project directory

    ```shell
    mkdir coos && cd $_
    ```

    From now on, any command presented by itself or following a single `$` should be run from this directory.

    ```shell
    pwd # would output /home/rickjames/code/coos
    ```

    ```shell
    $ pwd
    /home/rickjames/code/coos
    ```

2. Initialize `repo`.

    * Use the `stable` branch of the manifest because it's called "stable"
    * Specify [the `minilayout` group](https://chromium.googlesource.com/chromiumos/manifest.git/+/refs/heads/master#minilayout) for a smaller footprint. There's a chance [it might break things](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md#get-the-source-code) (see the last note in the section); different sources say different things about it.
    * I use `--verbose` for debugging. It dumps a lot so YMMV.

    ```shell
    repo init \
        --manifest-url https://chromium.googlesource.com/chromiumos/manifest.git \
        --manifest-branch stable \
        --groups minilayout \
        --repo-url https://chromium.googlesource.com/external/repo.git \
        --repo-rev stable \
        --verbose
    ```

3. Fetch everything first.

    * `--fail-fast` kills the sync after an error
    * `--no-manifest-update` skips updating the manifest (that we theoretically just downloaded)
    * `--network-only` fetchs remote content but does not apply it. [This tip](https://sites.google.com/a/chromium.org/dev/chromium-os/tips-and-tricks-for-chromium-os-developers#TOC-How-to-make-repo-sync-less-disruptive-by-running-the-stages-separately) makes it easier to debug failed `sync`s.
    * `--clone-bundle` uses `curl` to pull [`git` bundles](https://git-scm.com/book/en/v2/Git-Tools-Bundling) (when provided) instead of making a bunch of remote git calls. AFAIK, this requires a bit more space than the remote calls but these are way easier to debug, seem to run faster, and don't appear to fail anywhere near as much as the remote calls.
    * `--jobs` should be an integer in `[8]` specifying the number of parallel fetches you want to run; higher numbers require bigger network pipes

    ```shell
    repo sync \
        --fail-fast \
        --no-manifest-update \
        --network-only \
        --clone-bundle \
        --jobs 4 \
        --verbose
    ```

    Note that `--network-only` in combinattion with `--clone-bundle` will apply the contents of any bundles instead of just fetching them as we're downloading a file and then reconstructing the `git` changes from it.

4. Apply all the fetched things

    * `--fail-fast` like above
    * `--local-only` applies any remaining fetched changes from things that were not sourced from a bundle
    * `--jobs` should be an integer in `[8]` specifying the number of parallel sets of changes you want to apply; the higher the number the more compute you need

    ```shell
    repo sync \
        --fail-fast \
        --local-only \
        --jobs 1 \
        --verbose
    ```

### Building the Image

1. Use `cros_sdk` to jump into a `chroot`. It helps you with missing system deps and builds the `chroot` on first/second run.

    ```shell
    $ cros_sdk --enter
    [sudo] password for rickjames:
    The tool(s) lvchange, lvcreate, lvs, pvscan, thin_check, vgchange, vgcreate, vgs were not found.
    Please make sure the lvm2 and thin-provisioning-tools packages
    are installed on your host.
    Example(ubuntu):
        sudo apt-get install lvm2 thin-provisioning-tools

    # resove the dep issues

    $ cros_sdk --enter
    08:18:28: NOTICE: Mounted /home/rickjames/code/coos/chroot.img on chroot
    08:18:28: NOTICE: Downloading SDK tarball...
    which08:21:14: NOTICE: Creating chroot. This may take a few minutes...
    ...
    # a few minutes later
    ...
    cros_sdk:make_chroot: All set up.  To enter the chroot, run:
    $ cros_sdk --enter

    CAUTION: Do *NOT* rm -rf the chroot directory; if there are stale bind
    mounts you may end up deleting your source tree too.  To unmount and
    delete the chroot cleanly, use:
    $ cros_sdk --delete

    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $
    ```

2. Determine the board and `export` it. You'll need to find the board name [in the docs](https://cloud.google.com/container-optimized-os/docs/how-to/building-from-open-source). At the moment it's `lakitu`.

    ```shell
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ export BOARD=lakitu
    ```

    This isn't strictly necessary. It's an easy way [to run via copypasta](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md#select-a-board), which is nice. It will go away when you exit the `chroot`, though, so add it to the `.bashrc` for permanence.

    ```shell
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ echo "export BOARD=${BOARD}" >> ~/.bashrc
    ```

3. Set up the `chroot` to build our `BOARD`.

    ```shell
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ setup_board --board ${BOARD} --default
    ```

    The `--default` flag is optional. It writes the provided board name to `src/scripts/.default_board`, which means we don't have to keep specifying the board on every command (yes, that means the previous step was redundant, but now you have multiple ways to screw up the build!).

4. Define the shared user's (`chronos`) password.

    ```shell
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ ./set_shared_user_password.sh
    Enter password for shared user account: Password set in /etc/shared_user_passwd.txt
    ```

    We'll use these credentials later to get into the image, so don't forget this password.

5. Build all the sourced projects inside the image.

    * `--nowithdebug` removes debugging overhead; since we're attempting to pipeline this, we don't need that
    * `--jobs -1` is the default max number of concurrent jobs; you don't need to specify but it's useful to know about if you're experiencing issues with from too many jobs.

    ```shell
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ ./build_packages --nowithdebug --jobs -1
    ```

    **THIS WILL TAKE A VERY LONG TIME.** The docs say vanilla Chromium OS with everything can take [about 90 minutes](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md#build-the-packages-for-your-board). Our `coos` image from the `minilayout` might take a little less; I think my first run was a little over an hour. If you save and reuse the `chroot`, this goes down exponentially because you won't have to rebuild everything every time. I'm not entirely sure how to approach this on cloud build platforms yet. That build time is much longer than most allow for free and the `chroot` artifact can hit many tens of gigs (although it will periodically [run `fstrim`](https://linux.die.net/man/8/fstrim); more on that later).

### Licenses and Attribution

I take FOSS licensing very seriously. For hobby and small business devs, both protecting IP and limiting liability are huge. For major companies, mitigating superfluous litigation and properly attributing FOSS devs to remove licensing costs are good ideas (I think; I'm not a major company).

**Please note that I am not a lawyer and you should not miscontrue this as actual legal advice.**

#### Background

[The Google Cloud `coos` build docs](https://cloud.google.com/container-optimized-os/docs/how-to/building-from-open-source) have the potential to set you up for a world of hurt if you distribute any `coos` artifacts. Deploying artifacts internally without exposing them to them external consumers isn't that grey; in the US corporations are people and most corporations won't hire a developer without requiring them to sign over the rights for everything from the things the dev builds to the thoughts the dev has on the clock.

Exposing an interface to some `coos` artifacts to end users is a bit murkier. Some FOSS licenses explicitly allow things like SaaS without attribution. Others don't. Licenses don't trigger unless the licensed code is actually distributed and, arguably, most cloud artifact deployments do not distribute software.

Actually sharing `coos` artifacts is far from grey; if I send you a completed `coos` image, I have distributed software that contains FOSS licenses and I must provide attribution and possibly even release it under a copyleft license, depending what I include.

In theory, (**I am not a lawyer**), distributing build recipes, such as a Dockerfile, Vagrantfile, or custom build script does not trigger FOSS licenses if no code is included and the end user is the one who actually pulls all the licensed software in by running the recipe. THat's similar to hosting a webpage full of links; unless the link targets explicitly require something in return for targeting them or the links reproduce the external content, the page hasn't actually distributed anything. If webpages full of links were illegal, search engines, Google included, would be illegal.

All that being said, the right and moral thing to do is always attribute as much possible as often as you can when it's a grey area. The legal thing to do is consult a lawyer.

#### Do the Needful

`coos` is a specific instance of Chromium OS, which means it falls under [Chromium OS licensing requirements](https://www.chromium.org/chromium-os/licensing). Until I'm able to grok [the Chromium OS dev licensing process](https://dev.chromium.org/chromium-os/licensing/licensing-for-chromiumos-developers) and incorporate it into any automated process, I'll be sharing recipes and guides like this.

There's [an additional/alternative/totally different process](https://www.chromium.org/chromium-os/licensing/building-a-distro) that can be followed for Chromium OS shippers. This has the potential to drastically change how a `coos` functions, so I need to wrap my head around that too.

**Please note the first process seems to be poorly defined and might not work `coos`. The second process by telling you to not depend on it to fulfill legal obligations.** User beware. But also, like FOSS itself, try to believe that people can be good. If you actively work to fulfill license terms, most FOSS license holders will try to work with you if you screw it up. Just don't be evil.

#### Creating the Attribution File



https://www.chromium.org/chromium-os/licensing/building-a-distro


https://www.chromium.org/chromium-os/licensing

EEE TBD; old



    ```shell
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $
    ```

    ```shell
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $
    ```

    ```shell
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $
    ```


Those `coos` build instructions appear to be outdated, however, so we'll use [the Chromium OS guide](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md#Building-Chromium-OS) with the `lakitu` board.

    ```shell
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ setup_board --board=lakitu

    # should be a fairly short build

    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ ./build_packages --board=lakitu

    # much longer build
    ...
    Builds complete
    11:00:59 INFO    : Elapsed time (build_packages): 73m34s
    Done

    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ ./build_image --board=lakitu dev

    # shorter build
    ...
    11:56:49 INFO    : To convert it to a VM image, use:
    11:56:49 INFO    :   ./image_to_vm.sh --from=../build/images/lakitu/R85-13274.0.2020_06_12_1147-a1 --board=lakitu

    11:56:49 INFO    : Elapsed time (build_image): 9m6s
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ ./image_to_vm.sh --from=../build/images/lakitu/R85-13274.0.2020_06_12_1147-a1 --board=lakitu

    # quick conversion
    ...
    Creating final image
    Created image at /mnt/host/source/src/build/images/lakitu/R85-13274.0.2020_06_12_1147-a1
    You can start the image with:
    cros_vm --start --image-path /mnt/host/source/src/build/images/lakitu/R85-13274.0.2020_06_12_1147-a1/chromiumos_qemu_image.bin

    # exit the chroot so it can release some resources
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ exit
    ```

    At this point, there's a good chance our `chroot.img` is fairly massive. [It's a sparse file](https://groups.google.com/a/chromium.org/d/msg/chromium-os-dev/cc3pp8WzChc/07pKpWuPAwAJ), though, so [re-enter the `chroot` briefly to tidy it](https://chromium.googlesource.com/chromiumos/chromite/+/e3d5bd1ef1a99d5be8fc452f6c617efc33806058/scripts/cros_sdk.py#74).

    ```shell
    $ cros_sdk --enter
    [sudo] password for rickjames:
    12:26:02: NOTICE: /home/rickjames/code/coos/chroot.img is using 21 GiB more than needed.  Running fstrim.
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ exit
    logout
    ```

    If you ran `build_image` multiple times, you might also consider removing builds you're not using, as each build dir takes up several gigs of space and `cros_sdk` does not autoclean them.

    ```shell
    $ find src/build/images/lakitu/ \
        -mindepth 1 \
        -maxdepth 1 \
        -type d \
        -not -name `readlink src/build/images/lakitu/latest` #\
    #   -exec rm -rf {} \;
    # You should verify the find exclues
    # uncomment the line above and the tailing slash above it
    # to automatically nuke anything found

    src/build/images/lakitu/example-old-image-dir
    ```

    ```shell
    ls src/build/images/lakitu -I `readlink src/build/images/latest`
    ls src/build/images/lakitu -I "$( readlink src/build/images/latest )"  -ltr
    find src/build/images/lakitu -type d
    ```

### Running the Image with KVM

The offical docs provide [a snippet to run](https://cloud.google.com/container-optimized-os/docs/how-to/building-from-open-source#running_in_kvm) to verify the image.

```shell
kvm \
    -display none \
    -m 1024 \
    -netdev nic,model=virtio \
    -netdev user,hostfwd=tcp:127.0.0.1:9222-:22 \
    -hda src/build/images/lakitu/latest/
```
## Near Future

### Things to Track Down

* build logs
* times to run
* size on disk at each step

### Things to Test

* ~~[Use the `minilayout` group](https://chromium.googlesource.com/chromiumos/manifest/#minilayout): all the docs say it might not work; test?~~ Seems to work for now
* `repo init --partial-clone` with `repo sync --clone-bundle`: does this work and is it faster?
* `repo sync --current-branch --network-only`: will this break the `--local-only` run?
* `cros_sdk --nouse-image`: size difference?
* `cros_sdk --nousepkg`: size difference?

## Notes

* [The note at the bottom of this section](https://cloud.google.com/container-optimized-os/docs/how-to/building-from-open-source#obtaining_the_source_code) is laughably inaccurate: I'm not entirely sure just how beefy your PC and network pipes would have to be to get the entire source without any slimming in "several minutes." I've spent maybe six hours just trying to get a single working copy down; granted, some of that is due to stops and starts as I test new configurations from scratch. There are more than ten copies of the kernel in the full, each somewhere around `2.5GB` in size. There are still close to 200 other projects to download in addition to those, plus the time it takes `git` to reconstruct the repo from the bundle.

## License


[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

See [the NOTICE file](./NOTICE)
