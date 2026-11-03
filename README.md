# nbfx-installers: native packages of the nbfx releases

The binary release of [nbfx](https://github.com/gluonhq/nbfx), `nbfx-<version>-bin.zip`, runs on
any of its four platforms with a JDK 25+ the user supplies. This repository wraps that zip with a
JDK into the installer each platform expects, using
[Apache NetBeans nbpackage](https://github.com/apache/netbeans-nbpackage), the tool Apache NetBeans
and the Codelerity installers are packaged with. The zip itself is not changed: nbpackage unpacks
it, adds the runtime and a launcher that starts `bin/netbeans` (or `bin\netbeans64.exe`) with
`--jdkhome` pointing at the bundled JDK, and hands the result to the platform's packaging tool.

The packages live apart from the nbfx sources so that the signing material and the tokens the
packaging needs are secrets of this repository alone. They are a convenience: the release of nbfx
is the one of `gluonhq/nbfx` ([RELEASING.md](https://github.com/gluonhq/nbfx/blob/main/RELEASING.md)),
and the packages are neither signed with the release key nor part of its review.

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

A `.tar.gz` runtime is repacked with `tar` after the checksum, under the same name, before nbpackage
sees it. The jdk.java.net Linux tarball has no directory entries, and nbpackage (at the pinned
commit) creates the parent directory of regular file entries only, so the first symlink of an
all-symlink directory, `legal/java.compiler/ADDITIONAL_LICENSE_INFO`, fails to extract with
`NoSuchFileException`; `tar` writes the directory entries the extraction needs. The macOS tarball
has them already. A local Linux build needs the same repack (below).

## The workflow

[`packages.yml`](.github/workflows/packages.yml) builds every package on GitHub runners:

1. **The zip**: `nbfx-<version>-bin.zip` of the `gluonhq/nbfx` release named by `tag` is
   downloaded and checked against its `.sha512`. A published release or pre-release is enough;
   a draft needs `NBFX_RELEASE_TOKEN` (below).
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
- `release`: uploaded to the `gluonhq/nbfx` release of `tag`, replacing files of the same name.
  This is the step of [RELEASING.md](https://github.com/gluonhq/nbfx/blob/main/RELEASING.md) that
  attaches the packages to a published release candidate; promoting the candidate copies them to
  the final release, the same bytes, so a final release is not packaged again.

### Secrets

Signing happens when the repository secrets are set; without them every package is unsigned (a
fork). macOS uses the same secrets as `gluonhq/scenebuilder`:

| Secret | Use |
|---|---|
| `CERTIFICATES_FILE_BASE64`, `CERTIFICATES_PASSWORD` | the `.p12` with the Developer ID Application and Installer certificates, imported into the runner's keychain |
| `MACSIGN_USER` | the name in the identities: `Developer ID Application: <MACSIGN_USER>` signs the binaries and the bundle, `Developer ID Installer: <MACSIGN_USER>` the `.pkg` |
| `NOTARIZATION_USERNAME`, `NOTARIZATION_PASSWORD`, `NOTARIZATION_TEAMID` | the Apple ID, app-specific password and team for `notarytool`; the ticket is stapled to the `.pkg` |
| `NBFX_RELEASE_TOKEN` | a fine-grained personal access token with *Contents: read and write* on `gluonhq/nbfx`, used to upload the packages to its release (`publish=release`) and to read a release that is still a draft; the workflow's own token reads published releases without it |

nbpackage signs every native binary (the JavaFX `.dylib`s, the JDK's, the ones inside jars that
carry some) with the hardened runtime and its sandbox entitlements, then the bundle, then
`pkgbuild` signs the installer. The Windows installer and the Linux packages are not signed.

## A local build

```sh
# nbpackage: the 1.0 release, or `mvnw package` of a checkout (target/nbpackage-*-bin.zip)
curl -LO https://archive.apache.org/dist/netbeans/netbeans-nbpackage/1.0/nbpackage-1.0-bin.zip
unzip nbpackage-1.0-bin.zip

# the runtime of the platform, where its configuration expects it (runtimes.properties)
mkdir -p macos-aarch64/downloads
curl -o macos-aarch64/downloads/openjdk-27.0.1.0.3_macos-aarch64_bin.tar.gz <url>
shasum -a 256 macos-aarch64/downloads/*          # compare with the pin
# Linux only: repack the tarball so it carries directory entries (see "The bundled runtime")
# mkdir x && tar xzf <file> -C x && tar czf <file> -C x . && rm -r x

# the zip of a release, or the one `ant release-bin` makes in an nbfx checkout
gh release download 0.1.0 --repo gluonhq/nbfx --pattern 'nbfx-*-bin.zip' --dir build
mkdir -p dist
nbpackage-1.0/bin/nbpackage --verbose \
    --config macos-aarch64/macos-pkg.properties \
    --input build/nbfx-0.1.0-bin.zip --output dist -Pversion=0.1.0
```

`-P<key>=<value>` overrides any `package.` key, e.g. `-Pmacos.codesign-id="Developer ID
Application: ..." -Pmacos.pkgbuild-id="Developer ID Installer: ..."` to sign, or
`-Pinnosetup.tool=<path to ISCC.exe>`. On Linux without FUSE, run the AppImage build with
`APPIMAGE_EXTRACT_AND_RUN=1`. `--image-only` stops after the unpacked, laid-out image, to inspect
what the package will contain; a full run leaves that image directory next to the package (the
workflow deletes it before uploading).

## Open points

- **Icons.** Every configuration leaves the icon keys empty, so the packages show nbpackage's
  default, the Apache NetBeans logo, on purpose for now: an nbfx icon derived from it needs the
  NetBeans project's view on the trademark first. When one exists, it goes in `icons/` (an
  `.icns`, an `.ico`, a 48x48 `.png` and an `.svg`) and into the `package.*.icon` keys.
- **The Linux `StartupWMClass`** (`nbfx`) is unverified: check `xprop WM_CLASS` on the running
  window; a mismatch only affects dock grouping.
- **The Debian/RPM maintainer address** is a placeholder.
- **Windows signing** (Authenticode) is not set up.
- **The tarball repack** goes away once nbpackage creates parent directories for link entries too
  (a two-line fix in `ArchiveUtils`, to be proposed upstream) and `NBPACKAGE_REF` moves past it.
- **Triggering from nbfx.** Publishing a release candidate in `gluonhq/nbfx` could send the
  `nbfx-release` dispatch to this repository; that needs a token able to write here, stored in
  `gluonhq/nbfx`. Until then the workflow is run by hand after the candidate is published.
