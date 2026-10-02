# mupdf

Container images with `mutool`, the command-line tool of [MuPDF](https://mupdf.com/), for rendering, converting, merging, encrypting and repairing PDF files and for converting EPUB, XPS, CBZ, HTML and Markdown documents. MuPDF is compiled from the release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when Artifex publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by Artifex Software, which develops MuPDF. Report problems with the image in this repository and MuPDF bugs at [bugs.ghostscript.com](https://bugs.ghostscript.com/), under the MuPDF product.

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/mupdf \
  draw -r 150 -o page-%d.png input.pdf
```

Extract the text of a PDF:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/mupdf \
  draw -F txt -o output.txt input.pdf
```

The entrypoint runs `mutool` under `tini` in `/work`, so file names are relative to the directory you mount. Without arguments the image prints the MuPDF version, and `--help` lists the commands. A few more:

```sh
# Merge PDFs
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/mupdf \
  merge -o merged.pdf first.pdf second.pdf

# Rewrite a PDF with unused objects removed and its streams compressed
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/mupdf \
  clean -gggz input.pdf output.pdf

# Encrypt a PDF with AES-256
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/mupdf \
  clean -E aes-256 -U user-password -O owner-password input.pdf locked.pdf

# Convert an EPUB book to PDF, and a PDF to Word
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/mupdf \
  convert -o book.pdf book.epub
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/mupdf \
  convert -o output.docx input.pdf

# Check the digital signatures in a PDF
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/mupdf \
  sign -v signed.pdf
```

The [mutool documentation](https://mupdf.readthedocs.io/en/latest/tools/mutool.html) covers every command and option.

## What is in the image

- `mutool` in `/usr/local/bin`, with its commands: `draw`, `convert`, `clean`, `merge`, `info`, `pages`, `show`, `extract`, `create`, `sign`, `trim`, `poster`, `recolor`, `bake`, `grep`, `audit`, `trace` and `run`, which runs JavaScript against the MuPDF API.
- The libraries bundled with the MuPDF release, compiled into `mutool` as upstream builds it: FreeType, HarfBuzz, libjpeg, OpenJPEG, jbig2dec, lcms2mt (Artifex's thread-safe fork of Little CMS), MuJS, Gumbo, cmark-gfm, extract (Word output), Brotli and zlib. OpenSSL comes from the distro and is used for digital signatures.
- The fonts built into `mutool`: URW's versions of the 14 standard PDF fonts (from the URW base 35 set), Noto fonts for most other scripts, Source Han Serif and Droid Sans Fallback for Chinese, Japanese and Korean, and Charis SIL for EPUB and HTML. MuPDF does not use system fonts.

Not included: `muraster`, the viewers (`mupdf-gl` and `mupdf-x11`), OCR with Tesseract, barcode support, the hyphenation patterns used when laying out EPUB and HTML, and the MuPDF library and headers. The make flags are in `/usr/local/share/randomcontainers/mupdf/buildinfo`.

## Default or slim

MuPDF's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a MuPDF release such as `1.28.5`. `<minor>` and `<major>` are its shorter forms, `1.28` and `1`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current MuPDF version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

`mutool` writes temporary files to `/tmp` for some inputs, so a container started with `--read-only` also needs `--tmpfs /tmp`.

## Untrusted files

MuPDF parses PDF, XPS, EPUB and image formats in C, and its patch releases regularly fix memory errors and hangs caused by malformed files. The `mutool` commands do not run JavaScript embedded in a PDF; only a `mutool run` script that calls `enableJS()` turns it on. For files from unknown sources, take away what the container does not need:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --network none --read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges \
  --memory 1g --pids-limit 64 \
  ghcr.io/randomcontainers/mupdf draw -F txt -o output.txt untrusted.pdf
```

Mount only the directory the job needs. The libraries MuPDF bundles, such as FreeType and OpenJPEG, are compiled into `mutool`, so their fixes reach the image only with MuPDF releases. The images pick up a new release about a day after Artifex publishes it.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new MuPDF release and are rebuilt when the base image changes. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/mupdf:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends qpdf \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache qpdf`. The entrypoint is `["tini", "--", "mutool"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new MuPDF releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/mupdf/` holds the version, the source URL, the build options, the license files and `runtime-deps`, the list of distro packages MuPDF needs at run time. The list is empty when the base image already has them.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/mupdf:latest \
  --repo randomcontainers/mupdf --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/mupdf:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml`.

## Updates

The project checks the releases of [ArtifexSoftware/mupdf-downloads](https://github.com/ArtifexSoftware/mupdf-downloads/releases) every 15 minutes and skips release candidates. A release is picked up once it is 24 hours old. Artifex publishes no checksum file or signature for MuPDF, so the tarball is checked against the SHA-256 digest GitHub records for the release asset. The new version and the tarball's SHA-256 are then committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t mupdf:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

MuPDF is licensed under the GNU Affero General Public License, version 3 or later (AGPL-3.0-or-later). jbig2dec and extract, which also come from Artifex, are under the same license. Other parts have their own licenses:

- Apache-2.0: Gumbo and the Droid Sans Fallback font (`COPYING.gumbo`, `DroidSansFallback.NOTICE`).
- BSD-2-Clause: OpenJPEG and cmark-gfm (`LICENSE.openjpeg`, `COPYING.cmark-gfm`).
- BSD-3-Clause: the AES code from XySSL and the Adobe CMap files (`COPYING.crypt-aes`, `COPYING.cmaps`).
- FTL: FreeType (`FTL.TXT`).
- IJG: libjpeg from the Independent JPEG Group (`README.libjpeg`).
- ISC: MuJS and the UCDN Unicode database code (`COPYING.mujs`, `COPYING.ucdn`).
- MIT: lcms2mt, Brotli, parts of cmark-gfm, the Microsoft USE tables in HarfBuzz, the hash table code in FreeType, the Grisu number formatting code, `strverscmp` from musl, the UTF-8 decoder in Gumbo and the Open Iconic and Jam annotation icons (`LICENSE.lcms2mt`, `LICENSE.brotli`, `COPYING.harfbuzz-ms-use`, `COPYING.fthash`, `COPYING.ftoa`, `COPYING.strverscmp`, `COPYING.gumbo-utf8`, `COPYING.annotation-icons`).
- MIT-Modern-Variant: HarfBuzz (`COPYING.harfbuzz`).
- OFL-1.1: the URW standard PDF fonts and the Noto, Source Han Serif and Charis SIL fonts (`OFL.urw-base35`, `OFL.noto`, `OFL.source-han-serif`, `OFL.charis-sil`).
- Unicode-TOU: the reference code of the Unicode bidirectional algorithm (`COPYING.bidi-imp`).
- X11: the ICC profile header from SunSoft (`COPYING.icc34`).
- Zlib: zlib (`LICENSE.zlib`).
- Permissive notices that have no SPDX identifier: the ARC4 code by Kalle Kaukonen, the public domain MD5 code by Alexander Peslyak and the softSurfer line geometry code in lcms2mt (`COPYING.crypt-arc4`, `COPYING.crypt-md5`, `COPYING.lcms2mt-cmssm`).

Everything in the list is compiled into `mutool`. The files named above are in `/usr/local/share/randomcontainers/mupdf/licenses/`, with MuPDF's `COPYING` and jbig2dec's `LICENSE.jbig2dec`. The image's license label is `AGPL-3.0-or-later AND Apache-2.0 AND BSD-2-Clause AND BSD-3-Clause AND FTL AND IJG AND ISC AND MIT AND MIT-Modern-Variant AND OFL-1.1 AND Unicode-TOU AND X11 AND Zlib`. The hyphenation patterns in the tarball, which come under many licenses, are not built. The Ubuntu and Alpine packages in the image keep their own licenses.

The corresponding source for each image:

- MuPDF: every version has a GitHub release in this repository, named `v<version>`, with the exact `mupdf-<version>-source.tar.gz` that was compiled, bundled libraries included. The build applies no patches. The download URL is in `/usr/local/share/randomcontainers/mupdf/source`.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every make flag.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

Combined images that include MuPDF copy it from a slim MuPDF image from this repository. Their `com.randomcontainers.members` label records the MuPDF version and the digest of that image, whose own `org.opencontainers.image.revision` label names the commit here.

If you redistribute these images, or offer a modified MuPDF to users over a network, read what the AGPL requires of you. Artifex also sells [commercial licenses](https://artifex.com/licensing/) for use under other terms.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
