# A Container-Optimized OS Vagrant Box

WIP

## Notes

http://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html

https://cloud.google.com/container-optimized-os/docs/how-to/building-from-open-source

https://cloud.google.com/container-optimized-os

https://source.android.com/setup/develop/repo

```shell
repo init \
    --manifest-url https://chromium.googlesource.com/chromiumos/manifest.git \
    --manifest-branch master \
    --current-branch \
    --partial-clone \
    --repo-url https://chromium.googlesource.com/external/repo.git \
    --verbose

    # --groups minilayout test?
```

```shell
repo sync \
    --fail-fast \
    --current-branch \
    --no-tags \
    --jobs 4 \
    --verbose
```
