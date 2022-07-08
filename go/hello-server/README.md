# hello-server

## Build apks with melange

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

Create the repo index using apk and sign it using melange:
```
docker run -it -v $(pwd):/w -w /w/packages -v $(pwd)/packages:/w/packages \
    --entrypoint sh \
    distroless.dev/melange -c \
        'apk index -o APKINDEX.tar.gz **/*.apk && \
            melange sign-index --signing-key=../melange.rsa APKINDEX.tar.gz'
```

To debug the above:
```
docker run -it -v $(pwd):/w -w /w -v $(pwd)/packages:/w/packages --privileged \
    --entrypoint sh \
    distroless.dev/melange

# Build apks (use just --arch x86_64 to isolate issue)
melange build melange.yaml --out-dir /w/packages \
    --arch amd64,aarch64,armv7 \
    --repository-append /w/packages --keyring-append melange.rsa

# Install an apk
apk add ./packages/x86_64/hello-server-*.apk --allow-untrusted --force-broken-world

# Delete an apk
apk del hello-server --force-broken-world
```

## Build image with apko

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

## Sign image with cosign

TODO
