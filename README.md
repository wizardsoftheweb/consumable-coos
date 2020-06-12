# A Container-Optimized OS Vagrant Box

WIP

I want to locally play with [Google's `coos`](https://cloud.google.com/container-optimized-os) in both `vagrant` and containers but they don't publish an image ([AWS does](https://hub.docker.com/_/amazonlinux) and [Azure builds on top of Ubuntu](https://github.com/Azure/AKS/tree/master/vhd-notes/aks-ubuntu); Google builds their own OS from scratch).

## Notes

http://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html

https://cloud.google.com/container-optimized-os/docs/how-to/building-from-open-source

https://cloud.google.com/container-optimized-os

https://source.android.com/setup/develop/repo

## Prerequisites

http://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html

https://gerrit.googlesource.com/git-repo

https://source.android.com/setup/develop/repo

## Necessary Info

https://cloud.google.com/container-optimized-os/docs/how-to/building-from-open-source

Find the board name. Currently `lakitu`.

## Potentially Useful Local Setup

* [Using `protocol.version=2`](https://git-scm.com/docs/git-config#Documentation/git-config.txt-protocolversion). There doesn't seem to be any issue to set and forget this; it appears to revert gracefully.

    ```shell
    git config --global protocol.version 2
    ```

## Pulling the Source Code

1. Create a project directory

    ```shell
    mkdir coos && cd $_
    ```

2. Initialize `repo`.

    * Use the `stable` branch of the manifest
    * Specify the `minilayout` group for a smaller footprint

    ```shell
    repo init \
        --manifest-url https://chromium.googlesource.com/chromiumos/manifest.git \
        --manifest-branch stable \
        --groups minilayout \
        --repo-url https://chromium.googlesource.com/external/repo.git \
        --repo-rev stable \
        --verbose

        # --groups minilayout test?
        # --partial-clone issues?
    ```



## Things to Test

* [Use the `minilayout` group](https://chromium.googlesource.com/chromiumos/manifest/#minilayout): all the docs say it might not work; test?
* `repo init --partial-clone` with `repo sync --clone-bundle`: does this work and is it faster?
* `repo sync --current-branch --network-only`: will this break the `--local-only` run?

## Laughable Inaccuracies

* [The note at the bottom of this section](https://cloud.google.com/container-optimized-os/docs/how-to/building-from-open-source#obtaining_the_source_code): I'm not entirely sure just how beefy your PC and network pipes would have to be to get the entire source without any slimming in "several minutes." I've spent maybe six hours just trying to get a single working copy down; granted, some of that is due to stops and starts as I test new configurations from scratch. There are more than ten copies of the kernel in the full, each somewhere around `2.5GB` in size. There are still close to 200 other projects to download in addition to those, plus the time it takes `git` to reconstruct the repo from the bundle.
