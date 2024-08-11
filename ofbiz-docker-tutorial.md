
# **Docker Session: Creating an Apache OFBiz Docker Image**

## **Introduction**

Welcome to this session on Docker\! Today, we'll be learning how to create a Docker image for Apache OFBiz using a Dockerfile. Docker is a powerful tool for containerization, allowing us to package applications and their dependencies into a single, portable container.

### **What You Will Learn:**

* Understanding the structure of a Dockerfile.  
* Step-by-step breakdown of each instruction in our Dockerfile.  
* Building a Docker image and running a container for Apache OFBiz.

---

## **Part 1: Understanding Docker and Dockerfile**

### **What is Docker?**

Docker is a platform that enables developers to automate the deployment of applications inside lightweight, portable containers. Containers package an application and its dependencies together, ensuring consistency across various environments, whether it's your local machine, a staging server, or production.

### **What is a Dockerfile?**

A Dockerfile is a text document that contains a series of instructions to build a Docker image. The Dockerfile script tells Docker how to assemble the image step by step, including which base image to use, what software to install, and how to configure the application.

---

## **Part 2: Exploring the Dockerfile**

Let's walk through the Dockerfile used to create an image for Apache OFBiz. I'll explain each section in detail.

### **1\. Syntax Definition**

``` 
# syntax=docker/dockerfile:1
```
* **Explanation**: This line defines the syntax version for the Dockerfile. It ensures compatibility with certain Docker features that might require a specific syntax.

### **2\. Base Image Selection**

```
FROM eclipse-temurin:8 AS builder
```
* **Explanation**: The `FROM` instruction specifies the base image for the Docker build. In this case, we're using `eclipse-temurin:8`, which is an OpenJDK (Java) image. The `AS builder` label names this stage as `builder`, which we'll reference later in the Dockerfile.

### **3\. Installing Dependencies**

```  
	RUN apt-get update \
    && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*
```

* **Explanation**: This block updates the package list using `apt-get update`, installs Git (which is needed for building OFBiz), and then cleans up the package cache with `rm -rf /var/lib/apt/lists/*` to reduce the image size.

### **4\. Setting Up the Build Environment**


`WORKDIR /builder`

* **Explanation**: The `WORKDIR` instruction sets the working directory inside the container to `/builder`. All subsequent commands will be executed from this directory, making it easier to manage files.

### **5\. Preparing Gradle for OFBiz Build**
 
```
COPY --chmod=755 gradle/init-gradle-wrapper.sh gradle/
COPY --chmod=755 gradlew .
RUN ["sed", "-i", "s/shasum/sha1sum/g", "gradle/init-gradle-wrapper.sh"]
RUN ["gradle/init-gradle-wrapper.sh"]

# Run gradlew to trigger downloading of the gradle distribution 
RUN --mount=type=cache,id=gradle-cache,sharing=locked,target=/root/.gradle \
    ["./gradlew", "--console", "plain"]
```

* **Explanation**: The `COPY` commands bring necessary Gradle files into the container, with proper permissions using `--chmod=755`. The `RUN` command modifies the Gradle wrapper to replace `shasum` with `sha1sum` for compatibility and then prepares the Gradle environment.

### **6\. Building OFBiz**
```
# Copy all OFBiz sources.
COPY applications/ applications/
COPY config/ config/
COPY framework/ framework/
COPY gradle/ gradle/
COPY lib/ lib/
# We use a regex to match the plugins directory to avoid a build error when the directory doesn't exist.
COPY plugin[s]/ plugins/
COPY themes/ themes/
COPY APACHE2_HEADER build.gradle common.gradle gradle.properties NOTICE settings.gradle .

# Build OFBiz while mounting a gradle cache
RUN --mount=type=cache,id=gradle-cache,sharing=locked,target=/root/.gradle \
    --mount=type=tmpfs,target=runtime/tmp \
    ["./gradlew", "--console", "plain", "distTar"]
```

* **Explanation**: Here, we copy the OFBiz source code into the container. The `RUN` command triggers the OFBiz build using Gradle. The `--mount=type=cache` option caches dependencies to speed up future builds, and `--mount=type=tmpfs` mounts a temporary file system to reduce the disk I/O during the build.

### **7\. Creating the Runtime Base Image**

```
FROM eclipse-temurin:8 AS runtimebase

# xsltproc is used to disable OFBiz components during first run.
RUN apt-get update \
    && apt-get install -y --no-install-recommends xsltproc \
    && rm -rf /var/lib/apt/lists/*

RUN ["useradd", "ofbiz"]

# Create directories used to mount volumes where hooks into the startup process can be placed.
RUN ["mkdir", "--parents", \
    "/docker-entrypoint-hooks/before-config-applied.d", \
    "/docker-entrypoint-hooks/after-config-applied.d", \
    "/docker-entrypoint-hooks/before-data-load.d", \
    "/docker-entrypoint-hooks/after-data-load.d", \
    "/docker-entrypoint-hooks/additional-data.d"]
RUN ["/usr/bin/chown", "-R", "ofbiz:ofbiz", "/docker-entrypoint-hooks" ]

USER ofbiz
WORKDIR /ofbiz
```

