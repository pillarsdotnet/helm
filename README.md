<!--
Copyright (c) 2026 Robert August Vincent II <pillarsdotnet@gmail.com>
Co-author: Claude Code.
-->

# helm, packaged for Ubuntu

[`Helm`](https://helm.sh) is the package manager for `Kubernetes`. No Ubuntu
or Debian archive carries it, and the vendor apt repository on `baltocdn.com`
serves a certificate chain apt cannot verify, so this repository turns the
upstream release binary into a Debian package for the
[package registry](https://forgejo.bobbalisa.org/pillarsdotnet/-/packages)
on `forgejo.bobbalisa.org`.

There is no source build here: the upstream Linux release is a static Go
binary, and `debian/rules` downloads the release that `debian/changelog`
names from `get.helm.sh`, checks it against the `sha256` recorded in
`debian/upstream-sha256`, and installs it as `/usr/bin/helm`.

## Taking a new upstream release

One commit, two files:

```sh
v=4.3.2   # the new upstream version
curl -fsSL "https://get.helm.sh/helm-v$v-linux-amd64.tar.gz.sha256sum" \
  > debian/upstream-sha256
dch -v "$v-1" -D UNRELEASED "Repack the upstream $v release binary."
```

Push to `main`, and the Registry workflow builds and uploads the package
for every supported series. The series suffix
(`~ppa1~ubuntu<release>.1`) is added at build time by
`.forgejo/registry-upload` and never committed.

This packaging tracks the 3.x line, which upstream still maintains;
`Helm` 4 is current, and moving to it is the same two-file commit,
made deliberately rather than by surprise.

## Licence

`Helm` itself is published under the Apache License 2.0, and the packaging
follows it; see [`debian/copyright`](debian/copyright).
