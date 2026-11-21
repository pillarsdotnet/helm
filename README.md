<!--
Copyright (c) 2026 Robert August Vincent II <pillarsdotnet@gmail.com>
Co-author: Claude Code.
-->

# helm, packaged for Ubuntu

[`Helm`](https://helm.sh) is the package manager for `Kubernetes`. No Ubuntu
or Debian archive carries it, and the vendor apt repository on `baltocdn.com`
serves a certificate chain apt cannot verify, so this repository builds it
into a Debian package for the
[package registry](https://forgejo.bobbalisa.org/pillarsdotnet/-/packages)
on `forgejo.bobbalisa.org`.

`debian/rules` clones the release tag that `debian/changelog` names, checks
that `HEAD` is the commit in `debian/upstream-commit`, bumps the modules
listed in `GO_MODULE_PINS`, builds with the Go toolchain `GOTOOLCHAIN`
names, and installs the result as `/usr/bin/helm`.

## Why it is not a repack

Until 4.3.2 this package downloaded the upstream Linux release binary and
repacked it, which is less code and less to go wrong. It was given up
because a release binary cannot be remediated.

A static Go binary carries the toolchain and the module versions whoever
built it happened to have. `trivy` reads both out of the binary, so the
advisory count against a released `helm` only ever goes up between
releases, and a repack inherits whatever upstream shipped:

| binary | findings |
| --- | --- |
| 4.3.1 as released | 5 |
| 4.3.2 as released | 1 |
| 4.3.2 built here | 0 |

The one left in 4.3.2 as released is `GO-2026-5932`, which says
`golang.org/x/crypto/openpgp` is unmaintained and unsafe by design.
Upstream builds against `x/crypto` 0.58.0, which still ships that package;
0.59.0 removed it. Nothing upstream does to the release binary changes
that, and waiting for upstream to rebuild leaves the finding installed on
every host meanwhile.

So the three pins in `debian/rules` are the package: the upstream tag, the
Go toolchain, and the modules whose released versions carry advisories.
Scan after every bump -- a new toolchain or a new upstream release can add
one as easily as clear one.

## Taking a new upstream release

Two files, and then a scan:

```sh
v=4.3.3   # the new upstream version
git ls-remote https://github.com/helm/helm.git "refs/tags/v$v^{}"
# put that commit in debian/upstream-commit
dch -v "$v-1" -D UNRELEASED "Build the upstream $v release."
dpkg-buildpackage -b -us -uc
trivy rootfs --scanners vuln debian/helm/usr/bin
```

A finding at that point is a module or toolchain pin to raise, not a reason
to stop: raise it in `debian/rules`, rebuild, scan again. Check
<https://go.dev/dl/> for the current Go patch release at the same time --
most of what a Go binary ever scans with lives in the standard library.

Push to `main`, and the Registry workflow builds and uploads the package
for every supported series. The series suffix
(`~ppa1~ubuntu<release>.1`) is added at build time by
`.forgejo/registry-upload` and never committed.

This packaging tracks the 4.x line. Upstream maintains 3.x in parallel, so
`helm` 3 charts and scripts are worth a look before a host takes this.

## Licence

`Helm` itself is published under the Apache License 2.0, and the packaging
follows it; see [`debian/copyright`](debian/copyright).
