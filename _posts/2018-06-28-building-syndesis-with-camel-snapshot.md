---
layout: post
title: Building Syndesis platform with Apache Camel snapshot
---

In the last months I worked on [Syndesis](https://syndesis.io/) project. Syndesis is an hybrid integration platform based on Apache Camel. During this time I had the need to build this platform against a Camel Snapshot version to test some new features I added into the Apache Camel project and it wasn't truly easy. Adding the possibility to build the platform against different Camel snapshots and versions can be very useful to test Camel master and also to have an idea of how new/updated Camel components behave in this platform. I think it would be useful for end users too.

### Normal Workflow

Building Syndesis platform is not super easy at first sight, but it's very well documented and complete.

```bash
> syndesis/tools/bin$ ./syndesis build --help
Run Syndesis builds

Usage: syndesis build [... options ...]

Options for build:
-b  --backend                 Build only backend modules (core, extension, integration, connectors, server, meta)
    --images                  Build only modules with Docker images (ui, server, meta, s2i)
-m  --module <m1>,<m2>, ..    Build modules
                              Modules: ui, server, connector, s2i, meta, integration, extension, common
-d  --dependencies            Build also all project the specified module depends on
    --skip-tests              Skip unit and system test execution
    --skip-checks             Disable all checks
-f  --flash                   Skip checks and tests execution (fastest mode)
-i  --image-mode  <mode>      <mode> can be
                              - "none"      : No images are build (default)
                              - "openshift" : Build for OpenShift image streams
                              - "docker"    : Build against a plain Docker daemon
                              - "auto"      : Automatically detect whether to use
                                              "openshift" or "docker"
    --docker                  == --image-mode docker
    --openshift               == --image-mode openshift
-p  --project <project>       Specifies the project to create images in when using '--openshift'
-k  --kill-pods               Kill pods after the image has been created.
                              Useful when building with image-mode docker
-c  --clean                   Run clean builds (mvn clean)
    --batch-mode              Run mvn in batch mode
    --camel-snapshot          Run a build with a specific Camel snapshot. You'll need to set an environment variable CAMEL_SNAPSHOT_VERSION with the SNAPSHOT version you want to use.
    --man                     Open HTML documentation in the Syndesis Developer Handbook
```

I use [Minishift](https://github.com/minishift/minishift) to play with Syndesis and my normal workflow is the following:
First I spin up a Minishift instance

```bash
> minishift start --memory 8384
```

You could need to specify a vm-driver too with the --vm-driver flag.

Then I set the docker environment coming from Minishift

```bash
> eval $(minishift docker-env)
```

At this point I'm able to build

```bash
> syndesis/tools/bin$ ./syndesis build --openshift --clean
```

Once the build it's done (it may take a while) we are able to deploy Syndesis platform on Minishift

```bash
> syndesis/tools/bin$ ./syndesis minishift --install --openshift
```

Once the deployment is finished we are able to start using Syndesis platform.

### Building with a Camel Snapshot

The option you'll need in this case will be --camel-snapshot, in combination with an environment variable called `CAMEL_SNAPSHOT_VERSION`. 
In my case I need to test a new feature in a component from Camel 2.21.2-SNAPSHOT. The workflow to obtain a running Syndesis instance based on Camel 2.21.2-SNAPSHOT is the following (supposing you have a running Minishift). I built Camel 2.21.2-SNAPSHOT locally before following these steps.

Set the docker environment coming from Minishift

```bash
> eval $(minishift docker-env)
```

Export `CAMEL_SNAPSHOT_VERSION` environment variable

```bash
> export CAMEL_SNAPSHOT_VERSION="2.21.2-SNAPSHOT"
```

Run a Syndesis build with --camel-snapshot flag enabled

```bash
> syndesis/tools/bin$ ./syndesis build --openshift --clean --camel-snapshot
```

Once the build finished, you can deploy your Syndesis on Minishift

```bash
> syndesis/tools/bin$ ./syndesis minishift --install --openshift
```

This is all you need to test a Syndesis platform based on Camel snapshot.

### Conclusion

In the future I'll blog about the Syndesis platform a bit more. If you want to contribute you can start from the [Github project](https://github.com/syndesisio/syndesis/), the [site](https://syndesis.io/) or an [extension](https://github.com/syndesisio/syndesis-extensions). 





