# hello-melange-apko 💫

[![go](https://github.com/chainguard-dev/hello-melange-apko/actions/workflows/go.yml/badge.svg)](https://github.com/chainguard-dev/hello-melange-apko/actions/workflows/go.yml)
[![js](https://github.com/chainguard-dev/hello-melange-apko/actions/workflows/js.yml/badge.svg)](https://github.com/chainguard-dev/hello-melange-apko/actions/workflows/js.yml)
[![py](https://github.com/chainguard-dev/hello-melange-apko/actions/workflows/py.yml/badge.svg)](https://github.com/chainguard-dev/hello-melange-apko/actions/workflows/py.yml)
[![ruby](https://github.com/chainguard-dev/hello-melange-apko/actions/workflows/ruby.yml/badge.svg)](https://github.com/chainguard-dev/hello-melange-apko/actions/workflows/ruby.yml)
[![rust](https://github.com/chainguard-dev/hello-melange-apko/actions/workflows/rust.yml/badge.svg)](https://github.com/chainguard-dev/hello-melange-apko/actions/workflows/rust.yml)

This repo contains an  example app duplicated across 5 languages showing how to:

- Package source code into APKs using [`melange`](https://github.com/chainguard-dev/melange)
- Build and publish OCI images from APKs using [`apko`](https://github.com/chainguard-dev/apko)

The app itself is a basic HTTP server that returns "Hello World!"

```
$ curl -s http://localhost:8080
Hello World!
```

Wondering what "APKs" are? They're OS packages with a `.apk` extension (similar to `.rpm` / `.deb`) that are compatible with [`apk`](https://wiki.alpinelinux.org/wiki/Package_management).

## Variations

| Language   | Repo Path          | GitHub Action                                                  | Notes                                                     |
|------------|------------------- | -------------------------------------------------------------- | --------------------------------------------------------- |
| Go         | [`go/`](./go/)     | [`go.yml`](./.github/workflows/go.yml)       | uses gin                                                  |
| JavaScript | [`js/`](./js/)     | [`js.yml`](./.github/workflows/js.yml)       | uses express, vendors node_modules, depends on nodejs |
| Python     | [`py/`](./py/)     | [`py.yml`](./.github/workflows/py.yml)       | uses flask, vendors virtualenv, depends on python3    |
| Ruby       | [`ruby/`](./ruby/) | [`ruby.yml`](./.github/workflows/ruby.yml)   | uses sinatra, vendors bundle, depends on ruby         |
| Rust       | [`rust/`](./rust/) | [`rust.yml`](./.github/workflows/rust.yml)   | uses hyper, currently builds very slow cross-platform     |

Note: third-party server frameworks are used intentionally
to validate the use of dependencies.

## "The hard way"

This section shows how to run through each of the build stages locally and
pushing an image to GHCR.

Requirements:

- [`docker`](https://docs.docker.com/get-docker/)
- [`cosign`](https://docs.sigstore.dev/cosign/installation/)

Note: these steps should also work without `docker` on an apk-based Linux distribution such as [Alpine](https://www.alpinelinux.org/).

### Change directory

All of the following steps in this section assume that
from the root of this repository, you have changed directory
to one of the variations:

```
cd go/   # for Go
cd js/   # for JavaScript
cd py/   # for Python
cd ruby/ # for Ruby
cd rust/ # for Rust
```

### Build apks with melange

Make sure the `packages/` directory is removed:
```
rm -rf ./packages/
```

Create a temporary melange keypair:
```
docker run --rm -v "${PWD}":/work --entrypoint=melange --workdir=/work ghcr.io/wolfi-dev/sdk keygen
```

Build an apk for all architectures using melange:
```
docker run --rm --privileged -v "${PWD}":/work  \
    --entrypoint=melange --workdir=/work \
    cgr.dev/chainguard/sdk build melange.yaml \
    --arch amd64,aarch64,armv7 \
    --signing-key melange.rsa
```

To debug the above:
```
docker run --rm --privileged -it -v "${PWD}":/work \
    --entrypoint sh \
    cgr.dev/chainguard/sdk

# Build apks (use just --arch amd64 to isolate issue)
melange build melange.yaml \
    --arch amd64,aarch64,armv7 \
    --signing-key melange.rsa

# Install an apk
apk add ./packages/x86_64/hello-server-*.apk --allow-untrusted --force-broken-world

# Delete an apk
apk del hello-server --force-broken-world
```

### Build image with apko

*Note: you could skip this step and go to "Push image with apko".*

Build an apk for all architectures using melange:
```
# Your GitHub username
GITHUB_USERNAME="myuser"
REF="ghcr.io/${GITHUB_USERNAME}/hello-melange-apko/$(basename "${PWD}")"

docker run --rm -v "${PWD}":/work \
    --entrypoint=apko --workdir=/work ghcr.io/wolfi-dev/sdk build --debug apko.yaml \
    "${REF}" output.tar -k melange.rsa.pub \
    --arch amd64,aarch64,armv7
```

If you do not wish to push the image, you could load it directly:
```
ARCH_REF="$(docker load < output.tar | grep "Loaded image" | sed 's/^Loaded image: //' | head -1)"
docker run --rm --rm -p 8080:8080  "${ARCH_REF}"
```

Note: The output of `docker load` will print all architectures. The command above just picks the first one.
You could also choose to run `docker load < output.tar` and manually copy the architecture that matches your system.

To debug the above:
```
docker run --rm -it -v "${PWD}":/work \
    -e REF="${REF}" \
    --entrypoint sh \
    --workdir=/work ghcr.io/wolfi-dev/sdk

# Build image (use just --arch amd64 to isolate issue)
apko build --debug apko.yaml "${REF}" output.tar -k melange.rsa.pub --arch amd64,aarch64,armv7
```

## Push image with apko

Build and push an image to, for example, GHCR:
```
# Your GitHub username
GITHUB_USERNAME="myuser"
REF="ghcr.io/${GITHUB_USERNAME}/hello-melange-apko/$(basename "${PWD}")"

# A personal access token with the "write:packages" scope
GITHUB_TOKEN="*****"

docker run --rm -v "${PWD}":/work \
    -e REF="${REF}" \
    -e GITHUB_USERNAME="${GITHUB_USERNAME}" \
    -e GITHUB_TOKEN="${GITHUB_TOKEN}" \
    --entrypoint sh \
    --workdir=/work ghcr.io/wolfi-dev/sdk -c \
        'echo "${GITHUB_TOKEN}" | \
            apko login ghcr.io -u "${GITHUB_USERNAME}" --password-stdin && \
            apko publish --debug apko.yaml \
                "${REF}" -k melange.rsa.pub \
                --arch amd64,aarch64,armv7'
```

## Sign image with cosign

After the image has been published, sign it recursively using cosign (2.0+):

```
# Your GitHub username
GITHUB_USERNAME="myuser"
REF="ghcr.io/${GITHUB_USERNAME}/hello-melange-apko/$(basename "${PWD}")"

cosign sign -r -y "${REF}"
```

This should use "keyless" mode and open a browser window for you to
authenticate.

Note: prior to running above, you may need to re-login to GHCR
on the host using docker (or other tool):

```
# Your GitHub username
GITHUB_USERNAME="myuser"

# A personal access token with the "write:packages" scope
GITHUB_TOKEN="*****"

echo "${GITHUB_TOKEN}" | docker login ghcr.io -u "${GITHUB_USERNAME}" --password-stdin
```

## Verify the signature

Verify that the image is signed using cosign:

```
# Your GitHub username
GITHUB_USERNAME="myuser"
REF="ghcr.io/${GITHUB_USERNAME}/hello-melange-apko/$(basename "${PWD}")"

cosign verify "${REF}" --certificate-identity-regexp=.* --certificate-oidc-issuer-regexp=.*
```

## Run the hello server image

Finally, run the image using docker:

```
# Your GitHub username
GITHUB_USERNAME="myuser"
REF="ghcr.io/${GITHUB_USERNAME}/hello-melange-apko/$(basename "${PWD}")"

docker run --rm --rm -p 8080:8080 "${REF}"
```

Then in another terminal, try hitting the server using curl:

```
curl -s http://localhost:8080
```

```
Hello World!
```
