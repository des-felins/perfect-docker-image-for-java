---
theme: default
background: /background.jpg
title: Perfect Container Image
drawings:
  persist: false
transition: none
mdc: true
shiki: { theme: "nord" }
---

# Creating a Perfect Container Image for a Java Application 

Catherine Edelveis, BellSoft


---
layout: image-right
image: "/cat.png"
---

## 'whoami'

<img/>

🥑Developer Advocate at BellSoft

😍Love Java and Spring

👩‍💻Tech writer

👾CyberJAR Channel co-host (@cbrjar)

---

## About BellSoft
<br/>

Member of:
- JCP Executive Committee
- OpenJDK Vulnerability Group
- GraalVM Advisory Board
- Linux Foundation
- Cloud Native Computing Foundation


---

## About BellSoft
<br/>

Products:
- Liberica JDK
- Liberica Native Image Kit
- Alpaquita Linux

<br/>
Liberica is the JDK officially recommended by Spring

---

## So... What is a perfect Docker image for Java?
<br/>

<v-clicks>

- Memory-efficient
- Fast start
- Fast push/pull
- Automatic containerization without a Dockerfile 
- Hardened
- All at once?

</v-clicks>

---

## This is what chasing a perfection may look like
<br/>

<img src="/monster.png" class="center"  width="400px"/>

<style>
.center {
  display: block;
  margin-left: auto;
  margin-right: auto;
}
</style>

---

## Perfection is relative
<br/>

- Set priorities
- Mix & match various techniques
  <br><br>
  <v-click>The very techniques we will explore now 😎</v-click>

---

## NeuroWatch demo application
<br/>

- Petclinic on steroids
- MongoDB
- Vaadin
- Kafka
- Spring Security


---

## Starting point: Dockerfile
<br/>

```dockerfile {none|1|3|4|6}
FROM bellsoft/liberica-openjdk-debian:25 as builder
WORKDIR /app
ADD . /app/neurowatch
RUN cd neurowatch && ./mvnw -Pproduction clean package
EXPOSE 8081
ENTRYPOINT java -jar /app/neurowatch/target/*.jar
```

<v-click at="1">

- Base image with Liberica JDK and Debian

</v-click>

<v-click at="2">

- Add the project

</v-click>

<v-click at="3">

- Build a JAR

</v-click>

<v-click at="4">

- Run it!

</v-click>

---

## Result: 788MiB (820MB)
<br/>

```plain {none|19,18|9-17|8|5,4}{maxHeight:'300px'}
ID         TAG                          SIZE      COMMAND                                                                         │
│946a796e83 neurowatch-neurowatch:latest 0B        ENTRYPOINT ["/bin/sh" "-c" "java -jar /app/neurowatch/target/*.jar"]            │
│<missing>                               0B        EXPOSE map[8081/tcp:{}]                                                         │
│<missing>                               501.22MiB RUN /bin/sh -c cd neurowatch && ./mvnw -Pproduction clean package # buildkit    │
│<missing>                               8.65MiB   COPY . /app/neurowatch # buildkit                                               │
│<missing>                               0B        WORKDIR /app                                                                    │
│<missing>                               0B        ENV JAVA_HOME=/usr/lib/jvm/jdk-24.0.1-bellsoft-x86_64 PATH=/usr/lib/jvm/jdk-24.0│
│<missing>                               123.40MiB RUN |8 LIBERICA_IMAGE_VARIANT=lite LIBERICA_VM=server LIBERICA_GENERATE_CDS=fals│
│<missing>                               0B        ARG BASE_URL=https://download.bell-sw.com/java/                                 │
│<missing>                               0B        ARG LIBERICA_ROOT=/usr/lib/jvm/jdk-24.0.1-bellsoft                              │
│<missing>                               0B        ARG LIBERICA_BUILD=11                                                           │
│<missing>                               0B        ARG LIBERICA_VERSION=24.0.1                                                     │
│<missing>                               0B        ARG LIBERICA_JVM_DIR=/usr/lib/jvm                                               │
│<missing>                               0B        ARG LIBERICA_GENERATE_CDS=false                                                 │
│<missing>                               0B        ARG LIBERICA_VM=server                                                          │
│<missing>                               0B        ARG LIBERICA_IMAGE_VARIANT=lite                                                 │
│<missing>                               0B        ENV LANG=en_US.UTF-8 LANGUAGE=en_US:en                                          │
│<missing>                               44.94MiB  RUN /bin/sh -c apt-get update                                &&    apt-get insta│
│<missing>                               111.17MiB # debian.sh --arch 'amd64' out/ 'bookworm' '@1743984000'
```


