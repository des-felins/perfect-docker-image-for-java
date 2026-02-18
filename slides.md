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

# The Java Container Image Playbook: Design for Your Goals 

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

<br/>
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

## Result: 984MB
<br/>

```plain {none|19,18|9-17|8|5,4}{maxHeight:'300px'}
IMAGE          CREATED         CREATED BY                                      SIZE      COMMENT
666431581cee   4 minutes ago   ENTRYPOINT ["/bin/sh" "-c" "java -jar /app/n…   0B        buildkit.dockerfile.v0
<missing>      4 minutes ago   EXPOSE map[8081/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      4 minutes ago   RUN /bin/sh -c cd neurowatch && ./mvnw -Ppro…   355MB     buildkit.dockerfile.v0
<missing>      6 minutes ago   COPY . /app/neurowatch # buildkit               310MB     buildkit.dockerfile.v0
<missing>      6 minutes ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>      2 months ago    /bin/sh -c #(nop)  ENV JAVA_HOME=/usr/lib/jv…   0B        
<missing>      2 months ago    |8 BASE_URL=https://download.bell-sw.com/jav…   131MB     
<missing>      2 months ago    /bin/sh -c #(nop)  ARG BASE_URL=https://down…   0B        
<missing>      2 months ago    /bin/sh -c #(nop)  ARG LIBERICA_ROOT=/usr/li…   0B        
<missing>      2 months ago    /bin/sh -c #(nop)  ARG LIBERICA_BUILD=11        0B        
<missing>      2 months ago    /bin/sh -c #(nop)  ARG LIBERICA_VERSION=25.0…   0B        
<missing>      2 months ago    /bin/sh -c #(nop)  ARG LIBERICA_JVM_DIR=/usr…   0B        
<missing>      2 months ago    /bin/sh -c #(nop)  ARG LIBERICA_GENERATE_CDS…   0B        
<missing>      2 months ago    /bin/sh -c #(nop)  ARG LIBERICA_VM=server       0B        
<missing>      2 months ago    /bin/sh -c #(nop)  ARG LIBERICA_IMAGE_VARIAN…   0B        
<missing>      2 months ago    /bin/sh -c #(nop)  ENV LANG=en_US.UTF-8 LANG…   0B        
<missing>      2 months ago    /bin/sh -c apt-get update                   …   48.3MB    
<missing>      2 months ago    # debian.sh --arch 'arm64' out/ 'bookworm' '…   139MB     debuerreotype 0.16
```


---

## 984MB - should we worry about it?
<br/>

- Longer push - longer updates
- Longer pull - slower scaling
- Dozens commits/day, 984MB each: we wil quickly run out of space for all these images

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

## Result:  392MB
<br/>

```plain {none|7,15,16|4}{maxHeight:'300px'}
IMAGE          CREATED              CREATED BY                                      SIZE      COMMENT
2b665d1b1293   About a minute ago   ENTRYPOINT ["java" "-jar" "/app/app.jar"]       0B        buildkit.dockerfile.v0
<missing>      About a minute ago   EXPOSE map[8080/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      About a minute ago   COPY /app/neurowatch/target/neurowatch-*.jar…   74.1MB    buildkit.dockerfile.v0
<missing>      3 minutes ago        WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>      2 months ago         /bin/sh -c #(nop)  ENV JAVA_HOME=/usr/lib/jv…   0B        
<missing>      2 months ago         |6 LIBERICA_BUILD=11 LIBERICA_GENERATE_CDS=f…   131MB     
<missing>      2 months ago         /bin/sh -c #(nop)  ARG LIBERICA_GENERATE_CDS…   0B        
<missing>      2 months ago         /bin/sh -c #(nop)  ARG LIBERICA_USE_LITE=1      0B        
<missing>      2 months ago         /bin/sh -c #(nop)  ARG LIBERICA_ROOT=/usr/li…   0B        
<missing>      2 months ago         /bin/sh -c #(nop)  ARG LIBERICA_VARIANT=jre     0B        
<missing>      2 months ago         /bin/sh -c #(nop)  ARG LIBERICA_BUILD=11        0B        
<missing>      2 months ago         /bin/sh -c #(nop)  ARG LIBERICA_VERSION=25.0…   0B        
<missing>      2 months ago         /bin/sh -c #(nop)  ENV LANG=en_US.UTF-8 LANG…   0B        
<missing>      2 months ago         /bin/sh -c apt-get update                   …   48.3MB    
<missing>      2 months ago         # debian.sh --arch 'arm64' out/ 'bookworm' '…   139MB     debuerreotype 0.16
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

```docker{none|1|2|7}{maxHeight:'300px'}
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

## Result: 206MB
<br/>

```plain{none|6,2}{maxHeight:'300px'}
IMAGE          CREATED              CREATED BY                                      SIZE      COMMENT
e46b11d81a8d   About a minute ago   [stage-1 3/3] COPY --from=builder /app/neuro…   74.1MB    buildkit.exporter.image.v0
<missing>      About a minute ago   ENTRYPOINT ["java" "-jar" "/app/app.jar"]       0B        buildkit.dockerfile.v0
<missing>      About a minute ago   EXPOSE map[8080/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      21 hours ago         COPY /app/neurowatch/target/neurowatch-*.jar…   0B        buildkit.dockerfile.v0
<missing>      21 hours ago         WORKDIR /app                                    132MB     buildkit.dockerfile.v0
```

