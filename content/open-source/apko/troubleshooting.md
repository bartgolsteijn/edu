---
title: "Troubleshooting apko Builds"
type: "article"
lead: "Debugging and common errors"
date: 2022-08-10T11:07:52+02:00
lastmod: 2022-08-10T11:07:52+02:00
contributors: ["Erika Heidi"]
draft: false
images: []
menu:
  docs:
    parent: "apko"
weight: 300
toc: true
---

## Debug Options

To include debug-level information on apko builds, add `--debug` to your build command:

```shell
docker run --rm -v ${PWD}:/work distroless.dev/apko build --debug \
  apko.yaml hello-minicli:test hello-minicli.tar \
  -k melange.rsa.pub
```
## Common Errors

>Error: failed to build layer image: initializing apk: failed to fixate apk world: exit status 1

This error typically means that one of the requested apk packages could not be resolved or the apk index is missing.

### The requested package is not in the included repositories
For Alpine packages, make sure you've added the relevant package repositories you need and the package name is correct - search the [Alpine APK index](https://pkgs.alpinelinux.org/packages) for reference.

If this is your case, you should find error messages similar to this when enabling debug info with the `--debug` flag:

> ERROR: unable to select packages:
>  hello-minicli (no such package)

### apko is unable to find the local packages folder
With melange-built package(s), make sure you have a volume sharing your apko / melange files with the location `/work` inside the apko container.

### The apk index is missing
If you have a functional volume sharing your packages with the apko container and you're still getting this error, make sure you built a valid apk index as described in [step 4 of the Getting Started with melange Guide](/open-source/melange/getting-started-with-melange/#step-4--building-your-apk).

If this is your case, you should find error messages similar to this when enabling debug info with the `--debug` flag:

> ERROR: Not committing changes due to missing repository tags. Use --force-broken-world to override.

This is how your packages directory tree should look like, including the `APKINDEX.tgz` file for each architecture:

```
packages
├── aarch64
│   ├── APKINDEX.tar.gz
│   └── hello-minicli-0.1.0-r0.apk
├── armv7
│   ├── APKINDEX.tar.gz
│   └── hello-minicli-0.1.0-r0.apk
├── x86
│   ├── APKINDEX.tar.gz
│   └── hello-minicli-0.1.0-r0.apk
└── x86_64
    ├── APKINDEX.tar.gz
    └── hello-minicli-0.1.0-r0.apk

4 directories, 8 files
```