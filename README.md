# A Container-Optimized OS Vagrant Box

WIP

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

## Pulling the Source Code

1. Create a project directory

```shell
mkdir coos && cd $_
```

2. Initialize `repo`.

    * Use the `stable` branch of the manifest
    * Fetch minimal content (`--current-branch`, `--no-tags`, `--partial-clone`) for simplicity

```shell
repo init \
    --manifest-url https://chromium.googlesource.com/chromiumos/manifest.git \
    --manifest-branch stable \
    --current-branch \
    --no-tags \
    --partial-clone \
    --repo-url https://chromium.googlesource.com/external/repo.git \
    --repo-rev stable \
    --verbose

    # --groups minilayout test?
```

3. OPTIONAL BUT REALLY USEFUL FOR SPEED

    The full source includes many versions of the kernel, each several gigs in size. We're only interested in the kernel used for our board (currently `lakitu`).

    Create [the `local_manifests` directory](https://chromium.googlesource.com/external/repo.git/+/refs/heads/stable/docs/internal-fs-layout.md#Manifests) for overrides.
```shell
mkdir -p .repo/local_manifests
```


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

Fetch everything first

```shell
repo sync \
    --fail-fast \
    --network-only \
    --jobs 4 \
    --verbose
```

Apply all the fetched things

```shell
repo sync \
    --fail-fast \
    --network-only \
    --jobs 4 \
    --verbose
```

```shell
curl 'https://chromium.googlesource.com/chromiumos/manifest.git/+/refs/heads/stable/snapshot.xml?format=TEXT' \
    | base64 --decode \
    | sed '/kernel\/v/{ /lakitu/p }' \
    > lakitu.xml
```

```shell
sed '/kernel\/v/{ /lakitu/p }' ~/code/@wotw/test.xml
```
awk
<?xml version="1.0" encoding="UTF-8"?>
<remove-project name="chromiumos/third_party/kernel" />
<manifest>
</manifest>

