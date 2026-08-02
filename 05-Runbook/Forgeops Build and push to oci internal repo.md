
Default forgeops use docker to build images, so we can use forgeops export to get current config and edit then build images

## First login with oc

```shell
oc login -u username -p password
```

### Export current config idm

```shell
#Export current config
./bin/forgeops export idm --release-name pnguat pnguat-profile --sort
## this will create pnguat-profile in following location
ls docker/idm/config-profiles
```
### Export current config idm

```shell
./bin/forgeops config export am --release-name pnguat oc-profile --sort --no-upgrade
## this will create pnguat-profile in following location
ls docker/am/config-profile
```
After editing config files, lets build image

>Default it uses docker to build images
```shell
./bin/forgeops build idm -e pnguat -p pnguat-profile -t idm-custom-v1
```
now image will start to build with repo and tag in the `Dockerfile` in `docker/idm/Dockerfile`

Change the tag in `Dockerfile`

```Dockerfile
ARG REPO=<change if you want>
ARG TAG=8.1.1
```

now images are running in docker images

### Push images with docker
```shell
#login docker with oc login
docker login $(oc registry info) -u $(oc whoami) -p $(oc whoami -t)
docker tag idm:idm-custom-v1 $(oc registry info)/pngprd-images/idm:idm-custom-v1
docker push $(oc registry info)/pngprd-images/idm:idm-custom-v1
```


### Push images with podman
```shell
#login with podman
podman login --tls-verify=false $(oc registry info) -u $(oc whoami) -p $(oc whoami -t)
podman tag idm:idm-custom-v1 $(oc registry info)/pngprd-images/idm:idm-custom-v1
podman push $(oc registry info)/pngprd-images/idm:idm-custom-v1
```


### Delete old images

```shell
#use image id
docker rmi <image id> or porman rmi <image id>
#force
docker rmi <image id> -f or podman rmi <image id> -f
```


## OC Build

#### Command 1: Initialize the Template (Run Once)

This creates the structural objects inside OpenShift. It sets up the target ImageStream and tells the cluster to expect a binary payload.

Bash

```
oc new-build --binary=true --strategy=docker --name=idm-binary-build --to=idm:custom-test
```

#### Command 2: Execute and Follow the Build (Run every time you make changes)

This zips up your local directory, pushes it straight into the cluster's build engine, injects your custom configuration profile, and streams the engine logs live to your terminal screen.

Bash

```
oc start-build idm-binary-build --from-dir=./idm-oc --build-arg CONFIG_PROFILE=oc-profile --follow
```

### if that is not working

set env variable first

#### Step 1: Inject the profile into your Build Configuration

Run this command to tell OpenShift's Docker engine strategy to permanently map `CONFIG_PROFILE` to `oc-profile`:

Bash

```
oc set env bc/idm-binary-build CONFIG_PROFILE=oc-profile
```

_(If OpenShift throws a flag warning that it expects a `build-arg` pattern instead of an environment variable for Docker strategy builds, you can update the underlying template directly via `oc patch`:)_

Bash

```
oc patch bc/idm-binary-build -p '{"spec":{"strategy":{"dockerStrategy":{"buildArgs":[{"name":"CONFIG_PROFILE","value":"oc-profile"}]}}}}'
```

#### Step 2: Trigger the Build normally

Now that the blueprint explicitly holds the argument, you don't need to specify any arguments on the command line. Run your clean stream:

Bash

```
oc start-build idm-binary-build --from-dir=./idm-oc --follow
```

#### Cancel build

```shell
oc cancel-build bc/idm-binary-build
```


#### To see all the builds
```shell
oc get bc
```
### check updated images

```shell
oc debug istag/idm:custom-test-oc -n pnguat
```

#### Delete a image

```shell
oc delete istag idm:custom-test-oc -n pnguat
```