* **Explanation**: This section sets up the runtime environment using a new base image. The `RUN` commands install `xsltproc` (used for XML processing) and create a non-root user `ofbiz` for security. The `WORKDIR` is set to `/ofbiz`, the directory where OFBiz will run.

### **8\. Extracting and Preparing OFBiz**

```
# Extract the OFBiz tar distribution created by the builder stage.
RUN --mount=type=bind,from=builder,source=/builder/build/distributions/ofbiz.tar,target=/mnt/ofbiz.tar \
    ["tar", "--extract", "--strip-components=1", "--file=/mnt/ofbiz.tar"]
# Create directories for OFBiz volume mountpoints.
RUN ["mkdir", "/ofbiz/runtime", "/ofbiz/config", "/ofbiz/lib-extra"]
```

* **Explanation**: This command extracts the OFBiz distribution created in the `builder` stage into the runtime environment. The `--strip-components=1` option removes the top-level directory from the archive to simplify the file structure.

### **10\. Configuring Java Runtime**

```
# Append the java runtime version to the OFBiz VERSION file.
COPY --chmod=644 --chown=ofbiz:ofbiz VERSION .
RUN echo '${uiLabelMap.CommonJavaVersion}:' "$(java --version | grep Runtime | sed 's/.*Runtime Environment //; s/ (build.*//;')" >> /ofbiz/VERSION

```

### **10\. Defining Entry Point and Exposed Ports**

```
# Leave executable scripts owned by root and non-writable, addressing sonarcloud rule,
# https://sonarcloud.io/organizations/apache/rules?open=docker%3AS6504&rule_key=docker%3AS6504
COPY --chmod=555 docker/docker-entrypoint.sh docker/send_ofbiz_stop_signal.sh .

COPY --chmod=444 docker/disable-component.xslt .
COPY --chmod=444 docker/templates templates

EXPOSE 8443
EXPOSE 8009
EXPOSE 5005

ENTRYPOINT ["/ofbiz/docker-entrypoint.sh"]
CMD ["bin/ofbiz"]
```

* **Explanation**: The `COPY` commands bring necessary scripts and configuration files into the container, ensuring they have the correct permissions. The `EXPOSE` instructions declare the ports that OFBiz will use (8443 for HTTPS, 8009 for AJP, and 5005 for debugging). Finally, `ENTRYPOINT` sets the main entry point script, and `CMD` specifies the command to run when the container starts.

### **11\. Demo and Runtime Images**

```
# Load demo data before defining volumes. This results in a container image
# that is ready to go for demo purposes.
FROM runtimebase AS demo

USER ofbiz

RUN /ofbiz/bin/ofbiz --load-data
RUN mkdir --parents /ofbiz/runtime/container_state
RUN touch /ofbiz/runtime/container_state/data_loaded
RUN touch /ofbiz/runtime/container_state/admin_loaded
RUN touch /ofbiz/runtime/container_state/db_config_applied

VOLUME ["/docker-entrypoint-hooks"]
VOLUME ["/ofbiz/config", "/ofbiz/runtime", "/ofbiz/lib-extra"]
```

* **Explanation**: The `demo` stage is for creating a pre-loaded OFBiz container ready for demo purposes. It loads demo data and sets up necessary files to track the container state. The `VOLUME` instructions define directories that can be mounted as volumes.

dockerfile  
Copy code  
`FROM runtimebase AS runtime`  
`USER ofbiz`  
`VOLUME ["/docker-entrypoint-hooks"]`  
`VOLUME ["/ofbiz/config", "/ofbiz/runtime", "/ofbiz/lib-extra"]`

* **Explanation**: The `runtime` stage creates a base OFBiz runtime image without pre-loaded data, making it more versatile for production use. It also defines volumes for configuration, runtime data, and additional libraries.

---

## **Part 3: Building and Running the Docker Image**

### **Building the Docker Image**

To build the Docker image, run the following command in your terminal:


`docker build -t ofbiz-image .`

This command tells Docker to build an image named `ofbiz-image` using the Dockerfile in the current directory (`.`).

### **Running a Container**

Once the image is built, you can run a container with the following command:

`docker run -d -p 8443:8443 -p 8009:8009 -p 5005:5005 --name ofbiz-container ofbiz-image`

* **Explanation**: This command runs the OFBiz container in detached mode (`-d`), mapping the necessary ports from the container to the host machine (`-p`). The container is named `ofbiz-container`, and it uses the `ofbiz-image`.

### **Accessing OFBiz**

You can now access the OFBiz application by navigating to `https://localhost:8443` in your web browser.

---

## **Conclusion**

Thank you for attending this session on Docker and Apache OFBiz\! Today, we've learned how to create a Docker image for OFBiz using a Dockerfile, from understanding its structure to building and running the final container. I hope you found this session informative and that you'll be able to use Docker effectively in your projects.

---
