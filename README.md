# Argo CD image build speedup test
Disclaimer: Do not use any of these files in production!!! All files in this repository have been copied over from github.com/argoproj/argo-cd for testing purposes except for the .github.workflows directory. This repository will most likely be removed within 30 days.

## The purpose for this repository is testing a solution to speedup the build proccess of images for Argo CD.
Images being built on arm64 are taking an excessive amount of time due to the emulation of the architecture they are being built on.  In particular the argocd-ui stage which performs `yarn install --network-timeout 200000` & `RUN HOST_ARCH='amd64' NODE_ENV='production' NODE_ONLINE_ENV='online' NODE_OPTIONS=--max_old_space_size=8192 yarn build`

## Solution steps
### Changes were made to image.yaml
1. Setup a cache for argo-ui docker layer
2. Build the cache during push events, cache **cannot** be built during pull request for secuity purposes.  More on that can be found at 
[Restrictions for accessing a cache](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache)
3. Seperate existing build proccess into 3 steps to provide more flexibilty
- Ensure docker builds for pull request only on `linux/amd64` does not push to registry
- Build and push the latest image for push events on `linux/amd64,linux/arm64` pushes to registry
- Test-arm-image used with `test-arm-image` label to build on `linux/amd64,linux/arm64` does not push to registry
4. Clean up build cache if new cache was built

## Results
| Workflow | Image # | Event | Cache Built | Total Time | Cache Build Time | Image Time | ARCH | Cache |
| ---  |--- | --- | --- | --- | --- | --- | --- | -- |
| Image | 2 | PR | no | 6min 48s | n/a | 6min 38s | amd64 | no cache |
| Image | 3 | PR/Label | yes | 1hr 18m | 39m 49s | 36min 57s | amd64/arm64 | cache cannot be built during PR
| Image | 4 | Push/Merge | yes | 1hr 18m | 38m 44s | 36min 57s | amd64/arm64 | cache built and saved
| Image | 5 | PR | no | 6min 5s | n/a | 4min 39s | amd64 | cache used
| Image | 6 | Push/Merge| no | 34min 39s | n/a | 32min 7s | amd64/arm64 | cache used

## When will cache be rebuilt?
1. When base image changes
2. `ui/package.json` or `ui/yarn.lock` files are modified
3. when any files are changed in `ui/`

i.e. #3 above changes. Any instructions from Line 41 and below will be ran.  Any Instructions above Line 41 will still be used in the cache.

```DOCKERFILE
FROM docker.io/library/node:12.18.4 as argocd-ui

WORKDIR /src
ADD ["ui/package.json", "ui/yarn.lock", "./"]

RUN yarn install --network-timeout 200000

ADD ["ui/", "."]

ARG ARGO_VERSION=latest
ENV ARGO_VERSION=$ARGO_VERSION
RUN HOST_ARCH='amd64' NODE_ENV='production' NODE_ONLINE_ENV='online' NODE_OPTIONS=--max_old_space_size=8192 yarn build
```

## Steps to replicate on local dev environment
Be sure to have buildx running for multi architecture support
https://docs.docker.com/desktop/multi-arch/

### Build cache for argocd-ui stage
```
docker buildx build --platform linux/arm64,linux/amd64 --target=argocd-ui --push=false --cache-to=type=local,dest=/tmp/.buildx-cache-new,mode=max .
```
### Build argocd-ui stage from cache
```
docker buildx build --platform linux/arm64,linux/amd64 --target=argocd-ui --push=false --cache-from=type=local,src=/tmp/.buildx-cache-new .
```
### Build comlete DOCKERFILE
```
docker buildx build --platform linux/arm64,linux/amd64 --push=false --cache-from=type=local,src=/tmp/.buildx-cache-new . 
```
