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
    --repository-append /w/packages --keyring-append melange.rsa
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

## Build and push image with apko

TODO

## Sign image with cosign

TODO
