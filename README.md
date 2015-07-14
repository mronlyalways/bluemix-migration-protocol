Migrating Embedded Jetty to IBM Bluemix
=======================================

Overview
---------

This document describes the process of migrating an existing Java 8 web application to IBM Bluemix. The app uses Jetty in embedded mode and offers WebSockets for Push Notifications alongside traditional HTTP requests.

Code changes
-------------

### Maven

#### Resources

First, we need to adapt Maven to include all necessary resource files (i.e. all files in `src.main.webapp` and `src.main.resources.scenarios`). Locally this step was not necessary, because those directories were added to *"Order and Export"* in the *Eclipse Project Properties* and therefore copied to the output folder by the Eclipse Java Builder. On Bluemix, we only use Maven, so Maven needs to know about those resources too. This can be done by adding a `resources`-tag to the `pom.xml` file:
```
<build>
	<resources>
		<resource>
			<directory>src/main/webapp</directory>
		</resource>
		<resource>
			<directory>src/main/resources/scenarios</directory>
			<targetPath>scenarios</targetPath>
		</resource>
	</resources>
	...
</build>
```
This way, the `src.main.webapp` directory will be copied to the root of the resulting JAR, while `src.main.scenarios` will be copied to the folder `/scenarios` inside the JAR. `.classpath`, `.settings/` and `.project` are no longer needed in the repository.

#### Plugins
The created JAR must include all the dependencies in order to be executable on Bluemix. For this task, we use the *Maven Assembly Plugin*. Since the archive is executable, it needs also needs to have a main class, i.e. a predefined point of program entry. In our case, that is the `com.ibm.disposition.core.Server` class.

Add this code to your `pom.xml`:
```
<plugin>
	<artifactId>maven-assembly-plugin</artifactId>
	<configuration>
		<archive>
			<manifest>
				<mainClass>com.ibm.disposition.core.Server</mainClass>
			</manifest>
		</archive>
		<descriptorRefs>
			<descriptorRef>jar-with-dependencies</descriptorRef>
		</descriptorRefs>
	</configuration>
</plugin>
```
### Ports

When deploying an application to Cloud Foundry, the app does not directly communicate with a client, but uses the *Cloud Foundry Droplet Execution Agent* (**DEA**) as an intermediary. Therefore communication with Cloud Foundry works two-sided. For clients, communication is limited to the ports **80** and **443** for *HTTP* requests and **4443** for *TCP/WebSocket* traffic. Communication between the *DEA* and the application happens via a port that is assigned during the deployment phase (staging). The port is stored in an environment variable called `"PORT"` on the server and can be accessed via `System.getEnv("PORT")` in the application code.
```

                               +---------------+                                           
                               |               |                                           
                               |               |                          +---------------+
+----------+                   |      CF       |                          |               |
|          +------------------>+    DROPLET    +------------------------->+               |
|  CLIENT  |    80/443/4443    |   EXECUTION   |  System.getEnv("PORT");  |  APPLICATION  |
|          +<------------------+     AGENT     +<-------------------------+               |
+----------+                   |               |                          |               |
                               |               |                          +---------------+
                               |               |                                           
                               +---------------+                                           

```
Shows an overview of the ports used in communication.

Setting up Bluemix
-------------------

### Java 8

Bluemix requires some setup in order to support Java 8. First, we need to set the correct Runtime Environment (which is Java 7 by default). This can be done either by setting an environment variable in the Bluemix Web Console or with the `cf` command line tool, or by adding the value to the `manifest.yml` file in the project.

##### Setting the environment variable via the Bluemix Web Console:

1. While in Bluemix Project Overview click: *Environment Variables* > *USER-DEFINED*
2. Add `JBP_CONFIG_IBMJDK` for *Name* and `version: 1.8.+` for *Value*

##### Setting the environment variable using the command line tool:

Execute `cf set-env myapp JBP_CONFIG_IBMJDK "version: 1.8.+"` in your command line.

##### Setting the environment variable by adding it to manifest.yml:

Add this code to your manifest.yml:
```
env:
	JBP_CONFIG_IBMJDK: "version: 1.8.+"
```
#### Summary

This setup will only affect the runtime in which the application is executed, but will not change the version of the compiler which is used by Bluemix to build the project. How to setup Java 8 for building the application will be discussed in the next Chapter.

### Setting up the "Build & Deploy" chain

The CI pipeline can be configured in the "Build & Deploy" view. The particular stages are triggered by successful execution of the respective preceding stage and take the result of that stage as their input. The first stage is triggered by `git push`.

#### Stage 1: Build and Test

The first stage will build the application, package it to an executable fat JAR (i.e. including all dependencies) and run the provided (unit-) tests. The project uses Maven as the build manager, so this needs to be chosen in the "Builder Type" field accordingly. Earlier we argued, that setting the environment to support Java 8 will not affect the compiler. This needs to be done manually by exporting the `JAVA_HOME` bash environment variable (first line of the build script).

The resulting bash script for the first job in the Build Stage:
```
#!/bin/bash
export JAVA_HOME=~/java8
mvn clean compile assembly:single
```
The second job will execute the test suite we provided for the project.

#### Stage 2: Deploy

The deployment stage uses the JAR produced in the previous step and stages it with the `cf push` command. The provided argument is the name of the JAR.

```
#!/bin/bash
cf push "${CF_APP}" -p smartdisposition-66-1.0-SNAPSHOT-jar-with-dependencies.jar

# view logs
#cf logs "${CF_APP}" --recent
```
If you have done everything right, the app should now be up and running.

## Additional notes

+ A typo in the path ```src/test/**resources**``` prevented Maven from automatically copying test resources to the output folder. Because of the fix, the plugin **Build Helper Maven Plugin** is no longer necessary.