---

## 820MB - should we worry about it?
<br/>

- Longer push - longer updates
- Longer pull - slower scaling
- Dozens commits/day, 820MB each: we wil quickly run out of space for all these images

---
layout: cover
---

# Let's optimize the image size!

---

## Round 1: multi-stage builds
<br/>

- Several FROM instructions in one Dockerfile
- Clean final image
- Can use different base images

---

## Change the Dockerfile
<br/>

````md magic-move
```dockerfile
FROM bellsoft/liberica-openjdk-debian:25 as builder
WORKDIR /app
ADD . /app/neurowatch
RUN cd neurowatch && ./mvnw -Pproduction clean package
EXPOSE 8081
ENTRYPOINT java -jar /app/neurowatch/target/*.jar
```

```docker
FROM bellsoft/liberica-openjdk-debian:25 as builder

WORKDIR /app
ADD . /app/neurowatch
RUN cd neurowatch && ./mvnw -Pproduction clean package

FROM bellsoft/liberica-openjre-debian:25
WORKDIR /app
COPY --from=builder /app/neurowatch/target/neurowatch-*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```
````

---

## Result:  343MiB (359MB)
<br/>

```plain {none|7,15,16|4}{maxHeight:'300px'}
ID         TAG                          SIZE      COMMAND                                                                         │
│97855e4950 neurowatch-neurowatch:latest 0B        ENTRYPOINT ["java" "-jar" "/app/app.jar"]                                       │
│<missing>                               0B        EXPOSE map[8080/tcp:{}]                                                         │
│<missing>                               62.83MiB  COPY /app/neurowatch/target/neurowatch-*.jar app.jar # buildkit                 │
│<missing>                               0B        WORKDIR /app                                                                    │
│<missing>                               0B        ENV JAVA_HOME=/usr/lib/jvm/jre-24.0.1-bellsoft-x86_64 PATH=/usr/lib/jvm/jre-24.0│
│<missing>                               124.82MiB RUN |6 LIBERICA_VERSION=24.0.1 LIBERICA_BUILD=11 LIBERICA_VARIANT=jre LIBERICA_R│
│<missing>                               0B        ARG LIBERICA_GENERATE_CDS=false                                                 │
│<missing>                               0B        ARG LIBERICA_USE_LITE=1                                                         │
│<missing>                               0B        ARG LIBERICA_ROOT=/usr/lib/jvm/jre-24.0.1-bellsoft                              │
│<missing>                               0B        ARG LIBERICA_VARIANT=jre                                                        │
│<missing>                               0B        ARG LIBERICA_BUILD=11                                                           │
│<missing>                               0B        ARG LIBERICA_VERSION=24.0.1                                                     │
│<missing>                               0B        ENV LANG=en_US.UTF-8 LANGUAGE=en_US:en                                          │
│<missing>                               44.94MiB  RUN /bin/sh -c apt-get update                                &&    apt-get insta│
│<missing>                               111.17MiB # debian.sh --arch 'amd64' out/ 'bookworm' '@1743984000'
```

---

## Round 2: lightweight Linux for containers
<br/>

|                    | Alpaquita<br/>(musl) | Alpaquita<br/>(glibc) | Alpine<br/>(musl) | RHEL<br/>(Distroless UBI 10 Micro) | Ubuntu Jammy | Debian Slim |
|--------------------|----------------------|-----------------------|-------------------|------------------------------------|--------------|-------------|
| Size on Docker Hub | 3.8MB                | 11.7MB                | 3.45MB            | 7.37MB                             | 28.17MB      | 27.8MB      |


---

## Change the Dockerfile again
<br/>

