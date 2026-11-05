# nbfx-installers: native packages of the nbfx releases

The binary release of [nbfx](https://github.com/gluonhq/nbfx), `nbfx-<version>-bin.zip`, runs on
any of its four platforms with a JDK 25+ the user supplies. This repository wraps that zip with a
JDK into the installer each platform expects, using
[Apache NetBeans nbpackage](https://github.com/apache/netbeans-nbpackage). nbpackage unpacks the
zip, adds the runtime and a launcher that starts `bin/netbeans` (or `bin\netbeans64.exe`) with
`--jdkhome` pointing at the bundled JDK, and hands the result to the platform's packaging tool.

The packages live apart from the nbfx sources so that the signing material the packaging needs
is a secret of this repository alone, and they are released here, under the nbfx tag they
package. They are a convenience: the release of nbfx is the one of `gluonhq/nbfx`
([RELEASING.md](https://github.com/gluonhq/nbfx/blob/main/RELEASING.md)), which this repository
only reads, and the packages are neither signed with the release key nor part of its review.

| Platform | Package | Configuration | Tooling |
|---|---|---|---|
| macOS aarch64 | `nbfx-<version>-arm64.pkg` (an `.app` bundle in a signed, notarized installer) | `macos-aarch64/macos-pkg.properties` | `swift`, `codesign`, `pkgbuild` (Xcode command line tools) |
| Windows x64 | `nbfx-<version>.exe` (Inno Setup) | `windows-x64/windows-innosetup.properties` | [Inno Setup 6](https://jrsoftware.org/isinfo.php) |
| Linux x64 | `nbfx_<version>-1_amd64.deb` | `linux-x64/linux-deb.properties` | `dpkg`, `dpkg-deb`, `fakeroot` |
| Linux x64 | `nbfx-<version>-0.x86_64.rpm` | `linux-x64/linux-rpm.properties` | `rpm`, `rpmbuild` |
| Linux x64 | `nbfx-<version>-x86_64.AppImage` | `linux-x64/linux-appimage.properties` | [appimagetool](https://github.com/AppImage/appimagetool) |

There is no package for macOS x64: jdk.java.net has no JDK 27 for it (the port is deprecated
since JDK 27 and Oracle no longer builds it), so that platform keeps using the zip with a JDK of
the user's own.

Each configuration is the output of `nbpackage --type <type> --save-config`, filled in; the keys
are documented in place. What they share: the name `nbfx`, the publisher Gluon, the bundle and
application id `com.gluonhq.nbfx`, and the runtime of [`runtimes.properties`](runtimes.properties).

## The bundled runtime

[`runtimes.properties`](runtimes.properties) pins one JDK per platform by URL and SHA-256, plus
appimagetool: JDK 27 GA from [jdk.java.net](https://jdk.java.net/27/). Each configuration expects
its file at `<platform>/downloads/<file>` (git-ignored); the workflow puts it there, a local build
downloads it by hand. To move to another JDK, change the pin and the `package.runtime` file name
of the platform's configuration; nbpackage reads the architecture from the file name, so keep the
vendor's name.

## The workflow

[`packages.yml`](.github/workflows/packages.yml) builds every package on GitHub runners:

1. **The zip**: `nbfx-<version>-bin.zip` of the `gluonhq/nbfx` release named by `tag` is
   downloaded with the workflow's own token and checked against its `.sha512`. The release must
   be published (a pre-release is fine); a draft is not visible.
2. **nbpackage** is built from source (`mvnw package`) at a pinned commit of
   `apache/netbeans-nbpackage` (`NBPACKAGE_REF`), once, on Linux: it is pure Java, so one build
   serves every platform.
3. **One job per (platform, package type)** downloads the pinned runtime, runs nbpackage with the
   platform's configuration (`-Pversion=<version>` from the zip's name) and uploads the package as
   a workflow artifact named `nbfx-<tag>-<platform>-<type>`.
4. **Collect** merges them into one artifact, `nbfx-<tag>-packages`, with a `.sha256` list, and
   publishes where asked.

It runs by hand (`Actions → Packages → Run workflow`, or from anywhere
`gh workflow run packages.yml --repo gluonhq/nbfx-installers -f tag=0.1.0-rc1 -f publish=release`),
or through a `repository_dispatch` event of type `nbfx-release` with `{"tag": ..., "publish": ...}`
as the payload, which another workflow can send with a token that may write to this repository.
`publish` is:

- `nowhere`: the workflow artifacts only (the default);
- `snapshot`: the rolling `snapshot` pre-release of **this** repository, recreated on the packaged
  commit, for trying the packages and the signing without touching an nbfx release;
- `release`: the release of **this** repository named `tag`, created on the packaged commit (a
  pre-release when the tag has a pre-release part, as in `0.1.0-rc1`); when it exists already,
  its files of the same name are replaced. Nothing is uploaded to `gluonhq/nbfx`, so no token for
  that repository is needed.
