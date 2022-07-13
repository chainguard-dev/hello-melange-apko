# hello-melange-apko

This repo contains an  example app duplicated across 5 languages showing how to:

- Package source code into apks using [`melange`](https://github.com/chainguard-dev/melange)
- Build and publish OCI images using [`apko`](https://github.com/chainguard-dev/apko)

The app itself is a basic HTTP server that returns "Hello World!"

```
$ curl -s http://localhost:8080
Hello World!
```

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

Note: these steps should also work without `docker` on an [`apk`](https://docs.alpinelinux.org/user-handbook/0.1a/Working/apk.html)-based Linux distribution such as [Alpine](https://www.alpinelinux.org/), [Adélie](https://www.adelielinux.org/), etc.

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
docker run -it -v $(pwd):/w -w /w distroless.dev/melange keygen
```

Build an apk for all architectures using melange:
```
docker run -it -v $(pwd):/w -w /w -v $(pwd)/packages:/w/packages --privileged \
    distroless.dev/melange build melange.yaml --out-dir /w/packages \
    --arch amd64,aarch64,armv7 \
    --repository-append /w/packages --keyring-append melange.rsa
```

To debug the above:
```
docker run -it -v $(pwd):/w -w /w -v $(pwd)/packages:/w/packages --privileged \
    --entrypoint sh \
    distroless.dev/melange

# Build apks (use just --arch amd64 to isolate issue)
melange build melange.yaml --out-dir /w/packages \
    --arch amd64,aarch64,armv7 \
    --repository-append /w/packages --keyring-append melange.rsa

# Install an apk
apk add ./packages/x86_64/hello-server-*.apk --allow-untrusted --force-broken-world

# Delete an apk
apk del hello-server --force-broken-world
```

### Generate apk repo indexes

Create the repo indexes using apk and sign them using melange:
```
docker run -it -v $(pwd):/w -w /w/packages -v $(pwd)/packages:/w/packages \
    --entrypoint sh \
    distroless.dev/melange -c \
        'for d in `find . -type d -mindepth 1`; do \
            ( \
                cd $d && \
                apk index -o APKINDEX.tar.gz *.apk && \
                melange sign-index --signing-key=../../melange.rsa APKINDEX.tar.gz\
            ) \
        done'
```

### Build image with apko

*Note: you could skip this step and go to "Push image with apko".*

Build an apk for all architectures using melange:
```
# Your GitHub username
GITHUB_USERNAME="myuser"
REF="ghcr.io/${GITHUB_USERNAME}/melange-demo-app/$(basename "${PWD}")"

docker run -it -v $(pwd):/github/workspace -w /github/workspace \
    -v $(pwd)/packages:/github/workspace/packages \
    distroless.dev/apko build --debug apko.yaml \
    "${REF}" output.tar -k melange.rsa.pub \
    --build-arch amd64,aarch64,armv7
```

To debug the above:
```
docker run -it -v $(pwd):/github/workspace -w /github/workspace \
    -v $(pwd)/packages:/github/workspace/packages \
    -e REF="${REF}" \
    --entrypoint sh \
    distroless.dev/apko

# Build image (use just --build-arch amd64 to isolate issue)
apko build --debug apko.yaml "${REF}" output.tar -k melange.rsa.pub --build-arch amd64,aarch64,armv7
```

## Push image with apko

Build and push an image to, for example, GHCR:
```
# Your GitHub username
GITHUB_USERNAME="myuser"
REF="ghcr.io/${GITHUB_USERNAME}/melange-demo-app/$(basename "${PWD}")"

# A personal access token with the "write:packages" scope
GITHUB_TOKEN="*****"

docker run -it -v $(pwd):/github/workspace -w /github/workspace \
    -v $(pwd)/packages:/github/workspace/packages \
    -e REF="${REF}" \
    -e GITHUB_USERNAME="${GITHUB_USERNAME}" \
    -e GITHUB_TOKEN="${GITHUB_TOKEN}" \
    --entrypoint sh \
    distroless.dev/apko -c \
        'echo "${GITHUB_TOKEN}" | \
            apko login ghcr.io -u "${GITHUB_USERNAME}" --password-stdin && \
            apko publish --debug apko.yaml \
                "${REF}" -k melange.rsa.pub \
                --arch amd64,aarch64,armv7'
```

## Sign image with cosign

After the image has been published, sign it using cosign:

```
# Your GitHub username
GITHUB_USERNAME="myuser"
REF="ghcr.io/${GITHUB_USERNAME}/melange-demo-app/$(basename "${PWD}")"

COSIGN_EXPERIMENTAL=1 cosign sign -y -f "${REF}"
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
REF="ghcr.io/${GITHUB_USERNAME}/melange-demo-app/$(basename "${PWD}")"

COSIGN_EXPERIMENTAL=1 cosign verify "${REF}"
```

## Run the hello server image

Finally, run the image using docker:

```
# Your GitHub username
GITHUB_USERNAME="myuser"
REF="ghcr.io/${GITHUB_USERNAME}/melange-demo-app/$(basename "${PWD}")"

docker run -it --rm -p 8080:8080 "${REF}"
```

Then in another terminal, try hitting the server using curl:

```
curl -s http://localhost:8080
```

```
Hello World!
```
