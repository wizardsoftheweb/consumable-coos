# failed file pls ignore

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)  [![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/) [![Check the NOTICE](https://img.shields.io/badge/Check%20the-NOTICE-420C3B.svg)](./NOTICE)

## Potentially Useful Local Setup

* [Increasing `http.postBuffer`](https://stackoverflow.com/a/6849424). Read the analysis; YMMV. Might not be necessary when using `repo sync --clone-bundle`.

    ```shell
    git config --global http.postBuffer 524288000
    ```

## Pulling the Source Code

1. Create a project directory

    ```shell
    mkdir coos && cd $_
    ```

2. Initialize `repo`.

    * Use the `stable` branch of the manifest
    * Set up to fetch minimal content (`--current-branch`, `--no-tags`, `--partial-clone`) for simplicity

    ```shell
    repo init \
        --manifest-url https://chromium.googlesource.com/chromiumos/manifest.git \
        --manifest-branch stable \
        --current-branch \
        --no-tags \
        --repo-url https://chromium.googlesource.com/external/repo.git \
        --repo-rev stable \
        --verbose

        # --groups minilayout test?
        # --partial-clone issues?
    ```

3. ~~OPTIONAL BUT REALLY USEFUL FOR SPEED~~

    This causes problems during the `./build_packages` step apparently.

    The full source includes many versions of the kernel, each several gigs in size. We're only interested in the kernel used for our board (currently `lakitu`).

    Create [the `local_manifests` directory](https://chromium.googlesource.com/external/repo.git/+/refs/heads/stable/docs/internal-fs-layout.md#Manifests) for overrides.

    ```shell
    mkdir -p .repo/local_manifests
    ```

    Create a `lakitu` manifest.

        * Removes all kernels not related to `lakitu`

    ```shell
    awk '\
        BEGIN { \
            print "<?xml version=\"1.0\" encoding=\"UTF-8\"?>"; \
            print "<manifest>"; \
            print " <remove-project name=\"chromiumos/third_party/kernel\" />"; \
        } \
        /kernel\/v[^"]*lakitu/{ \
            print $0; \
        } \
        END { \
            print "</manifest>"; \
        }' .repo/manifests/default.xml \
        > .repo/local_manifests/lakitu.xml
    ```
4. Fetch everything first

    ```shell
    repo sync \
        --fail-fast \
        --no-manifest-update \
        --network-only \
        --clone-bundle \
        --jobs 1 \
        --verbose
    ```

    There's a chance this will fail; still working on that. Use `--jobs 1` for debugging completely failed syncs.

    ~~This doesn't seem to exit on some failures -- not sure what's going on there. I'm not sure how to diagnose the issue without completely abandoning parallel jobs. It seems like some of the issue is with `--retry-fetches`; however, I haven't figured out how to tell what breaks where even with `--fail-fast --jobs 1 --verbose`.~~ Removing [the `--partial-clone` flag](https://git-scm.com/docs/partial-clone) from `repo init` seems to have resolved this although it's pulling more in terms of total file size (I think).

    At this point, the directory is about this big:

    ```shell
    $ du -sh .repo
    11G    .repo
    ```

5. Apply all the fetched things

    ```shell
    repo sync \
        --fail-fast \
        --local-only \
        --jobs 1 \
        --verbose
    ```

    Now the directory is bigger.

    ```shell
    $ du -sh
    19G	   .
    ```

6. Use `cros_sdk` to jump into a `chroot`. It helps you with missing system deps and builds the `chroot` on first/second run.

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

7. Build the image. As mentioned before, you'll need to find the board name [in the docs](https://cloud.google.com/container-optimized-os/docs/how-to/building-from-open-source). At the moment it's `lakitu`.

    ```shell
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ setup_board --board=lakitu
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $ ./build_packages --board=lakitu
    (cr) (stable/(fe780ea...)) rickjames@couch ~/trunk/src/scripts $
    ```