```docker{none|1,7|2}{maxHeight:'300px'}
FROM bellsoft/liberica-runtime-container:jdk-25-musl as builder
RUN apk add --no-cache nodejs npm
WORKDIR /app
ADD . /app/neurowatch
RUN cd neurowatch && ./mvnw -Pproduction clean package

FROM bellsoft/liberica-runtime-container:jre-25-slim-musl
WORKDIR /app
COPY --from=builder /app/neurowatch/target/neurowatch-*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```


---

## Result: 189MiB (198MB)
<br/>

```plain{none|6,2}{maxHeight:'300px'}
ID         TAG                          SIZE      COMMAND                                                                         │
│c88af46c34 neurowatch-neurowatch:latest 62.83MiB  [stage-1 3/3] COPY --from=builder /app/neurowatch/target/neurowatch-*.jar app.ja│
│<missing>                               0B        ENTRYPOINT ["java" "-jar" "/app/app.jar"]                                       │
│<missing>                               0B        EXPOSE map[8080/tcp:{}]                                                         │
│<missing>                               0B        COPY /app/neurowatch/target/neurowatch-*.jar app.jar # buildkit                 │
│<missing>                               126.43MiB WORKDIR /app  
```


---
layout: cover
---

# But! The app layer has the same size (65MB)

---

## What are 65MB?
<br/>

Smallest change, and we push/pull 65MB

Can we do something about it?

---
layout: cover
---

# Let's optimize push/pull!

---

## Split a single JAR into layers
<br/>

- Separate layers for:
    - dependencies
    - SpringBootLoader
    - SNAPSHOT dependencies
    - application

---

## Dockerfile
<br/>

```dockerfile {none|7,9|10|12,15-18|20}{maxHeight:'180px'}
FROM bellsoft/liberica-runtime-container:jdk-25-musl as builder
RUN apk add --no-cache nodejs npm
WORKDIR /app
ADD . /app/neurowatch
RUN cd neurowatch && ./mvnw -Pproduction clean package

FROM bellsoft/liberica-runtime-container:jre-25-musl as optimizer
WORKDIR /app
COPY --from=builder /app/neurowatch/target/neurowatch-*.jar app.jar
RUN java -Djarmode=tools -jar /app/app.jar extract --layers --destination extracted

FROM bellsoft/liberica-runtime-container:jre-25-slim-musl

WORKDIR /app
COPY --from=optimizer /app/extracted/dependencies/ ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```
<v-click at="1">

- Intermediate optimizer stage

</v-click>

<v-click at="2">

- Extract layers

</v-click>

<v-click at="3">

- Copy layers into the final image

</v-click>

<v-click at="4">

- Run as usual

</v-click>

---

## Result: 189MiB (198MB)
<br/>

```plain{none|5-8|2}{maxHeight:'180px'}
│ID         TAG                          SIZE      COMMAND                                                                        │
│b811866cc5 neurowatch-neurowatch:latest 6.72MiB   [stage-2 6/6] COPY --from=optimizer /app/extracted/application/ ./             │
│<missing>                               0B        ENTRYPOINT ["java" "-jar" "/app/app.jar"]                                      │
│<missing>                               0B        EXPOSE map[8080/tcp:{}]                                                        │
│<missing>                               0B        COPY /app/extracted/application/ ./ # buildkit                                 │
│<missing>                               0B        COPY /app/extracted/snapshot-dependencies/ ./ # buildkit                       │
│<missing>                               459.4 KB  COPY /app/extracted/spring-boot-loader/ ./ # buildkit                          │
│<missing>                               55.86MiB  COPY /app/extracted/dependencies/ ./ # buildkit                                │
│<missing>                               126.43MiB WORKDIR /app
```

---

## What exactly are we optimizing?
<br/>

push/pull layer size!

The size of the actively updated layer is now 7MB!

<v-click at="1">Can we skip writing all that in a Dockerfile? 😰</v-click>
<br>
<v-click at="2">Can we skip Dockerfiles altogether? 😳</v-click>

---
layout: quote
---

# Mom, look, no Dockerfile!

Buildpacks

---

## Automating image build with buildpacks
<br/>

- Take the app source code and create a container image
- One command
- Layered JAR and an SBOM by default
- Configurable

