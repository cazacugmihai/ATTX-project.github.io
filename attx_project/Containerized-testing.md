<h1 style="color:red">Work In Progress</h1>

# Containerised Testing

This page describes the necessary steps required to run integration/BDD tests within a container or local environment alongside any required services running in their own containers.

### Table of Contents
<!-- TOC START min:1 max:3 link:true update:false -->
  - [Testing Workflow](#testing-workflow)
  - [Testing with Gradle](#testing-with-gradle)
  - [Structure of feature-test images](#structure-of-feature-test-images)
    - [runTests.sh script](#runtestssh-script)
    - [Gradle build files](#gradle-build-files)
    - [Dockerfile](#dockerfile)
    - [Configuring Jenkins](#configuring-jenkins)
  - [Running tests against custom images](#running-tests-against-custom-images)
  - [Developing tests](#developing-tests)

<!-- TOC END -->

Platform Tests repository: https://github.com/ATTX-project/platform-tests

## Testing Workflow

Overview of the steps performed during the test suite:
* Start all required Semantic Broker containers
* Wait for the services to come up
* Build test container (only during the containerised testing task)
* Start test container (only during the containerised testing task)
* Build and Run tests
* Export reports to the host machine
* Stop all containers
* The build is Successful or Failed depending on the results form the test container

## Testing with Gradle

There are two tasks for running the tests:
* `runIntegTests` -  for running the test in the local environment with all the ports exposed, thus allowing for rapid development; It will also start the containers needed, by default they will expose ports; Adding a `-PrunEnv=console` parameter will allow for continuous testing without removing any of the started containers;
* `runContainerTests` - for running tests in the CI environment or a closed test setup, and for this one needs the Gradle property `-PtestEnv=CI`. This task will build and run the tests inside a container on the same network as the other containers without the need of exposing all the ports.

Running the tests inside the container:
* `gradle -PregistryURL=attx-dev:5000 -PsnapshotRepoURL=http://attx-dev:8081 -PtestEnv=CI clean runContainerTests`

Run the test locally and exposing the ports. At the end of the tests the containers are removed:
* `gradle -PregistryURL=attx-dev:5000  -PsnapshotRepoURL=http://attx-dev:8081 -PtestEnv=dev clean runIntegTests`

Run the test locally from console and exposing the container ports:
* `gradle -PregistryURL=attx-dev:5000  -PsnapshotRepoURL=http://attx-dev:8081 -PtestEnv=dev -PrunEnv=console clean runIntegTests`

Test reports are exported to the folder configured in `copyReportFiles` task (default `$buildDir/reports/`).

## Structure of Feature-test Images

### runTests.sh Script

This script is the starting point for a `testContainer`. It first checks and waits (for a limited time) for certain ports in the network to become available and then runs the test suite.
**TO DO**
In the future, the goal is to use Service discovery component to query for up-and-running services.

Example: Waiting for services to be available:
```bash
dockerize -wait tcp://mysql:3306 -timeout 240s
dockerize -wait http://fuseki:3030 -timeout 60s
dockerize -wait http://gmapi:4302/health -timeout 60s
```

[Dockerize (Github)](https://github.com/jwilder/dockerize)

These checks and waits are complementary to the ones done in the Gradle script and the reason is that some tests need to have some fixture in place (e.g. ATTX DPU) which also wait for other services to start. Thus the tests need to have everything ready before starting, meaning the last dependent container needs to finish before running.

If the time-out is reached and the service is still not available, the process exits with status code 1.

### Gradle Build Files

Test project contains two Gradle configurations, one for developing tests and managing containers and the other one for running tests inside the container; these are `build.gradle` and `build.test.gradle` respectively.

**build.gradle**

Example of the most relevant parts:

```groovy

ext.src = [
    "${artifactRepoURL}/restServices/archivaServices/searchService/artifact?g=org.uh.hulib.attx.wc.uv&a=attx-l-replaceds&v=${uvreplaceDS}&p=jar":"attx-l-replaceds-${uvreplaceDS}.jar"
]

import de.undercouch.gradle.tasks.download.Download
task downloadTestFiles

for (s in src) {
    task "downloadTestFiles_${s.key.hashCode()}"(type: Download) {
        src s.key
        dest new File("$projectDir", s.value)
    }
    downloadTestFiles.dependsOn("downloadTestFiles_${s.key.hashCode()}")
}


if (!project.hasProperty("testEnv") || project.testEnv == "dev") {
    ext.testSet = "localhost"
} else if (project.testEnv == "CI"){
    ext.testSet = "container"
} else {
    throw new GradleException("Build project environment option not recognised.")
}

ext {
    testTag           = "latest"
    testImageGM       = "${testTag}"
    testImageFuseki   = "${testTag}"
    testImageES5      = "${testTag}"
    imageBase         = ("${testTag}" == "dev") ?  "attxproject" : "${imageRepo}:${imageRepoPort}"
}

dcompose {
    createComposeFile.useTags = true
    registry ("$registryURL") {
        // no user/pass
    }
    networks {
      pdTest
    }

    messagebroker {
        forcePull = true
        forceRemoveImage = true
        image = 'rabbitmq:3.6.12-management'
        networks = [pdTest]
        if (testSet == "localhost") {
            portBindings = ['4369:4369','5671:5671', '5672:5672', '15671:15671', '15672:15672', '25672:25672']
        }
        env = ['RABBITMQ_DEFAULT_USER=user', 'RABBITMQ_DEFAULT_PASS=password']
    }

    fuseki {
        forcePull = true
        forceRemoveImage = true
        image = "${imageBase}/attx-fuseki:${testImageFuseki}"
        networks = [pdTest]
        hostName = 'fuseki'
        if (testSet == "localhost") {
            portBindings = ['3030:3030']
        }
        env = ['ADMIN_PASSWORD=pw123']
    }


    graphmanager {
        forcePull = true
        forceRemoveImage = true
        image = "${imageBase}/gm-api:${testImageGM}"
        dependsOn = [messagebroker, fuseki]
        networks = [pdTest]
        hostName = 'graphmanager'
        env = ['MHOST=messagebroker', 'GHOST=fuseki']
        volumes = ['/attx-sb-shared']
        if (testSet == "localhost") {
            portBindings = ['4302:4302']
        }
        binds = ["${volumeDir}:/attx-sb-shared:rw"]
    }

    test {
        ignoreExitCode = true
        baseDir = file('.')
        dockerFilename = 'Dockerfile'
        buildArgs = ['UVreplaceDS': "$uvreplaceDS"]
        env = ["REPO=$artifactRepoURL"]
        if (testSet == "container") {
            binds = ["/var/run/docker.sock:/run/docker.sock"]
        }
        dependsOn = [graphmanager] // This dependency is not really needed however it reminds the scope of the tests.
        command = ['sh', '-c', '/tmp/runTests.sh']
        waitForCommand = true
        forceRemoveImage = true
        attachStdout = true
        attachStderr = true
        networks = [pdTest]
        volumes = ['/attx-sb-shared']
        binds = ["${volumeDir}:/attx-sb-shared:rw"]
    }
}

task copyFilesIntoContainer(type: DockerCopyFileToContainer) {
    targetContainerId { dcompose.graphmanager.containerId }
    hostPath = "$projectDir/src/test/resources/data/"
    remotePath = "/attx-sb-shared/"
}

task copyReportFiles(type: DcomposeCopyFileFromContainerTask) {
    service = dcompose.test
    containerPath = '/tmp/build/reports/tests'
    destinationDir = file("build/reports/")
    cleanDestinationDir = false
}

startTestContainer.finalizedBy copyReportFiles


// making sure the that fresh build of test classes is done before building the image
// Other prerequisites for the tests (e.g. artifacts) should be added here.
buildTestImage.dependsOn downloadTestFiles
buildTestImage.dependsOn testClasses

task checkDPUDone(type: DockerWaitContainer) {
    dependsOn startGmapiContainer
    targetContainerId {dcompose.attxdpus.containerId}
    doLast{
        if(getExitCode() != 0) {
            println "ATTX DPU Container failed with exit code \${getExitCode()}"
        } else {
            println "Everything is peachy."
        }
    }
}

startTestContainer.dependsOn checkDPUDone

task runContainerTests {
    dependsOn startTestContainer
    finalizedBy removeImages
    doLast {
        if(dcompose.test.exitCode != 0){ throw new GradleException("Tests within the container Failed!") }
    }
}
```

`dcompose` task configuration is first used to reference all the images required by the tests. In the example, test will require only the container (e.g. `graphmanager`) that is the object of the tests, meanwhile the container that is the object of the tests will require the additional containers (`fuseki`, `messagebroker` and `graphmanager`) in addition to the test container.
Service configurations should include at least `image` and in order to run the tests in a local environment the `portBindings` properties. Test container configuration is mostly same for all the test projects.

Rest of the configuration is the same for all the test projects (assuming that test container is always called `test`). `copyReportFiles` task copies report files from inside the container to the build/from-container directory.

`downloadTestFiles` downloads the `dev-test-helper` artifact as the CI cannot download the artifact from inside the container without having the artifact repository (Archiva) exposed publicly or on the same container network as the `dcompose` task network.

**(not used as the tests are build and run inside the test container)** `ShadowJar` task create one big jar with all the dependencies required to run the tests, which allows us to run Gradle within in the container in offline mode, which does not trigger dependency downloads (faster). Other option would have been to attach a volume with the preloaded dependencies, but that would also required maintenance.  

Dependencies are loaded from the überjar created by the `shadowJar` task in the `build.gradle`. This jar and classes directory includes everything that is needed to run the tests, so we can actually run Gradle in the offline mode (`--offline`). Building step is faster, since we don't have to download all the dependencies every time the tests are run.
In order to use the `shadowJar` the following are required in the build file:

```groovy
shadowJar {
    classifier = 'tests'
    from sourceSets.test.output
    configurations = [project.configurations.testRuntime]
}
buildTestImage.dependsOn shadowJar
```

`checkDPUDone` waits for the ATTX DPUS to be added to the front-end where there is a need for such test fixtures. The ATTX DPU waits for MySQL  (for about 4 minutes to be up) in order to add the DPU. If everything is OK the exit code of that container will be (By convention, an 'exit 0' indicates success - http://www.tldp.org/LDP/abs/html/exit-status.html). The same strategy is used to check the Tests fail or not inside the container - if the exit code is `0` the tests were successful if not the tests failed.

**build.test.gradle**

This is a very minimal build file that basically only includes dependency configuration.

The system properties are required to be set in the container and these need to be the same as the `hostName` of the containers so that are no conflicts when accessing them in the network. The solution of having them inside the Gradle configurations instead of the test code is a decision of keeping the implementation independent of the configuration of the containers.

Another reason is to allow for the tests to be run in local environment, meaning the ports are exposed and the tests need to take into consideration that the requests make inside the network between containers need to be done under the `hostName` of the specific containers (e.g. no http://localhost:4301/health but instead http://wfapi:4301/health).

```groovy
plugins {
    id "java"
}

ext {
    imageRepo = "attx-dev"
    artifactRepoPort = "8081"
}

if (!project.hasProperty("artifactRepoURL")) {
    ext.artifactRepoURL = "http://${imageRepo}:${artifactRepoPort}"
}

repositories {
    mavenCentral()
    maven { url "${artifactRepoURL}/repository/attx-releases"}
}

dependencies {
    testCompile "org.junit.jupiter:junit-jupiter-api:5.0.0",
            'org.junit.platform:junit-platform-runner:1.0.1',
            'info.cukes:cucumber-java8:1.2.5',
            'info.cukes:cucumber-junit:1.2.5',
            'com.mashape.unirest:unirest-java:1.4.9',
            'org.skyscreamer:jsonassert:1.5.0',
            'org.awaitility:awaitility-groovy:3.0.0',
            'org.uh.hulib.attx.dev:dev-test-helper:1.5',
            'org.uh.hulib.attx.wc.uv:uv-common:1.0-SNAPSHOT',
            'com.rabbitmq:amqp-client:4.2.0',
            'org.jdom:jdom2:2.0.5',
            'org.apache.jena:jena-core:3.4.0'
    testRuntime \
        "org.junit.jupiter:junit-jupiter-engine:5.0.0",
            'org.junit.vintage:junit-vintage-engine:4.12.0'
}

task containerIntegTest(type: Test) {
    Map<String, Integer> serviceMap = [ "frontend" : 8080,
                  "uvprov" : 4301,
                  "provservice" : 7030,
                  "fuseki" : 3030,
                  "graphmanager" : 4302,
                  "es5": 9210,
                  "rmlservice": 8090,
                  "messagebroker": 5672,
                  "messagebroker": 5671,
                  "messagebroker": 4369 ]
    ext.getHostPort = {services ->
        serviceMap.each{ host, port ->
            systemProperty "${host}.port", "${port}".toInteger()
            systemProperty "${host}.host", "${host}"
        }
    }

    ext.removeHostPort = { services ->
        serviceMap.each{ host, port ->
            systemProperties.remove "${host}.port"
            systemProperties.remove "${host}.host"
         }
    }

    doFirst {
        getHostPort(serviceMap)
    }
    forkEvery = 2
    maxParallelForks = 2
    testLogging {
        showStackTraces = true
    }
    doLast {
        removeHostPort(serviceMap)
    }

}
```


### Dockerfile Description

```Dockerfile
FROM frekele/gradle:4.1-jdk8

RUN apt-get update \
    && apt-get install -y wget \
    && apt-get clean

ENV DOCKERIZE_VERSION v0.5.0

RUN wget https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz \
    && tar -C /usr/local/bin -xzvf dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz \
    && rm dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz

RUN mkdir -p /tmp

WORKDIR /tmp

ARG UVreplaceDS

COPY src /tmp/src
COPY build.test.gradle /tmp/build.gradle
COPY runTests.sh /tmp
RUN chmod 700 /tmp/runTests.sh

COPY attx-l-replaceds-${UVreplaceDS}.jar /tmp/attx-l-replaceds-${UVreplaceDS}.jar
```

`frekele/gradle` image contains all the components to run Gradle 4.1. `dockerize` is used by the `runTests.sh` script to poll ports on different containers as a crude way to determine whether the service each container provides is up or not. Note that only `build.test.gradle` file is copied to the image. Image also doesn't run any script by default, instead the dcompose Gradle plugin is used to attach a command  `runTests.sh` to the test container, which runs dockerized checks and runs the tests.

```bash
#!/bin/sh

# Wait for MySQL, the big number is because CI is slow.
dockerize -wait tcp://mysql:3306 -timeout 240s
dockerize -wait http://fuseki:3030 -timeout 60s
dockerize -wait http://fuseki:3030 -timeout 60s
dockerize -wait http://graphmanager:4302/health -timeout 60s

echo  "Archiva repository URL: $REPO"

gradle -b /tmp/build.gradle -PartifactRepoURL=$REPO containerIntegTest
```

### Configuring Jenkins

Example configuration for Jenkins using pipeline plugin and declarative configurations:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') { // for display purposes
            steps {
                // Get some code from a GitHub repository
                git branch: 'dev', url: 'https://github.com/ATTX-project/platform-deployment.git'
            }
        }
        stage('Compile/Package/Test') {
            steps {
                echo sh (script: "${GRADLE_HOME}/bin/gradle --console=plain -b ${workspace}/build.gradle -PregistryURL=attx-dev:5000 -PartifactRepoURL=http://archiva:8080 -PtestEnv=CI :pd-feature-tests:clean :pd-feature-tests:runContainerTests", returnStdout: true)
            }
            post {
                always {
                    echo 'publishing reports'
                    publishHTML target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: false,
                        reportDir: 'pd-feature-tests/build/reports/tests/integTest',
                        reportFiles: 'index.html',
                        reportName: 'pd-feature-tests-reports'
                        ]                    
                }
            }            
        }
    }     
}
```

Jenkins sets `testEnv=CI` in order to run the tests in a container. Report publication is configured as a `post` block of the build stage, so that is always run even if the build/test step fails.

## Running Tests Against Custom Images

If you want to run tests against other that 'latest' tag/versions of the images, do the following, where `testtag` is the tag/version needed for tests:

* Push your custom images to the private Docker repo with some tags (e.g. `attx-dev:5000/gm-api:testtag`).
* Modify `dcompose` configuration int he `build.gradle` of the project you are working with by changing the tag of the selected images. For example:

```groovy
dcompose {
...
  graphmanager {
    image = 'attx-dev:5000/gm-api:testtag'
  }
...
}
```

## Developing Tests

When developing tests, it is easier to just run them inside the IDE and work on services on the `localhost` or use the `runIntegTests` tasks for running the integration tests with the containers exposing ports.