---
layout: image
image: size.svg
---

---
layout: cover
---

# But! The app layer has the same size (74MB)

---

## What are 74MB?
<br/>

Smallest change, and we push/pull 74MB

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

## Result: 206MB
<br/>

```plain{none|5-8|2}{maxHeight:'180px'}
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
8a242e10320e   57 seconds ago   [stage-2 6/6] COPY --from=optimizer /app/ext…   1.21MB    buildkit.exporter.image.v0
<missing>      57 seconds ago   ENTRYPOINT ["java" "-jar" "/app/app.jar"]       0B        buildkit.dockerfile.v0
<missing>      57 seconds ago   EXPOSE map[8080/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      57 seconds ago   COPY /app/extracted/application/ ./ # buildk…   0B        buildkit.dockerfile.v0
<missing>      57 seconds ago   COPY /app/extracted/snapshot-dependencies/ .…   0B        buildkit.dockerfile.v0
<missing>      57 seconds ago   COPY /app/extracted/spring-boot-loader/ ./ #…   72.7MB    buildkit.dockerfile.v0
<missing>      22 hours ago     COPY /app/extracted/dependencies/ ./ # build…   0B        buildkit.dockerfile.v0
<missing>      22 hours ago     WORKDIR /app                                    132MB     buildkit.dockerfile.v0
```

---

## What exactly are we optimizing?
<br/>

push/pull layer size!

The size of the actively updated layer is now 1.21MB!

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

```bash {1|2|3}
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

## Result: 403MB
<br/>

Peeking under the hood:

```plain{none|1,2|4-10|11|13-16|33}{maxHeight:'180px'}
[INFO]  > Pulling builder image 'docker.io/paketobuildpacks/builder-noble-java-tiny:latest' 100%
[INFO]  > Pulling run image 'docker.io/paketobuildpacks/ubuntu-noble-run-tiny:0.0.47' for platform 'linux/arm64' 100%
[INFO]  > Running creator
[INFO]     [creator]     6 of 26 buildpacks participating
[INFO]     [creator]     paketo-buildpacks/ca-certificates   3.11.0
[INFO]     [creator]     paketo-buildpacks/bellsoft-liberica 11.5.1
[INFO]     [creator]     paketo-buildpacks/syft              2.26.0
[INFO]     [creator]     paketo-buildpacks/executable-jar    6.14.0
[INFO]     [creator]     paketo-buildpacks/dist-zip          5.11.0
[INFO]     [creator]     paketo-buildpacks/spring-boot       5.35.0
[INFO]     [creator]       BellSoft Liberica JRE 25.0.1: Contributing to layer
[INFO]     [creator]       Creating slices from layers index
[INFO]     [creator]         dependencies (69.3 MB)
[INFO]     [creator]         spring-boot-loader (457.5 KB)
[INFO]     [creator]         snapshot-dependencies (0.0 B)
[INFO]     [creator]         application (2.9 MB)
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
<br/>
<v-click at="7">Native images with buildpacks also possible</v-click>

---

# Alpaquita buildpacks for Spring Boot
<br/>

```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <builder>bellsoft/buildpacks.builder:glibc</builder>
        </image>
    </configuration>
</plugin>
```

<br/>

Image size: 215MB (shell is there)

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

<br/>

<v-click>Right now, our app starts in ~3.7s</v-click>

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

## AOT Cache with a Spring app
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

## AOT Cache with a Spring app
<br/>

```properties
spring.cloud.stream.enabled=false
spring.data.mongodb.auto-index-creation=false
app.create-test-users=false
vaadin.launch-browser=false
```

---

## AOT Cache with a Spring app
<br/>

```dockerfile {none|12|22-26|15}{maxHeight:'250px'}
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

RUN java -Dspring.aot.enabled=true \
         -Dspring.profiles.active=aot \
         -XX:AOTCacheOutput=app.aot \
         -Dspring.context.exit=onRefresh \
         -jar /app/app.jar
```

---

## Price: 355MB
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
Start is 1.4s

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

```dockerfile {none|1|5|6|8|10,11}{maxHeight:'180px'}
FROM bellsoft/liberica-native-image-kit-container:jdk-25-nik-25-musl as builder
RUN apk add --no-cache nodejs npm
WORKDIR /app
ADD . /app/neurowatch
ENV SPRING_PROFILES_ACTIVE=native
RUN cd neurowatch && ./mvnw -Pproduction -Pnative native:compile

FROM bellsoft/alpaquita-linux-base:musl
WORKDIR /app
ENTRYPOINT ["/app/app"]
COPY --from=builder /app/neurowatch/target/native/neurowatch /app/app
```

<br/>
application-native.properties

```properties
spring.data.mongodb.auto-index-creation=false
spring.cloud.stream.bindings.logConsumer-in-0.consumer.autoStartup=false
vaadin.launch-browser=false
```

---

## Result: start in 0.7s
<br/>

- Based on Linux only
- Start at peak performance
- Image size: 225MB 

But:
- Need to rebuild the whole image every time

---
layout: image
image: start.svg
---


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

```dockerfile {none|1,6,11}{maxHeight:'400px'}
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
4. Immutable images

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