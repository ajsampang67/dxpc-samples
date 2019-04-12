# Workspace project layout for DXP Cloud

The project is based on Liferay Workspace, see the Liferay Workspace plugin applied in [settings.gradle](settings.gradle).

## Development Guidelines (local)

We've prepared a `docker-compose` file, which can be used to start the complete Liferay DXP stack locally, in a very similar fashion on how it will be started inside DXP Cloud. You can check the file inside _wedeploy/_ directory: [wedeploy/docker-compose.yml](wedeploy/docker-compose.yml).

Every time you build the services (invoke the `distWeDeploy` task), the compose file will be processed and copied into `build/wedeploy` together with all the built services and configs.

You can either use the file manually, by invoking `docker-compose --file build/wedeploy/docker-compose.yml ...` in you command line, if you are familiar with the syntax of the docker-compose CLI. Or you can use the 3 shortcuts -- Gradle tasks prepared for you.

### Prerequisites

1. Make sure your have Docker server set up locally and ready and that `docker-compose` and `docker` commands are in your PATH. This is what you should see:

    ```
    $ docker version
    Client: Docker Engine - Community
     Version:           18.09.2
     API version:       1.39
     Go version:        go1.10.8
     Git commit:        6247962
     Built:             Sun Feb 10 04:12:39 2019
     OS/Arch:           darwin/amd64
     Experimental:      false
    
    Server: Docker Engine - Community
     Engine:
      Version:          18.09.2
      API version:      1.39 (minimum version 1.12)
      Go version:       go1.10.6
      Git commit:       6247962
      Built:            Sun Feb 10 04:13:06 2019
      OS/Arch:          linux/amd64
      Experimental:     true
    
    $ docker-compose version
    docker-compose version 1.23.2, build 1110ad01
    docker-py version: 3.6.0
    CPython version: 3.6.6
    OpenSSL version: OpenSSL 1.1.0h  27 Mar 2018
  
    ```

    * **NOTE**: Depending on the setup of Docker daemon in your OS, you might need to use `sudo` for these commands. As a result, you also need to instruct Gradle to do it. You can do so by using `-Pliferay.workspace.wedeploy.local.stack.use.sudo=true` for all tasks below which communicate with Docker daemon (`startDxpCloudLocal`, `stopDxpCloudLocal`, `destroyDxpCloudLocal`, `deploy`)
        * This assumes that your OS user executing Gradle has a no-password permission to use `sudo`.
        * You can use `gradle-local.properties` file to set this project property for all your call of the Gradle tasks in this project.
    * Alternatively (and preferably), check Docker docs on how to run without `sudo`, e.g. see the [instructions for Linux](https://docs.docker.com/install/linux/linux-postinstall/).
    
1. Non-Linux only: Make sure that the directory / drive where your project sources are located is allowed to be bind-mounted into Docker containers.
    * For OSX:
      * make sure that the directory itself, or any of its parents, is whitelisted for "File Sharing" in your Docker;
      * please check https://docs.docker.com/docker-for-mac/osxfs/#namespaces for details.
    * For Windows:
      * make sure the drive where your project is located is whitelisted;
      * please check https://docs.docker.com/docker-for-windows/#shared-drives for details.

1. Non-Linux only: Make sure your Docker has enough resources (4CPUs, 8g memory).
    * Because Docker needs a VM with Linux in non-Linux operating system, the VM's sizing determines how much of the host's resources are the containers allowed to use;
    * 4CPUs and 8g memory is the recommended minimum;
    * if possible allocate more - you might get a better experience running the stack locally.

1. Make sure you don't have `gradle-local.properties` present or that it does not modify any of the default values listed in [gradle.properties](gradle.properties).

1. Make sure you have JDK 8 installed and set as the default, since it will be used by Gradle to compile the modules and the `liferay` service uses JDK 8 JVM to run Liferay DXP.

### Starting the stack

`./gradlew startDxpCloudLocal`

* You can use this if you want to start DXP stack with current custom modules built and deployed.
* Performs A start of the docker-compose stack (`docker-compose up ...`). Several Docker containers will be started for you, after Liferay becomes healthy, you can access it (through the nginx webserver) on localhost:8080. Anonymous volumes will be renewed, named volumes will be reused (if they exist).
* `distWeDeploy` will be executed as a dependency task to stay as close to the Jenkins CI build as possible.

**Notes**:

* Depending on the setup of Docker daemon in your OS, you may need to use `./gradlew -Pliferay.workspace.wedeploy.local.stack.use.sudo startDxpCloudLocal` if using `startDxpCloudLocal` gives you errors.  
  * see the `sudo` note in Prerequisites section above for details    
    
* Upon startup, you may experience indexing issues and issues with deploying modules. If this is the case, stop the stack and then restart it. 

* Hot-deploy was set up between the workspace sources and the running `liferay` service. If you change sources of a module and want to redeploy it into your local stack, you have to invoke `deploy` task, see section _Hot-deploy into the stack_ below.

* `SIGTERM` from your shell (e.g. CTRL-C) will most likely **not** stop the stack. You will just be detached from the containers, so you won't see the logs any more, but the containers will still be running (and consuming your computers resources). You can confirm this by doing `docker ps` to list all running processes (containers) in your local Docker server. You can stop the stack by doing `./gradlew stopDxpCloudLocal`, see below.
 
### Stopping the stack 
                
`./gradlew stopDxpCloudLocal`

* You can use this if you want to pause your work, but want to get back to the same DXP data later, possibly with a new version of your modules or even new Docker image for DXP in your stack.
* It performs a stop of the docker-compose stack (`docker-compose stop ...`).
* All named persistent volumes will be retained (see `docker volume ls`) -- so after starting the stack again, the data of Liferay DXP  (database, doclib) will be there from previous start.

**Notes**: 

* For now, Elasticsearch data is on a anonymous volume and will be **not** retained between restarts, see `startDxpCloudLocal` above.
* However, the `liferay` service inside `docker-compose.yml` is configured with environment variable `LIFERAY_INDEX_PERIOD_ON_PERIOD_STARTUP=true`, which is the equivalent of `index.on.startup=true` inside `portal-ext.properties`. So after each startup of DXP, you should have all your data reindexed into `search` service automatically.

### Destroying the stack
 
`./gradlew destroyDxpCloudLocal`
    
* You can use this if you want to stop your work **and** wipe out all the data (database, doclib, search) and start fresh, later.
* Performs a destroy of the docker-compose stack (`docker-compose destroy ...`). If any container is running, it will be stopped first. Any data volume (both named and anonymous) will be removed.

### Hot-deploy into the stack

Assuming your stack is running, you can hot-deploy any module using the standard `deploy` task, from any module's / theme's directory. You can also deploy all the custom modules / themes, by invoking the task in the project's root directory:

```bash
$ ./gradlew deploy

``` 

This will recursively invoke the `deploy` task in any subproject which has this task declared (it's added by the Liferay Workspace plugin applied on the whole project).
 
The build will:
1. detect you are using local dxpcloud stack, 
1. package the updated modules's artifacts (their .jar / .war files) and 
1. send them (using `docker cp`) to the `liferay` service Docker containers. Liferay monitors various directories (`deploy`, `osgi/modules`, `osgi/war`) for changes and redeploys the modules as needed.

You should see the following in the logs of your stack after redeploying a modules:

```
  liferay_1    | 2019-02-26 14:06:59.956 INFO  [fileinstall-/opt/liferay/osgi/modules][BundleStartStopLogger:42] STOPPED com.liferay.acme.navigation.web_1.0.0 [78]
  liferay_1    | 2019-02-26 14:07:00.040 INFO  [Refresh Thread: Equinox Container: 25f3e02c-7d5d-4b53-9036-b50636b69ff0][BundleStartStopLogger:39] STARTED com.liferay.acme.navigation.web_1.0.0 [78]
```

**Notes**: 

* Depending on the setup of Docker daemon in your OS, you may need to use `../../../gradlew -Pliferay.workspace.wedeploy.local.stack.use.sudo deploy` if using `deploy` gives you errors.  
    * See the `sudo` note in Prerequisites section above for details.
    
* The deploy is incremental -- Gradle tasks might be UP-TO-DATE if Gradle did not detect any changes in the deployable artifact (.jar / .war files). Also Liferay won't redeploy given OSGi bundle if the underlying file is exactly the same (modify stamp, file size etc.). If you want to remove any deployable artifacts built before, run `./gradlew clean` and then do a deploy.

* If you want to undeploy an OSGi bundle either delete the file from the container, either:
  * using `docker exec ... bash` (`/opt/liferay` is the Liferay's home) or 
  * using the Gogo shel:l `telnet localhost 11311`.

See below for details.

### OSGi console (telnet client)

The OSGi console in DXP was enabled (on the usual port 11311), the port is exposed outside of the Docker container and it's mapped in the docker-compose.yml to 127.0.0.1:11311 in the host machine. Therefore, you can transparently use telnet locally to go to Gogo shell. You can also most likely simply use `localhost` as the hostname:

```bash
/opt/devel/liferay/github/acme-project $ telnet localhost 11311
Trying ::1...
Connected to localhost.
Escape character is '^]'.
____________________________
Welcome to Apache Felix Gogo

g!

```

**Note:** Use type `disconnect` + `y` (yes) to leave the Gogo shell, **not** exit or quit -- it would terminate the whole OSGi framework in DXP.

### SSH into locally running DXP Cloud service

1. Make sure you have started your local stack (`./gradlew startDxpCloudLocal`)
1. Assuming your Gradle project is named `acme-project` (see [settings.properties](settings.properties)) and you want to access the `liferay` service as if using SSH, you ca do the following:

    ```bash
    docker exec -it dxpcloud_acme-project_liferay_1 bash
    ```

1. Use `docker ps` to list the containers currently running in your local Docker server to find all your project's containers.
     

## Custom modules, themes and wars for Liferay

Standard Liferay Workspace behavior – put everything into *modules/*, *themes/* and *wars/* as usual. All the product will be deployed into Liferay bundle inside 'liferay' service in DXP Cloud.

## DXP Cloud services and their configs -- wedeploy/*/config

Directory [wedeploy/](wedeploy/) contains definition of all your services as deployed into DXP Cloud and their implementation details. It's structured as `wedeploy/<service>`. A each service has it's configs listed in `config` directory, for all available environments.

`wedeploy/liferay/config/common/portal-all.properties` and various `wedeploy/liferay/config/*/portal-env.propeties` were pre-created for you.

**Note**: `[root]/configs/` directory you might have used before is not utilized for DXP Cloud configurations any more. Moreover, the directory cannot be present in the project, otherwise the build will refuse to invoke the task `distWeDeploy`.

## Patching your Liferay before deploy to DXP Cloud

Place your patches (*.zip files) into:
* wedeploy/liferay/hotfix

The base image used for Liferay service (e.g. wedeploy/dxpcloud-liferay-dxp:7.1.0-ga1-fp2-1.1 or newer) has a support to pick up patches when placed into hotfix/ directory next to the wedeploy.json.

**Note**: The patches will be installed before Liferay starts up (on service's startup) which will delay the startup of Liferay. For this reason, do not use this approach to install fixpacks. Instead, use a different image for 'liferay' service with the correct patch pre-installed.

## Build scripts -- gradle/wedeploy/*

All DXP Cloud build scripts are located inside [gradle/wedeploy/](gradle/wedeploy/). You should only rarely have to update these, most likely only if DXP Cloud support instructs you to do so.
