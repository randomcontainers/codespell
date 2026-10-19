# codespell

Container images with [codespell](https://github.com/codespell-project/codespell), which finds common misspellings in source code, documentation and other text files and can fix them in place. It looks each word up in lists of known misspellings, such as `teh` for `the`, so names and terms it does not know pass without complaint. codespell is built into a wheel from its release tag and installed into a virtual environment on Ubuntu or Alpine. The images are rebuilt when codespell publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the codespell project. Report problems with the image in this repository and problems with codespell itself in the [codespell issue tracker](https://github.com/codespell-project/codespell/issues).

## Quick start

Check every file in the current directory and below:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/codespell .
```

Pass the files or directories to check; without arguments the image prints the help. codespell prints one line per misspelling and exits with 65 when it found any, 0 otherwise:

```
./README.md:40: occured ==> occurred
./src/parse.c:88: fo ==> of, for, to, do, go
```

It skips files and directories whose names start with a dot, such as `.git`, unless you pass `--check-hidden`, and it skips binary files. It does not read `.gitignore`, so generated directories such as `node_modules` or `build` are checked unless you skip them (see [Ignoring words and files](#ignoring-words-and-files)). `--check-filenames` also checks the words in each file's path.

`-w` writes the fixes into the files:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/codespell -w .
```

Only misspellings with a single suggestion are fixed. The others, such as `fo` above, are still reported, and the exit code stays 65 until you fix or ignore them. `-i 3 -w` asks before each fix and lets you pick one of several suggestions; it needs a terminal, so add `-it` to `docker run`.

Check text from standard input, without mounting anything:

```sh
printf 'recieve\n' | docker run --rm -i ghcr.io/randomcontainers/codespell -
```

```
1: recieve
	recieve ==> receive
```

The output has colors only when the container has a terminal (`docker run -t`); `--disable-colors` turns them off in that case too.

## Ignoring words and files

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/codespell --skip './vendor,*.min.js' -L inout,ser .
```

- `--skip` (`-S`) takes comma-separated globs. Each is matched against file and directory names and against paths as codespell builds them from the arguments, so `./vendor` matches the top-level `vendor` directory when you check `.`, and `vendor` matches a directory of that name anywhere.
- `-L` (`--ignore-words-list`) takes comma-separated words that are not misspellings in your project. A word in lowercase is accepted in any case; a word with capitals, such as `Wrod`, only in exactly that spelling. `-I FILE` reads such words from a file, one per line.
- A `codespell:ignore` comment, such as `# codespell:ignore` or `// codespell:ignore fo`, accepts the misspellings on its line, or only the words listed after it. `codespell:ignore-next-line` does the same for the next line.
- `--builtin` chooses the dictionaries. The default is `clear,rare`; the others are `informal`, `usage`, `code`, `names`, `en-GB_to_en-US` and `en_to_en-OX`, and `all` selects every one.

## Configuration

codespell reads its options from a `[tool.codespell]` table in `pyproject.toml` and from a `[codespell]` section in `setup.cfg` or `.codespellrc`, in the working directory. In the container that is `/work`, so mount the root of your repository. The keys are the long option names without the dashes:

```toml
[tool.codespell]
skip = "./vendor,*.min.js"
ignore-words-list = "inout,ser"
check-filenames = true
```

Options on the command line replace the values from the file, except `--skip`, `-L`, `-I`, `-D`, `-x` and `--uri-ignore-words-list`, which add to them. With the file above, `--skip ./build` skips `./build` as well as `./vendor` and `*.min.js`.

`pyproject.toml` is read with Python's own `tomllib`, so nothing else needs to be installed. `--toml` and `--config` read the options from another file.

## pre-commit

pre-commit runs a `docker_image` hook with the repository mounted at `/src` as the working directory and, with rootful Docker, as your user, so codespell reads the configuration at the root of the repository:

```yaml
repos:
  - repo: local
    hooks:
      - id: codespell
        name: codespell
        language: docker_image
        entry: ghcr.io/randomcontainers/codespell:2.4.3
        types: [text]
```

Use the newest exact version tag in place of `2.4.3` and update it by hand after each codespell release. Docker reuses an image it has already pulled, so a moving tag such as `latest` would stay on the first version it pulled.

## codespell or typos

[typos](https://github.com/crate-ci/typos), published by this project as `ghcr.io/randomcontainers/typos` (see [randomcontainers/typos](https://github.com/randomcontainers/typos)), is a similar checker that also works from a list of known misspellings. It splits identifiers such as `getRecieveTime` and `max_lenght` into words before looking them up, and it skips the files that `.gitignore` lists. codespell reads each of those identifiers as one word that is not in its lists and lets it pass. Use typos when misspellings inside identifiers matter to you. Use codespell when your project already configures it, for example in `pyproject.toml` or as a pre-commit hook, or for prose, where the optional `informal` and `usage` dictionaries suggest more formal words and recommended terms.

## What is in the image

- `codespell` in `/usr/local/bin`, linked from a virtual environment in `/usr/local/lib/codespell` that uses the distro's `python3`. The virtual environment holds codespell only: codespell has no dependencies, and pip is not installed in it.
- codespell's dictionaries, in the `codespell_lib/data` directory of the virtual environment.

Not included: chardet, which `-e` (`--hard-encoding-detection`) needs; codespell stops with an error when you pass that option. Without it, codespell reads files as UTF-8 and falls back to Latin-1.

## Default or slim

codespell's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a codespell release such as `2.4.3`. `<minor>` and `<major>` are its shorter forms, `2.4` and `2`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

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

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current codespell version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are built natively on GitHub-hosted runners, without emulation. codespell is pure Python, and each of the four builds makes its `py3-none-any` wheel from the same commit.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. codespell only reads files unless you pass `-w`, so without it the mount can be read-only (`-v "$PWD:/work:ro"`). It can read and fix only the files its UID has access to, so run it as your own user:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` follow new codespell releases and are rebuilt when the distro changes. codespell runs from a virtual environment in `/usr/local/lib/codespell`; the `python3` on `PATH` is the distro's interpreter and cannot import codespell. The distro packages codespell needs are listed in `/usr/local/share/randomcontainers/codespell/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/codespell:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends git \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, use `apk add --no-cache git`. The entrypoint is `["tini", "--", "codespell"]`; set your own `ENTRYPOINT` if your image runs a script. To pick up new codespell releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/codespell:latest \
  --repo randomcontainers/codespell --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists the distro and Python packages in it with their versions:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/codespell:latest --format '{{ json .SBOM }}'
```

codespell's files on PyPI have no provenance attestation, so the images do not use them. The build clones the `v<version>` tag of [codespell-project/codespell](https://github.com/codespell-project/codespell), stops unless the tag points to the commit recorded in `package.yml`, and builds the wheel from that checkout with the distro's setuptools and setuptools_scm, without downloading anything else. A tag that is moved after the version was picked up fails the build. The commit is in `/usr/local/share/randomcontainers/codespell/source` and `buildinfo`.

## Updates

The project checks the releases of [codespell-project/codespell](https://github.com/codespell-project/codespell/releases) every 15 minutes and skips pre-releases. A release is picked up once it is 24 hours old. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t codespell:local .
```

Use `Dockerfile.alpine` for the Alpine image. The build stops when `SOURCE_COMMIT` is not the commit of the `v<version>` tag.

## Licenses

codespell is licensed under the [GNU General Public License, version 2](https://github.com/codespell-project/codespell/blob/main/COPYING) (GPL-2.0-only). Its dictionaries are derivative works of English Wikipedia, an improved version of Wikipedia's [list of common misspellings](https://en.wikipedia.org/wiki/Wikipedia:Lists_of_common_misspellings/For_machines), and the codespell project releases them under the [Creative Commons Attribution-ShareAlike 3.0 Unported License](https://creativecommons.org/licenses/by-sa/3.0/) (CC-BY-SA-3.0). The image's license label is `GPL-2.0-only AND CC-BY-SA-3.0`.

The license files are in `/usr/local/share/randomcontainers/codespell/licenses/codespell/`: `COPYING`, codespell's GPL text from the wheel; `dictionaries.NOTICE`, the attribution for the dictionaries; and `CC-BY-SA-3.0`, the license text published by Creative Commons. The wheel does not include the last two, so they come from [`licenses/`](licenses/) in this repository.

The corresponding source for each image:

- codespell: every version has a GitHub release in this repository, named `v<version>`, with `codespell-<version>.tar.gz`, an archive of the commit the wheel was built from. The build applies no patches; it only writes the version into `codespell_lib/_version.py`. codespell is pure Python, so the installed files in `/usr/local/lib/codespell` are its source too. `/usr/local/share/randomcontainers/codespell/source` names the commit and the release.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

Python and the other Ubuntu and Alpine packages in the images keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