---

## Basic usage
<br/>

The pack utility

```bash {1|2|3|4}
pack build neurowatch \
  --path . \
  -e BP_JVM_VERSION=25
```

Maven

```bash
mvn spring-boot:build-image
```

Gradle

```bash
gradle bootBuildImage
```

---

## Result: 357MB
<br/>

Peeking under the hood:

```plain{none|1,2|4-10|11|13-16|33}{maxHeight:'180px'}
[INFO]  > Pulling builder image 'docker.io/paketobuildpacks/builder-noble-java-tiny:latest' 100%
[INFO]  > Pulling run image 'docker.io/paketobuildpacks/ubuntu-noble-run-tiny:0.0.20' for platform 'linux/arm64' 100%
[INFO]  > Running creator
[INFO]     [creator]     6 of 26 buildpacks participating
[INFO]     [creator]     paketo-buildpacks/ca-certificates   3.10.3
[INFO]     [creator]     paketo-buildpacks/bellsoft-liberica 11.2.4
[INFO]     [creator]     paketo-buildpacks/syft              2.16.1
[INFO]     [creator]     paketo-buildpacks/executable-jar    6.13.2
[INFO]     [creator]     paketo-buildpacks/dist-zip          5.10.2
[INFO]     [creator]     paketo-buildpacks/spring-boot       5.33.3
[INFO]     [creator]       BellSoft Liberica JRE 24.0.1: Contributing to layer
[INFO]     [creator]       Creating slices from layers index
[INFO]     [creator]         dependencies (77.4 MB)
[INFO]     [creator]         spring-boot-loader (459.4 KB)
[INFO]     [creator]         snapshot-dependencies (0.0 B)
[INFO]     [creator]         application (18.0 MB)
[INFO]     [creator]     ===> EXPORTING
[INFO]     [creator]     Adding layer 'paketo-buildpacks/ca-certificates:helper'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/bellsoft-liberica:helper'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/bellsoft-liberica:java-security-properties'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/bellsoft-liberica:jre'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/executable-jar:classpath'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/spring-boot:helper'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/spring-boot:spring-cloud-bindings'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/spring-boot:web-application-type'
[INFO]     [creator]     Adding layer 'buildpacksio/lifecycle:launch.sbom'
[INFO]     [creator]     Adding layer 'buildpacksio/lifecycle:launcher'
[INFO]     [creator]     Adding layer 'buildpacksio/lifecycle:config'
[INFO]     [creator]     Adding layer 'buildpacksio/lifecycle:process-types'
[INFO]     [creator]     Saving docker.io/library/neurowatch:0.0.1-SNAPSHOT...
[INFO]     [creator]     Adding cache layer 'paketo-buildpacks/syft:syft'
[INFO]     [creator]     Adding cache layer 'paketo-buildpacks/spring-boot:spring-cloud-bindings'
[INFO]     [creator]     Adding cache layer 'buildpacksio/lifecycle:cache.sbom'

```

<v-click at="6">Ubuntu slim: no shell, bigger image</v-click>
<br>
<v-click at="7">Can use bellsoft/buildpacks.builder:musl: smaller and with shell</v-click>
<br>
<v-click at="8">Native images with buildpacks also possible</v-click>


---
layout: cover
---

# What if the service startup speed is more important than the image size?

---

## Startup problem is multi-dimensional
<br/>

- Need faster rollout for faster feedback loop
- Services starting in chain impact scaling efficiency
- Warmup is also an issue (may take minutes)

---
layout: cover
---

# Let's optimize the startup speed!

---

## Round 1: AOT Cache
<br/>

- CDS → AppCDS → AOT Cache
- Part of Project Leyden
- Java 24: Ahead-of-Time Class Loading & Linking (JEP 483)
- Java 25: Ahead-of-Time Method Profiling (JEP 515)

<br/>
<br/>

> Improve startup time by making the classes of an application instantly available, in a loaded and linked state, when the HotSpot Java Virtual Machine starts. Achieve this by monitoring the application during one run and storing the loaded and linked forms of all classes in a cache for use in subsequent runs. Lay a foundation for future improvements to both startup and warmup time.

---

