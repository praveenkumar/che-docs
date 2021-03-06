---
title: "Configuration&#58 Docker"
keywords: docker, configuration
tags: [installation, docker]
sidebar: user_sidebar
permalink: docker-config.html
folder: setup-docker
---

{% include links.html %}

Configuration is handled by modifying [che.env](https://github.com/eclipse/che/blob/master/dockerfiles/init/manifests/che.env) placed in the root of a host folder volume mounted to `:/data`. This configuration file is generated during the `che init` phase. If you rerun `che init` in an already initialized folder, the process will abort unless you pass `--force`, `--pull`, or `--reinit`.

You can also pass an environment variable directly in docker run syntax: `-e CHE_ENV_NAME=value`.

Each variable is documented with an explanation and usually commented out. If you need to set a variable, uncomment it and configure it with your value. You can then run `che config` to apply this configuration to your system. `che start` also reapplies the latest configuration.

You can run `che init` to install a new configuration into an empty directory. This command uses the `che/init:<version>` Docker container to deliver a version-specific set of puppet templates into the folder.

If you run `che config`, che runs puppet to transform your puppet templates into a che instance configuration, placing the results into `/che/instance` if you volume mounted that, or into a `instance` subdirectory of the path you mounted to `/che`.  Each time you start che, `che config` is run to ensure instance configuration files are properly generated and consistent with the configuration you have specified in `che.env`.

Administration teams that want to version control your che configuration should save `che.env`. This is the only file that should be saved with version control. It is not necessary, and even discouraged, to save the other files. If strategy were to perform a `che upgrade` we may replace these files with templates that are specific to the version that is being upgraded. The `che.env` file maintains fidelity between versions and we can generate instance configurations from that.

The version control sequence would be:

1. `che init` to get an initial configuration for a particular version.
2. Edit `che.env` with your environment-specific configuration.
3. Save `che.env` to version control.
4. Setup a new folder and copy `che.env` from version control into the folder you will mount to `:/data`.
5. Run `che config` or `che start`.

## Single Port Policy

By default, Che lets Docker publish exposed ports in a random manner - Docker chooses available ports from the ephemeral port range to expose workspace [servers][servers]. This, however, brings in certain network requirements, namely opening the ephemeral port range (and Keycloak port 5050 for multi user Che) to the world.

To run Che in a single port mode add `-e CHE_SINGLE_PORT=true` to your run syntax. In this case, a Traefik container will be used to route traffic through a single port.

**Wildcard DNS**

In a single port mode, Che builds URLs of workspace services using the following pattern - `serviceName-machineName-ws-ID.IP.wildcardDNSProvider`. So, if your external IP is `193.12.34.56`, URL of a workspace agent will look like `wsagent-http-dev-machine-workspace0bcgkgkvsqi31b4u.193.12.34.56.nip.io`

* by default [nip.io](http://nip.io/) is used. This is an external wildcard DNS provider, and if it is down for some reason, networking in single port Che is broken.
* you can use a different wildcard DNS provider with `CHE_SINGLEPORT_WILDCARD__DOMAIN_HOST` env.
* if you don't want the external IP to be part of the url (`serviceName-machineName-ws-ID.wildcardDNSProvider`, for example to use a wildcard SSL certificate), you can specify `CHE_SINGLEPORT_WILDCARD__DOMAIN_IPLESS=true` with a custom wildcard DNS (e.g. `CHE_SINGLEPORT_WILDCARD__DOMAIN_HOST=domain.tld`. Be aware that you need to have a matching DNS entry for `*.domain.tld`. If you are using the multi-user mode, Keycloak will be provided at `keycloak.domain.tld`.

**Multi-User Mode**

Make sure `webOrigins` and `redirectUris` in Keycloak client settings (`che-public` client) reference your `CHE_DOCKER_IP_EXTERNAL` value, i.e IP that external users will use to log in. Keycloak admin console in multi user Che is available at `http://keycloak.$IP.$wildcardDNSProvider:$chePort/auth/` where `$IP` is either your `docker0` IP or `CHE_DOCKER_IP_EXTERNAL` value.

## Logs and User Data

When Che initializes itself, it stores logs, user data, database data, and instance-specific configuration in the folder mounted to `:/data/instance` or an `instance` subfolder of what you mounted to `:/data`.  

Che's containers save their logs in the same location:

```shell
/instance/logs/che/2016                 # Server logs
/instance/logs/che/che-machine-logs     # Workspace logs
```

User data is stored in:

```shell
/instance/data/che                      # Project backups (we synchronize projects from remote ws here)
```

Instance configuration is generated by Che and is updated by our internal configuration utilities. These 'generated' configuration files should not be modified and stored in:

```shell
/instance/che.ver.do_not_modify         # Version of che installed
/instance/docker-compose-container.yml  # Docker compose to launch Che from within a container
/instance/docker-compose.yml            # Docker compose to launch Che from the host without container
/instance/config                        # Configuration files for Che which are volume mounted into containers
```

## JDBC Configuration
Eclipse Che uses [H2](http://www.h2database.com/html/main.html) for single-user builds and PostgreSQL database in a multi-user flavor.

Depending on the used database, JDBC Connection pool will be initialized with respective default values.
These values can be overridden through che.env.

```
# Example of configuration for using H2 database (default for single-user Che)
CHE_JDBC_USERNAME=
CHE_JDBC_PASSWORD=
CHE_JDBC_DATABASE=jdbc:h2:che
CHE_JDBC_URL=jdbc:postgresql://postgres:5432/dbche
CHE_JDBC_DRIVER__CLASS__NAME=org.h2.Driver
CHE_JDBC_MAX__TOTAL=8
CHE_JDBC_MAX__IDLE=2
CHE_JDBC_MAX__WAIT__MILLIS=-1
```

```
# Example of configuration for using PostgreSQL database (default for multi-user Che)
CHE_JDBC_USERNAME=pgche
CHE_JDBC_PASSWORD=pgchepassword
CHE_JDBC_DATABASE=dbche
CHE_JDBC_URL=jdbc:postgresql://postgres:5432/dbche
CHE_JDBC_DRIVER__CLASS__NAME=org.postgresql.Driver
CHE_JDBC_MAX__TOTAL=20
CHE_JDBC_MAX__IDLE=10
CHE_JDBC_MAX__WAIT__MILLIS=-1
```

**Logback configuration**

By default ws-master and ws-agent are configured to use logback as default logging backend. With has such configuration
```
<configuration>
    <contextListener class="ch.qos.logback.classic.jul.LevelChangePropagator">
        <resetJUL>true</resetJUL>
    </contextListener>
    <contextListener class="org.eclipse.che.commons.logback.EnvironmentVariablesLogLevelPropagator"/>

    <property name="max.retention.days" value="60" />

    <jmxConfigurator/>

    <appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%-41(%date[%.15thread]) %-45([%-5level] [%.30logger{30} %L]) - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <append>true</append>
        <prudent>true</prudent>
        <encoder>
            <charset>utf-8</charset>
            <pattern>%-41(%date[%.15thread]) %-45([%-5level] [%.30logger{30} %L]) - %msg%n</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${che.logs.dir}/archive/%d{yyyy/MM/dd}/catalina.log</fileNamePattern>
            <maxHistory>${max.retention.days}</maxHistory>
        </rollingPolicy>
    </appender>


    <include optional="true" file="${che.local.conf.dir}/logback/logback-additional-appenders.xml"/>
    <include optional="true" file="${catalina.home}/conf/logback-additional-appenders.xml"/>

    <logger name="org.apache.catalina.loader" level="OFF"/>
    <logger name="org.apache.catalina.session.PersistentManagerBase" level="OFF"/>
    <logger name="org.apache.jasper.servlet.TldScanner" level="OFF"/>

    <root level="${che.logs.level:-INFO}">
        <appender-ref ref="stdout"/>
        <appender-ref ref="file"/>
    </root>
</configuration>
```
Usually, it stored in tomcat's conf folder as logback.xml file.
There are two ways to exted this configuration.
1. Put your configuration in ${che.local.conf.dir}/logback/logback-additional-appenders.xml or ${catalina.home}/conf/logback-additional-appenders.xml file
2. Provide environment variable in such form 
```
CHE_LOGGER_CONFIG=logger1=logger1_level,logger2=logger2_level
```
for example
```
CHE_LOGGER_CONFIG=org.eclipse.che=DEBUG,org.eclipse.che.api.installer.server.impl.LocalInstallerRegistry=OFF 
```
In case if you are using docker cli you can put this variable to che.env for ws-master or workspace config in case of workspace agent.


**Logstash JSON Encoder**

Che-server comes with `logstash-json-encoder` which allows you to send logs in a JSON format for Logstash or any other log consumers. Adding a new appender can be done by adding a `logback-additional-appenders.xml` in `${CHE_LOCAL_CONF_DIR}/logback/` or `${CATALINA_HOME}/conf/` folder from your custom assembly or mounted volume:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<included>
    <appender name="file-json" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${che.logs.dir}/logs/catalina.log.json</file>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
          <level>info</level>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${che.logs.dir}/archive/%d{yyyy/MM/dd}/catalina.log.json</fileNamePattern>
            <maxHistory>${max.retention.days}</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
    </appender>

    <root level="${che.logs.level:-INFO}">
        <appender-ref ref="file-json"/>
    </root>
</included>
```

This appender will log in a JSON format to a rolling `/logs/logs/catalina.log.json` file.

Other `logback-logstash-encoder` appenders can be added as well. See <https://github.com/logstash/logstash-logback-encoder#usage>.

## oAuth

You can configure Google, GitHub, Microsoft or BitBucket oAuth for use when users perform git operations. See: [Version Control](version-control.html#github-oauth)

## Stacks and Samples

[Stacks][stacks] define the recipes used to create workspace runtimes. They appear in the stack library of the dashboard. You can create your own.

`CHE_PREDEFINED_STACKS_RELOAD__ON__START` (false by default) defines stack loading policy. When set to false, stacks are loaded from a json file only once - when database is initialized. When set to true, json is sourced every time Che server starts.

Code samples allow you to define sample projects that are cloned into a workspace if the user chooses it when creating a new project. You can add your own. In your `${LOCAL_DATA_DIR}/instance/data/templates` create a json file with your custom samples - it will be sourced each time Che server starts. Here's how default [Che samples.json](https://github.com/eclipse/che/blob/master/ide/che-core-ide-templates/src/main/resources/samples.json) look like.

## Workspace Limits

You can place limits on how users interact with the system to control overall system resource usage. You can define how many workspaces created, RAM consumed, idle timeout, and a variety of other parameters.

You can also set limits on Docker's allocation of CPU to workspaces, which may be necessary if you have a very dense workspace population where users are competing for limited physical resources.

Workspace idle timeout can be configured in `che.env` , so that inactive workspaces will be shutdown automatically over this length of time in milliseconds. By default, this value is set to 3600000 (1 hour). If set to "0", then workspaces will not be stopped automatically. Currently, keyboard and mouse interactions in IDE, as well as HTTP requests to ws-agent count as activity

## JAVA_OPTS

There can be several Java processes running in a workspace machine. Some of the Java agents are special purpose agents started in a machine to provide core and additional IDE functionality. These are workspace agent and a [Maven plugin][dependency-management] that are both started in own JVM. On top of that, you can run own Java programs and use build tools like Maven. A set of the following environment variables can help optimize RAM consumption:


**User-Defined Envs**

A user can provide own [environment variables][env-variables] per workspace machine:

```
JAVA_OPTS                                    # machine-wide java opts
MAVEN_OPTS                                   # machine-wide maven opts
CHE_WORKSPACE_WSAGENT__JAVA__OPTIONS           # java opts to adjust java opts of ws-agent
CHE_WORKSPACE_MAVEN__SERVER__JAVA__OPTIONS   # java opts to adjust java opts of the maven server
```

Che admins (basically whoever has access to `che.env` or Che server environment directly) can override user-defined envs:

```
CHE_WORKSPACE_JAVA__OPTIONS                 # overrides the default value of JAVA_OPTS of all workspaces
CHE_WORKSPACE_MAVEN__OPTIONS                # overrides the default value of MAVEN_OPTS of all workspaces
CHE_WORKSPACE_WSAGENT__JAVA__OPTIONS        # overrides the default value of JAVA_OPTS of all ws-agents
CHE_WORKSPACE_MAVEN__SERVER__JAVA__OPTIONS  # overrides the default value of JAVA_OPTS of all maven servers
```

You can find default values in [che.env](https://github.com/eclipse/che/blob/master/dockerfiles/init/manifests/che.env#L127-L141).

## Hostname

The IP address or DNS name of where the Che endpoint will service your users. If you are running this on a local system, we auto-detect this value as the IP address of your Docker daemon. On many systems, especially those from cloud hosters like DigitalOcean, you may have to explicitly set this to the external IP address or DNS entry provided by the provider. You can edit this value in `che.env` and restart Che, or you can pass it during initialization:

```
docker run <OTHER-DOCKER_OPTIONS> -e CHE_HOST=<ip-addr-or-dns> eclipse/che:<version> start
```

## Networking

Eclipse Che makes connections between three entities: the browser, the Che server running in a Docker container, and a workspace running in a Docker container.

If you distribute these components onto different nodes, hosts or IP addresses, then you may need to add additional configuration parameters to bridge different networks.

Also, since the Che server and your Che workspaces are within containers governed by a Docker daemon, you also need to ensure that these components have good bridges to communicate with the daemon.

Generally, if your browser, Che server and Che workspace are all on the same node, then `localhost` configuration will always work.

**WebSockets**

Che relies on web sockets to stream content between workspaces and the browser. We have found many networks and firewalls to block portions of Web socket communications. If there are any initial configuration issues that arise, this is a likely cause of the problem.

**Topology**

The Che server runs in its own Docker container, "Che Docker Container", and each workspace gets an embedded runtime which can be a set of additional Docker cotainers, "Docker Container(n)". All containers are managed by a common Docker daemon, "docker-ip", making them siblings of each other. This includes the Che server and its workspaces - each workspace runtime environment has a set of containers that is a sibling to the Che server, not a child.

**Connectivity**

The browser client initiates communication with the Che server by connecting to `che-ip`. This IP address must be accessible by your browser clients. Internally, Che runs on Tomcat which is bound to port `8080`. This port can be altered by setting `CHE_PORT` during start or in your `che.env`.

When a user creates a workspace, the Che server connects to the Docker daemon at `docker-ip` and uses the daemon to launch a new set of containers that will power the workspace. These workspace containers will have a Docker-configured IP address, `workspace-container-ip`. The `workspace-container-ip` isn't usually reachable by your browser host, `docker-ip` will be used to establish the connections between the browser and workspace containers.

Che goes through a progression algorithm to establish the protocol, IP address and port to establish communications when it is booting or starting a workspace. You can override certain parameters in Che's configuration to overcome issues with the Docker daemon, workspaces, or browsers being on different networks.

```
# Browser --> Che Server
#    1. Default is '${CHE_HOST}:${SERVER_PORT}/wsmaster/api'. In fact, requests are sent to whatever IP/hostname is in your browser address bar
#    2. Else use the value of che.api
#
# Che Server --> Docker Daemon Progression:
#    1. Use the value of che.infra.docker.daemon_url
#    2. Else, use the value of DOCKER_HOST system variable
#    3. Else, use Unix socket over unix:///var/run/docker.sock
#    4. Else default docker0 IP - 172.17.42.1
#
# Che Server --> Workspace Connection:
#        1. Use the value of che.docker.ip
#        2. Else, use address of docker0 bridge network, if available
#
# Browser --> Workspace Connection:
#    1. Use the value of che.docker.ip.external
#    2. Else, use che.docker.ip value
#    3. Else use value provided by ws container inspect
#
# Workspace Agent --> Che Server
#    1. If set, use value of CHE_INFRA_DOCKER_MASTER__API__ENDPOINT
#    2. Default is 'http://che-host:${SERVER_PORT}/api', where 'che-host' is IP of docker0 (linux) VM IP (Mac and Win).
```

It is common for configuration with firewalls, routers, networks and hosts to make the default values we detect to establish these connections incorrect. You can run `docker run <DOCKER_OPTIONS> eclipse/che info --network` to run a test that makes connections between simulated components to reflect the networking setup of Che as it is configured. You do not need all connections to pass for Che to be properly configured. For example, on a Windows machine, this output may exist, just indicating that `localhost` is not an acceptable domain for communications, but the IP address `10.0.75.2` is.

```
INFO: ---------------------------------------
INFO: --------   CONNECTIVITY TEST   --------
INFO: ---------------------------------------
INFO: Browser    => Workspace Agent (localhost): Connection failed
INFO: Browser    => Workspace Agent (10.0.75.2): Connection succeeded
INFO: Server     => Workspace Agent (External IP): Connection failed
INFO: Server     => Workspace Agent (Internal IP): Connection succeeded
```

You can also perform additional tests yourself against an already-running Che server. You will need to use `docker ps` and `docker inspect` on the command line to get the container name and IP address of your Che server, and then you can run additional tests:

```
# Browser => Workspace Ageent (External IP):
$ curl http://<che-ip>:<che-port>/wsagent/ext/

# Server => Workspace Agent (External IP):
docker exec -ti <che-container-name> curl http://<che-ip>:<che-port>/wsagent/ext/

# Server => Workspace Agent (Internal IP):
docker exec -ti <che-container-name> curl http://<workspace-container-ip>:4401/wsagent/ext/
```

**DNS Resolution**

The default behavior is for Che and its workspaces to inherit DNS resolver servers from the host. You can override these resolvers by setting `CHE_DNS_RESOLVERS` in the `che.env` file and restarting Che. DNS resolvers allow programs and services that are deployed within a user workspace to perform DNS lookups with public or internal resolver servers. In some environments, custom resolution of DNS entries (usually to an internal DNS provider) is required to enable the Che server and the workspace runtimes to have lookup ability for internal services.

```shell
# Update your che.env with comma separated list of resolvers:
CHE_DNS_RESOLVERS=10.10.10.10,8.8.8.8
```

## Single-Port Routing

Currently not supported in Che 6.  

## Private Images  

When users create a workspace in Eclipse Che, they must select a Docker image to power the workspace. We provide ready-to-go stacks which reference images hosted at the public Docker Hub, which do not require any authenticated access to pull. You can provide your own images that are stored in a local private registry or at Docker Hub. The images may be publicly or privately visible, even if they are part of a private registry.

If your stack images that Che wants to pull require authenticated access to any registry then you must configure registry authentication.

In `che.env`:

```
CHE_DOCKER_REGISTRY_AUTH_REGISTRY1_URL=url1
CHE_DOCKER_REGISTRY_AUTH_REGISTRY1_USERNAME=username1
CHE_DOCKER_REGISTRY_AUTH_REGISTRY1_PASSWORD=password1

CHE_DOCKER_REGISTRY_AWS_REGISTRY1_ID=id1
CHE_DOCKER_REGISTRY_AWS_REGISTRY1_REGION=region1
CHE_DOCKER_REGISTRY_AWS_REGISTRY1_ACCESS__KEY__ID=key_id1
CHE_DOCKER_REGISTRY_AWS_REGISTRY1_SECRET__ACCESS__KEY=secret1
```

There are different configurations for AWS EC2 and the Docker registry. You can define as many different registries as you'd like, using the numerical indicator in the environment variable. In case of adding several registries just copy set of properties and append `REGISTRY[n]` for each variable.

**Pulling Private Images in Stacks**

Once you have configured private registry access, any Che stack that has a `FROM <registry>/<repository>` that requires authenticated access will use the provided credentials within `che.env` to access the registry.

```text  
# Syntax
FROM <repository>/<image>:<tag>

# Example:
FROM my.registry.url:9000/image:latest
```

Read more about registries in the [Docker documentation](https://docs.docker.com/registry/).

## Privileged Mode

Docker privileged mode allows a container to have root-level access to the host from within the container. This enables containers to do more than they normally would, but opens up security risks. You can enable your workspaces to have privileged mode, giving your users root-level access to the host where Che is running (in addition to root access of their workspace). Privileged mode is necessary if you want to enable certain features such as Docker in Docker.

By default, Che workspaces powered by a Docker container are not configured with Docker privileged mode.  There are many security risks to activating this feature - please review the various issues with blogs posted online.  

```shell
# Update your che.env:
CHE_DOCKER_PRIVILEGED=true
```

## Mirroring Docker Hub  

If you are running a private registry internally to your company, you can [optionally mirror Docker Hub](https://docs.docker.com/registry/recipes/mirror/). Your private registry will download and cache any images that your users reference from the public Docker Hub. You need to [configure your Docker daemon to make use of mirroring](https://docs.docker.com/registry/recipes/mirror).

## Using Docker In Workspaces

If you'd like your users to work with projects which have their own Docker images and Docker build capabilities inside of their workspace, then you need to configure the workspace to work with Docker. You have three options:

1. Activate Docker privileged mode, where your user workspaces have access to the host.

```shell
# Update your codenvy.env to allow all Che workspaces machines/containers privileged rights:
CHE_DOCKER_PRIVILEGED=true;
```
2. Configure Che workspaces to volume mount the host docker daemon socket file.

```shell
# Update your codenvy.env to allow all Che workspaces to volume mount their host Daemon when starting:
CHE_WORKSPACE_VOLUME=/var/run/docker.sock:/var/run/docker.sock;
```
3. Configure Docker daemon to listen to listen to tcp socket and specify `DOCKER_HOST` environment variable in workspace machine. Each host environment will have different network topology/configuration so below is only to be used as general example.
Configure your Docker daemon to listen on TCP.  First, add the following to your Docker configuration file (on Ubuntu it's `/etc/default/docker` - see the Docker docs for the location for your OS):

Second, export `DOCKER_HOST` variable in your workspace. You can do this in the terminal or make it permanent by adding `ENV DOCKER_HOST=tcp://$IP:2375` to a workspace recipe, where `$IP` is your docker daemon machine IP.

```shell
# Listen using the default unix socket, and on specific IP address on host.
# This will vary greatly depending on your host OS.
sudo dockerd -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375
# Verify that the Docker API is responding at: http://$IP:2375/containers/json
```

```shell
# In workspace machine
docker -H tcp://$IP:2375 ps

# Shorter form
export DOCKER_HOST="tcp://$IP:2375"
docker ps
```

These three tactics will allow user workspaces to perform `docker` commands from within their workspace to create and work with Docker containers that will be outside the workspace. In other words, this makes your user's workspace feel like their laptop where they would normally be performing `docker build` and `docker run` commands.

You will need to make sure that your user's workspaces are powered from a stack that has Docker installed inside of it. Che default Docker recipe images do not have Docker installed, but you can build own image though [TODO: link to custom stack authoring].

## Development Mode

You can debug the Che binaries that are running within the Che server. You can debug either the binaries that are included within the `eclipse/che-server` image that you download from DockerHub or you can mount a local Che git repository to debug binaries built in a local assembly. By using local binaries, this allows Che developers to perform a rapid edit / build / run cycle without having to rebuild Che's Docker images.

Dev mode is activated by passing `--debug` to any command on the CLI.

```
# Activate dev mode with embedded binaries
docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    -v <local-path>:/data \
                       eclipse/che:<version> [COMMAND] --debug
```

You can replace the binaries in your local image with local binaries by volume mounting the Che git repository to `:/repo` in your Docker run command.

```
docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    -v <local-path>:/data \
                    -v <local-repo>:/repo \
                       eclipse/che:<version> [COMMAND] --debug
```

You can also optionally use your local binaries in production mode by volume mounting `:/repo` without passing `--debug`. There are two locations that files in your Che source repository will be used instead of those in the image:

1. During the `che config` phase, the source repository's `/dockerfiles/init/modules` and `/dockerfiles/init/manifests` will be used instead of the ones that are included in the `eclipse/che-init` container.
2. During the `che start` phase, a local assembly from `assembly/assembly-main/target/` is mounted into the `eclipse/che-server` runtime container. You must `mvn clean install` the `assembly/assembly-main/` folder prior to activating development mode.

Volume mounting `:/repo` will also make use of your repository's puppet manifests and other files (replacing those that are stored within the CLI's base image). If you only want to volume mount a new set of assemblies and ignore the other items in a repository, you can do so by volume mounting `:/assembly` to a folder that is the base of a binary (we do not yet support volume mounting a `.tgz` file).

```
docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    -v <local-path>:/data \
                    -v <local-assembly-folder>:/assembly \
                       eclipse/che:<version> [COMMAND]
```


To activate jpda suspend mode for debugging Che server initialization, in the `che.env`:

```
CHE_DEBUG_SUSPEND=true
```

To change che debug port, in the `che.env`:

```
CHE_DEBUG_PORT=8000
```

## Production Mode

You can also build own `INIT` and `SERVER` images to have custom configuration and binaries. To do so, clone [Che repo](https://github.com/eclipse/che) and copy `dockerfiles` dir to the root of your custom assembly. If your custom Che server does not need any custom configuration, you may proceed to building Che server image by executing `dockerfiles/build.sh`. Once done, tag the resulted image the way you need. If your custom Che server requires custom configuration, and you want to let users override them in `che.env`, you will need to build own init image with a custom [che.env](https://github.com/eclipse/che/blob/master/dockerfiles/init/manifests/che.env) file.

Once done, you can start your custom binaries this way:

```
docker run -ti -v '/var/run/docker.sock:/var/run/docker.sock -v /local/data/path:/data -e "IMAGE_CHE=your/che-server" -e "IMAGE_INIT=your/init-image" eclipse/che:$tag start'
```

`IMAGE_CHE` is the image you have built in `dockerfiles/che`, and `IMAGE_INIT` is the one from `dockerfiles/init`.

## Docker Unix Socket Mounting vs TCP Mode

The `-v /var/run/docker.sock:/var/run/docker.sock` syntax is for mounting a Unix socket so that when a process inside the container speaks to a Docker daemon, the process is redirected to the same socket on the host system.

However, peculiarities of file systems and permissions may make it impossible to invoke Docker processes from inside a container. If this happens, the Che startup scripts will print an error about not being able to reach the Docker daemon with guidance on how to resolve the issue.

An alternative solution is to run Docker daemon in TCP mode on the host and export `DOCKER_HOST` environment variable in the container.  You can tell the Docker daemon to listen on both Unix sockets and TCP.  On the host running the Docker daemon:

```shell  
# Set this environment variable and restart the Docker daemon
DOCKER_OPTS=" -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock"

# Verify that the Docker API is responding at:
http://localhost:2375/info
```

Having verified that your Docker daemon is listening, run the Che container with the with `DOCKER_HOST` environment variable set to the IP address of `docker0` or `eth0` network interface. If `docker0` is running on 1.1.1.1 then:

```shell  
docker run -ti -e DOCKER_HOST=tcp://1.1.1.1:2375 -v /var/run/docker.sock:/var/run/docker.sock -v ~/Documents/che-data1:/data eclipse/che start
```

Alternatively, you can save this env in `che.env` and restart Che.

## Proxies/Firewalls/Ports

You can install and operate Che behind a proxy:

1. Configure each physical node's Docker daemon with proxy access.
2. Optionally, override workspace proxy settings for users if you want to restrict their Internet access.

Before starting Che, configure [Docker's daemon for proxy access](https://docs.docker.com/engine/admin/systemd/#/http-proxy). If you have Docker for Windows or Docker for Mac installed on your desktop and installing Che, these utilities have a GUI in their settings which let you set the proxy settings directly.

Please be mindful that your `HTTP_PROXY` and/or `HTTPS_PROXY` that you set in the Docker daemon must have a protocol and port number. Proxy configuration is quite finnicky, so please be mindful of providing a fully qualified proxy location.

If you configure `HTTP_PROXY` or `HTTPS_PROXY` in your Docker daemon, we will add `localhost,127.0.0.1,CHE_HOST` to your `NO_PROXY` value where `CHE_HOST` is the DNS or IP address. We recommend that you add the short and long form DNS entry to your Docker's `NO_PROXY` setting if it is not already set.

We will add some values to `che.env` that contain some proxy overrides. You can optionally modify these with overrides:

```
CHE_HTTP_PROXY=<YOUR_PROXY_FROM_DOCKER>
CHE_HTTPS_PROXY=<YOUR_PROXY_FROM_DOCKER>
CHE_NO_PROXY=localhost,127.0.0.1,<YOUR_CHE_HOST>
CHE_HTTP_PROXY_FOR_WORKSPACES=<YOUR_PROXY_FROM_DOCKER>
CHE_HTTPS_PROXY_FOR_WORKSPACES=<YOUR_PROXY_FROM_DOCKER>
CHE_NO_PROXY_FOR_WORKSPACES=localhost,127.0.0.1,<YOUR_CHE_HOST>
```

The last three entries are injected into workspaces created by your users. This gives your users access to the Internet from within their workspaces. You can comment out these entries to disable access. However, if that access is turned off, then the default templates with source code will fail to be created in workspaces as those projects are cloned from GitHub.com. Your workspaces are still functional, we just prevent the template cloning.

On Linux, a firewall may block inbound connections from within Docker containers to your localhost network. As a result, the workspace agent is unable to ping the Che server. You can check for the firewall and then disable it.

Firewalls will typically cause traffic problems to appear when you are starting a new workspace. There are certain network configurations where we direct networking traffic between workspaces and Che through external IP addresses, which can flow through routers or firewalls. If ports or protocols are blocked, then certain functions will be unavailable.


**Running Behind a Firewall (Linux/Mac)**

```shell
# Check to see if firewall is running:
systemctl status firewalld

# Check for list of open ports
# Verify that ports 8080tcp, 32768-65535tcp are open
firewall-cmd --list-ports

# Optionally open ports on your local firewall:
firewall-cmd --permanent --add-port=8080/tcp
... and so on

# You can also verify that ports are open:
nmap -Pn -p <port> localhost

# If the port is closed, then you need to open it by editing /etc/pf.conf.
# For example, open port 1234 for TCP for all interfaces:
pass in proto tcp from any to any port 1234

# And then restart your firewall
```

**Running Che Behind a Firewall (Windows)**

There are many third party firewall services. Different versions of Windows OS also have different firewall configurations. The built-in Windows firewall can be configured in the control panel under "System and Security":

1. In the left pane, right-click `Inbound Rules`, and then click `New Rule` in the action pane.
2. In the `Rule Type` dialog box, select `Port`, and then click `Next`.
3. In the `Protocol and Ports` dialog box, select `TCP`.
4. Select specific local ports, enter the port number to be opened and click `Next`.
5. In the `Action` dialog box, select `Allow the Connection`, and then click `Next`.
6. In the `Name` dialog box, type a name and description for this rule, and then click `Finish`.


**Limiting Che Ports**

Eclipse Che uses Docker to power its workspaces. Docker uses the [ephemeral port range](https://en.wikipedia.org/wiki/Ephemeral_port) when exposing ports for services running in the container. So when a Tomcat server is started on port 8080 inside a Che workspace Docker automatically selects an available port from the ephemeral range at runtime to map to that Tomcat instance.

Docker will select its ports from anywhere in the ephemeral range. If you wish to reduce the size of the ephemeral range in order to improve security you can do so, however, keep in mind that each Che workspace will use at least 2 ports plus whatever ports are required for the services the user adds to their workspace.

Limiting the ephemeral range can only be done at the host level - you can read more about it (and some of the risks in doing so) here: [http://www.ncftp.com/ncftpd/doc/misc/ephemeral_ports.html](http://www.ncftp.com/ncftpd/doc/misc/ephemeral_ports.html)

To change the ephemeral range:

* On Linux: [http://www.ncftp.com/ncftpd/doc/misc/ephemeral_ports.html#Linux](http://www.ncftp.com/ncftpd/doc/misc/ephemeral_ports.html#Linux)
* On Windows: [http://www.ncftp.com/ncftpd/doc/misc/ephemeral_ports.html#Windows](http://www.ncftp.com/ncftpd/doc/misc/ephemeral_ports.html#Windows)
