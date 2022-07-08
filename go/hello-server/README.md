# hello-server

## Build apks with melange

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

## Generate apk repo indexes

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

## Build image with apko

*Note: you could skip this step and go to "Push image with apko".*

Build an apk for all architectures using melange:
```
REF="ghcr.io/chainguard-dev/melange-apko-cosign-examples/go/hello-server"

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
REF="ghcr.io/chainguard-dev/melange-apko-cosign-examples/go/hello-server"

# Your GitHub username
GITHUB_USERNAME="myuser"

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
REF="ghcr.io/chainguard-dev/melange-apko-cosign-examples/go/hello-server"

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
REF="ghcr.io/chainguard-dev/melange-apko-cosign-examples/go/hello-server"

COSIGN_EXPERIMENTAL=1 cosign verify "${REF}"
```

## Run the hello server image

Finally, run the image using docker:

```
REF="ghcr.io/chainguard-dev/melange-apko-cosign-examples/go/hello-server"

docker run -it --rm -p 8080:8080 "${REF}"
```

Then in another terminal, try hitting the server using curl:

```
curl -s http://localhost:8080
```

```
Hello World!
```