## Layering a Spring app
<br/>

Useful tip: enable Spring AOT
(more data in the archive!)

```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<executions>
		<execution>
			<id>process-aot</id>
			<goals>
				<goal>process-aot</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

---

## Layering a Spring app
<br/>

```dockerfile {none|12|22-24|25-27|15}{maxHeight:'180px'}
FROM bellsoft/liberica-runtime-container:jdk-25-cds-musl as builder
RUN apk add --no-cache nodejs npm
WORKDIR /app
ADD . /app/neurowatch
RUN cd neurowatch && ./mvnw -Pproduction clean package

FROM bellsoft/liberica-runtime-container:jre-25-cds-musl as optimizer
WORKDIR /app
COPY --from=builder /app/neurowatch/target/neurowatch-*.jar app.jar
RUN java -Djarmode=tools -jar /app/app.jar extract --layers --destination extracted

FROM bellsoft/liberica-runtime-container:jre-25-cds-slim-musl

WORKDIR /app
ENTRYPOINT ["java", "-Dspring.aot.enabled=true", "-XX:AOTCache=app.aot", "-jar", "/app/app.jar"]
COPY --from=optimizer /app/extracted/dependencies/ ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
EXPOSE 8080

RUN java -Dspring.aot.enabled=true -XX:AOTMode=record / \
    -XX:AOTConfiguration=app.aotconf -Dspring.context.exit=onRefresh / \
    -jar /app/app.jar
RUN java -Dspring.aot.enabled=true -XX:AOTMode=create \
    -XX:AOTConfiguration=app.aotconf -XX:AOTCache=app.aot \
    -jar /app/app.jar

