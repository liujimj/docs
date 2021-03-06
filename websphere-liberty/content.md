# Overview

The images in this repository contain IBM WebSphere Application Server Liberty for Developers and the IBM Java Runtime Environment. See the license section below for restrictions relating to the use of this image. For more information about WebSphere Application Server Liberty, see the [WASdev](https://developer.ibm.com/wasdev/docs/category/getting-started/) site.

# Image User

This image runs by default with `USER 1001` (non-root), as part of group `0`. All of the folders accessed by WebSphere Liberty have been given the appropriate permissions, but if your extending Dockerfile needs permission to another location you can simply temporarily switch into root and provide the needed permissions, example:

```dockerfile
USER root
RUN mkdir -p /myFolder && chown -R 1001:0 /myFolder
USER 1001
```

Also, you have to make sure that the artifacts you are copying into the image (via `COPY`) have the correct permission to be `read` and `executed` by user `1001` or group `0`. For example, you can do `chmod 777 server.xml` to ensure your `server.xml` can be `read` and `executed` by user `1001`, as well as any artifacts such as the application's `EAR` or `WAR` file, JDBC driver, or other files that are placed on the image via `COPY`.

# Tags

There are multiple tags available in this repository. The image with the tag `beta` contains the contents of the install archive for the latest monthly beta. The other images are all based on the latest generally available fix pack.

The `kernel` image contains just the Liberty kernel and no additional runtime features. This image can be used as the basis for custom built images that contain only the features required for a specific application. For example, the following Dockerfile starts with this image, copies in the `server.xml` that lists the features required by the application, and then uses the `installUtility` command to download those features from the online repository.

```dockerfile
FROM %%IMAGE%%:kernel
COPY Sample1.war /config/dropins/
COPY server.xml /config/
RUN installUtility install --acceptLicense defaultServer
```

The `webProfile8` image contains the features required for Java EE8 Web Profile compliance. The `javaee8` image extends this image and adds the features required for Java EE8 Full Platform compliance. The `javaee8` image is also tagged with `latest`.

The `webProfile7` image contains the features required for Java EE7 Web Profile compliance. The `javaee7` image extends this image and adds the features required for Java EE7 Full Platform compliance.

The `webProfile8`, `javaee8`, `webProfile7` and `javaee7` images also contain a common set of features that are expected to be of use for a typical production scenario. These features are: `appSecurity-2.0`, `collectiveMember-1.0`, `localConnector-1.0`, `ldapRegistry-3.0`, `monitor-1.0`, `requestTiming-1.0`, `restConnector-2.0`, `sessionDatabase-1.0`, `ssl-1.0`, `transportSecurity-1.0` and `webCache-1.0`.

The `microProfile1` and `microProfile2` images contains the features required to support MicroProfile 1.4 and MicroProfile 2.0 (respectively).

The `springBoot1` and `springBoot2` images contain all features required for running Spring Boot 1.5 and 2.0 applications; including `springBoot-1.5` or `springBoot-2.0`, respectively, plus `servlet-4.0`, `jsp-2.3`, `webSocket-1.1`, and `transportSecurity-1.0`.

# Usage

The images are designed to support a number of different usage patterns. The following examples are based on the Java EE8 Liberty [application deployment sample](https://developer.ibm.com/wasdev/docs/article_appdeployment/) and assume that [DefaultServletEngine.zip](https://github.com/WASdev/sample.servlet/releases/download/V1/DefaultServletEngine.zip) has been extracted to `/tmp` and the `server.xml` updated to accept HTTP connections from outside of the container by adding the following element inside the `server` stanza:

```xml
<httpEndpoint host="*" httpPort="9080" httpsPort="-1"/>
```

1.	Each image contains a default server configuration that specifies the corresponding features and exposes ports 9080 and 9443 for HTTP and HTTPS respectively. A .WAR file can therefore be mounted in the `dropins` directory of this server and run. The following example starts a container in the background running a .WAR file from the host file system with the HTTP and HTTPS ports mapped to 80 and 443 respectively.

	```console
	$ docker run -d -p 80:9080 -p 443:9443 \
	    -v /tmp/DefaultServletEngine/dropins/Sample1.war:/config/dropins/Sample1.war \
	    %%IMAGE%%:webProfile8
	```

	When the server is started, you can browse to http://localhost/Sample1/SimpleServlet on the Docker host.

	Note: If you are using the boot2docker virtual machine on OS X or Windows, you need to get the IP of the virtual host by using the command `boot2docker ip` instead of by using localhost.

2.	For greater flexibility over configuration, it is possible to mount an entire server configuration directory from the host and then specify the server name as a parameter to the run command. Note: This particular example server configuration provides only HTTP access.

	```console
	$ docker run -d -p 80:9080 \
	  -v /tmp/DefaultServletEngine:/config \
	  %%IMAGE%%:webProfile8
	```

3.	You can also build an application layer on top of this image by using either the default server configuration or a new server configuration. In this example, we have copied the `Sample1.war` from `/tmp/DefaultServletEngine/dropins` to the same directory as the following Dockerfile.

	```dockerfile
	FROM %%IMAGE%%:webProfile8
	COPY Sample1.war /config/dropins/
	```

	This can then be built and run as follows:

	```console
	$ docker build -t app .
	$ docker run -d -p 80:9080 -p 443:9443 app
	```

4.	You can mount a data volume container that contains the application and the server configuration on to the image. This has the benefit that it has no dependency on files from the host but still allows the application container to be easily re-mounted on a newer version of the application server image. This example assumes that you have copied the `/tmp/DefaultServletEngine` directory in to the same directory as the Dockerfile.

	Build and run the data volume container:

	```dockerfile
	FROM %%IMAGE%%:webProfile8
	COPY DefaultServletEngine /config
	```

	```console
	$ docker build -t app-image .
	$ docker run -d -v /config \
	    --name app app-image true
	```

	Run the WebSphere Liberty image with the volumes from the data volume container mounted:

	```console
	$ docker run -d -p 80:9080 \
	  --volumes-from app %%IMAGE%%:webProfile8
	```

# Using `springBoot` images

The `springBoot` images introduce capabilities specific to the support of Spring Boot applications, including the `springBootUtility` used to separate Spring Boot applications into thin applications and dependency library caches. To elaborate these capabilities this section assumes the standalone Spring Boot 2.0.x application `hellospringboot.jar` exists in the `/tmp` directory.

1.	A Spring Boot application JAR deploys to the `dropins/spring` directory within the default server configuration, not the `dropins` directory. Liberty allows one Spring Boot application per server configuration. The following example starts a container running a Spring Boot application.

	```console
	$ docker run -d -p 8080:9080 \
	    -v /tmp/hellospringboot.jar:/config/dropins/spring/hellospringboot.jar \
	    %%IMAGE%%:springBoot2
	```

	Similarly, you can create a Spring Boot application layer over this image by adding the application JAR to the `dropins/spring` directory. In this example we copied `hellospringboot.jar` from `/tmp` to the same directory containing the following Dockerfile.

	```dockerfile
	FROM %%IMAGE%%:springBoot2
	COPY hellospringboot.jar /config/dropins/spring/
	```

	The custom image can be built and run as follows.

	```console
	$ docker build -t app .
	$ docker run -d -p 8080:9080 app
	```

2.	The `springBoot` images provide the library cache directory, `lib.index.cache`, which contains an indexed library cache created by the `springBootUtility` command. Use `lib.index.cache` to provide the library cache for a thin application.

	For example, run the following command to thin the `hellospringboot.jar` application.

	```console
	$ <wlp>/bin/springBootUtility thin \
	   --sourceAppPath=/tmp/hellospringboot.jar \
	   --targetLibCachePath=/tmp/lib.index.cache \
	   --targetThinAppPath=/tmp/thinhellospringboot.jar
	```

	You can run the thin application by mounting both the target application JAR and library cache when starting the container.

	```console
	$ docker run -d -p 8080:9080 \
	    -v /tmp/thinhellospringboot.jar:/config/dropins/spring/thinhellospringboot.jar \
	    -v /tmp/lib.index.cache:/lib.index.cache \
	    %%IMAGE%%:springBoot2
	```

	Similarly, you can use the `springBootUtility` command to create thin application and library cache layers over a `springBoot` image. The following example uses docker staging to efficiently build an image that deploys a fat Spring Boot application as two layers containing a thin application and a library cache.

	```dockerfile
	FROM %%IMAGE%%:springBoot2 as staging
	COPY hellospringboot.jar /staging/myFatApp.jar
	RUN springBootUtility thin \
	   --sourceAppPath=/staging/myFatApp.jar \
	   --targetThinAppPath=/staging/myThinApp.jar \
	   --targetLibCachePath=/staging/lib.index.cache
	FROM %%IMAGE%%:springBoot2
	COPY --from=staging /staging/lib.index.cache /lib.index.cache
	COPY --from=staging /staging/myThinApp.jar /config/dropins/spring/myThinApp.jar
	```

	For Spring Boot applications packaged with library dependencies that rarely change across continuous application updates, you can use the capabilities mentioned above to to share library caches across containers and to create even more efficient docker layers that leverage the docker build cache.

# Providing your own keystore/truststore

By default, when a `websphere-liberty` image starts, a Liberty server XML snippet is generated in `/config/configDropins/defaults/keystore.xml` that specifies a `keyStore` stanza with a generated password. This causes Liberty to generate a default keystore and truststore with a self-signed certificate when it starts (see the [Knowledge Center](https://www.ibm.com/support/knowledgecenter/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/rwlp_liberty_ssl_defaults.html) for more information). When providing your own keystore/truststore, this default behavior can be disabled by ensuring that a file already exists at `/config/configDropins/defaults/keystore.xml` (for example, added as part of your Docker build). This file can contain your keystore configuration or could just contain an empty `<server></server>` stanza.

# Using IBM JRE Class data sharing

The IBM JRE provides a feature [Class data sharing](http://www-01.ibm.com/support/knowledgecenter/SSYKE2_8.0.0/com.ibm.java.lnx.80.doc/diag/understanding/shared_classes.html) which offers transparent and dynamic sharing of data between multiple Java Virtual Machines running on the same host by using shared memory backed by a file. When running the Liberty Docker image, it looks for the file at `/opt/ibm/wlp/output/.classCache`. To benefit from Class data sharing, this location needs to be shared between containers either through the host or a data volume container.

Taking the application image from example 3 above, containers can share the host file location (containing the shared cache) `/tmp/websphere-liberty/classCache` as follows:

```console
docker run -d -p 80:9080 -p 443:9443 \
    -v /tmp/websphere-liberty/classCache:/opt/ibm/wlp/output/.classCache app
```

Or, create a named data volume container that exposes a volume at the location of the shared file:

```console
docker run -e LICENSE=accept -v /opt/ibm/wlp/output/.classCache \
    --name classcache %%IMAGE%% true
```

Then, run the WebSphere Liberty image with the volumes from the data volume container classcache mounted as follows:

```console
docker run -d -p 80:9080 -p 443:9443 --volumes-from classcache app
```

# Running WebSphere Liberty in read-only mode

Liberty writes to two different directories when running: `/opt/ibm/wlp/output` and `/logs`. In order to run the Liberty image in read-only mode these may be mounted as temporary file systems. If using the provided image, the keystore will be generated on initial start up in the server configuration. This means that the server configuration directory either needs to be read-write or the keystore will need to be built into the image. In the example command `/config` is mounted as a read-write volume.

```console
docker run -d -p 80:9080 -p 443:9443 \
    --tmpfs /opt/ibm/wlp/output --tmpfs /logs -v /config --read-only \
    %%IMAGE%%:javaee8
```

# Changing locale

The base Ubuntu image does not include additional language packs. To use an alternative locale, build your own image that installs the required language pack and then sets the `LANG` environment variable. For example, the following Dockerfile starts with the `%%IMAGE%%:webProfile8` image, installs the Portuguese language pack, and sets Brazilian Portuguese as the default locale:

```dockerfile
FROM %%IMAGE%%:webProfile8
RUN apt-get update \
  && apt-get install -y language-pack-pt-base \
  && rm -rf /var/lib/apt/lists/*
ENV LANG pt_BR.UTF-8
```