```

---

## Price: 293.9MiB (308MB)
<br/>

```plain {2}{maxHeight:'180px'}
│ID         TAG                          SIZE      COMMAND                                                                        │
│858ba1d1e4 neurowatch-neurowatch:latest 75.87MiB  mount / from exec /bin/sh -c java -Dspring.aot.enabled=true -XX:AOTMode=create │
│<missing>                               2.00MiB   RUN /bin/sh -c java -Dspring.aot.enabled=true -XX:AOTMode=create -XX:AOTConfigu│
│<missing>                               6.72MiB   RUN /bin/sh -c java -Dspring.aot.enabled=true -XX:AOTMode=record -XX:AOTConfigu│
│<missing>                               0B        EXPOSE map[8080/tcp:{}]                                                        │
│<missing>                               0B        COPY /app/extracted/application/ ./ # buildkit                                 │
│<missing>                               0B        COPY /app/extracted/snapshot-dependencies/ ./ # buildkit                       │
│<missing>                               55.86MiB  COPY /app/extracted/spring-boot-loader/ ./ # buildkit                          │
│<missing>                               0B        COPY /app/extracted/dependencies/ ./ # buildkit                                │
│<missing>                               0B        ENTRYPOINT ["java" "-Dspring.aot.enabled=true" "-XX:AOTCache=app.aot" "-jar" "/│
│<missing>                               153.52MiB WORKDIR /app 
```

The image is bigger because of the archive
<br>
Start is 50-60% faster (1.2s)


---

## Round 2: Native Image
<br/>

- GraalVM Native Image - AOT compilation of a Java app
- A single .exe file
- No JVM
- Fast start without warmup

But
- Prolonged build time
- Special treatment of dynamic features (Reflection, Dynamic Proxies, etc.)

---

## Native Image in Docker
<br/>

```dockerfile {none|1|5|7|9,10}{maxHeight:'180px'}
FROM bellsoft/liberica-native-image-kit-container:jdk-25-nik-25-musl as builder
RUN apk add --no-cache nodejs npm
WORKDIR /app
ADD . /app/neurowatch
RUN cd neurowatch && ./mvnw -Pproduction -Pnative native:compile

FROM bellsoft/alpaquita-linux-base:musl
WORKDIR /app
ENTRYPOINT ["/app/app"]
COPY --from=builder /app/neurowatch/target/native/neurowatch /app/app
```

---

## Result: 198MB
<br/>

```plain {2,5}{maxHeight:'180px'}
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
757636acfada   8 minutes ago    [stage-1 3/3] COPY --from=builder /app/neuro…   185MB     buildkit.exporter.image.v0
<missing>      55 minutes ago   COPY /app/neurowatch/target/native/neurowatc…   0B        buildkit.dockerfile.v0
<missing>      55 minutes ago   ENTRYPOINT ["/app/app"]                         0B        buildkit.dockerfile.v0
<missing>      20 hours ago     WORKDIR /app                                    8.64MB    buildkit.dockerfile.v0
```

- Based on Linux only
- Start in 0.4 s at peak performance

But:
- Need to rebuild the whole image every time

---

## Round 3: CRaC
<br/>

- Coordinated Restore at Checkpoint is an OpenJDK project
- A snapshot of a warmed-up Java app
- Like pausing and resuming a video game
- Start within milliseconds
- Good old JVM is still there 

<br/>
<v-click> Sounds tempting... Where's the catch?</v-click>


---

## CRaC is difficult
<br/>

- A snapshot may contain sensitive data
- May need to augment the code for reliable checkpoint and restore
- Not all solutions support CRaC out-of-the-box
- Need more than just a Dockerfile

---

## Spring Data MongoDB does not support CRaC

<br>
What are the options?

- Fall back to another solution
- Request support
- Implement support on our own!


---

## Add CRaC dependency
<br/>

```xml
<dependency>
    <groupId>org.crac</groupId>
    <artifactId>crac</artifactId>
    <version>1.5.0</version>
</dependency>
```

---

## Befriending MongoDB with CRaC
<br/>

Custom MongoClient

```java 
public class MongoClientProxy implements MongoClient {
    volatile MongoClient delegate;

    public MongoClientProxy(MongoClient initialClient) {
        this.delegate = initialClient;
    }

    public void close() {
        delegate.close();
    }
}  
```

---

## Befriending MongoDB with CRaC
<br/>

Custom CRaC Resource

```java {1,2|7-11|13-16|18-21}{maxHeight:'260px'}
@Component
static public class MongoClientResource implements Resource {

    private final MongoClientProxy mongoClientProxy;
    private final MongoConnectionDetails details;

    public MongoClientResource(MongoClient mongoClientProxy, MongoConnectionDetails details) {
        this.mongoClientProxy = (MongoClientProxy) mongoClientProxy;
        this.details = details;
        Core.getGlobalContext().register(this);
    }

    @Override
    public void beforeCheckpoint(Context<? extends Resource> context) {
        mongoClientProxy.delegate.close();
    }

    @Override
    public void afterRestore(Context<? extends Resource> context) {
        mongoClientProxy.delegate = MongoClients.create(details.getConnectionString());
    }
}
```

---

## Befriending MongoDB with CRaC
<br/>

Replace MongoClient in the context

```java {1,3|4|2,5}
@Bean
@Primary
public MongoClient mongoClient(MongoConnectionDetails details) {
    MongoClient initialClient = MongoClients.create(details.getConnectionString());
    return new MongoClientProxy(initialClient);
}
```

---

## Dockerfile
<br/>

```docker {none|7|11}
FROM bellsoft/liberica-runtime-container:jdk-21-musl as builder
RUN apk add --no-cache nodejs npm
WORKDIR /app
ADD . /app/neurowatch
RUN cd neurowatch && ./mvnw -Pproduction clean package

FROM bellsoft/liberica-runtime-container:jre-21-crac-cds-stream-musl

WORKDIR /app
COPY --from=builder /app/neurowatch/target/neurowatch-*.jar app.jar
ENTRYPOINT ["java", "-XX:CRaCCheckpointTo=/app/checkpoint", "-jar", "/app/app.jar"]
```

<v-click at="2">This does not create the checkpoint yet!</v-click>

---

## First stage
<br/>

Create a preliminary image

```bash
docker build -t nw-pre-crac -f Dockerfile-crac .
```

Run it

```bash
ID=$(docker run --cap-add CAP_SYS_PTRACE --cap-add CAP_CHECKPOINT_RESTORE \
-p 8080:8080 --network=host -d nw-pre-crac)

```
<br/>

- CAP_SYS_PTRACE to access the whole process tree
- CAP_CHECKPOINT_RESTORE to perform checkpoint/restore without root access

---

## Second stage
<br/>

Perform a checkpoint

```bash
docker exec -it $ID jcmd 129 JDK.checkpoint
```

Create a new image with a CRAC-ed app

```bash
docker commit $ID cracked
```

<v-click>

Now, we are ready!

```bash
docker run --rm \
    --entrypoint java \
    --network host cracked:latest \
    -XX:CRaCRestoreFrom=/app/checkpoint
```

</v-click>

---
layout: cover
---

# The end<v-click>?</v-click>

---
layout: cover
---

# A final touch
## Hardened images:
## new container security standard

---
layout: two-cols-header
---

## Shift security left without burdening the engineers
<br/>

<div class="hi-grid">
  <div class="hb">Low to zero CVEs</div>
  <div class="arrow">➡️</div>
  <div class="hb">Safer base from the start</div>

  <div class="hb">No package manager, minimalistic base</div>
  <div class="arrow">➡️</div>
  <div class="hb">Minimized attack surface</div>

  <div class="hb">Immutable component set</div>
  <div class="arrow">➡️</div>
  <div class="hb">No tampering with container at runtime</div>

  <div class="hb">SBOM, digital signature</div>
  <div class="arrow">➡️</div>
  <div class="hb">You know what's in the image and who made it</div>

  <div class="hb">Continuous patching</div>
  <div class="arrow">➡️</div>
  <div class="hb">The image stays safe</div>
</div>

<style>
.hi-grid{
  display:grid;
  grid-template-columns: 1fr 56px 1fr;
  row-gap: 14px; column-gap: 18px;
  align-items:center;
  max-width: 1100px;
}
.hb{
  background: #0f1424;
  border: 2px solid #00e6ff;
  border-radius: 12px;
  padding: 16px 18px;
  min-height: 64px;
  display:flex; align-items:center;
  font-size: 18px; line-height:1.3;
}
.arrow{
  text-align:center; font-size: 38px; line-height:1; font-weight: 700;
  color:#00e6ff; opacity:0.95;
}
</style>

---

## How to choose a hardened base?
<br/>

Several vendors, incl. BellSoft, Chainguard, Docker

Look for a vendor providing:
- Team focused on OS security, packaging, and compliance
- Signed images and standard attestations
- SLA for patches
- OS and runtime built from source in every image

---

## Dockerfile
<br/>

```dockerfile {none|1,6,11}{maxHeight:'180px'}
FROM bellsoft/hardened-liberica-runtime-container:jdk-25-glibc as builder
WORKDIR /app
ADD . /app/neurowatch
RUN cd neurowatch && ./mvnw -Pproduction clean package

FROM bellsoft/hardened-liberica-runtime-container:jre-25-glibc as optimizer
WORKDIR /app
COPY --from=builder /app/neurowatch/target/neurowatch-*.jar app.jar
RUN java -Djarmode=tools -jar /app/app.jar extract --layers --destination extracted

FROM bellsoft/hardened-liberica-runtime-container:jre-25-glibc

WORKDIR /app
COPY --from=optimizer /app/extracted/dependencies/ ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

---
layout: cover
---

# Quick summary?

---

## Size and update speed
<br/>

1. Multi-stage builds + lightweight base = optimal size
2. Use layers for faster push/pull

---

## Startup time
<br/>

1. AOT Cache - faster 'blood-less' start. Work in progress
2. Native Image - fast start, optimal size and memory consumption, but may be challenging
3. CRaC - fastest start, but may require code refactoring

---

## Facilitated build process
<br/>

1. Buildpacks: no need to write and maintain a Dockerfile
2. Fine-tuning is not always possible or may be complicated

---

## Container image hardening
<br/>

1. Clear CVE management process
2. Provenance data for compliance and audits
3. Lower operational overhead

---
layout: cover
---

# We have created a toolkit for YOUR perfect Docker image.
## And now: Mix & Match!

---

## Thank you for your attention!
<br/>

- BlueSky: @cat-edelveis.bsky.social
- X: cat_edelveis
- LinkedIn: cat-edelveis
- YouTube: @cbrjar