---
layout: post
title:  "Unveiling Secrets: A Dive Into Passwords and Sensitive Data within Docker Images"
excerpt_separator: <!--more-->
---

*NOTE*: This is an ad for a CLI tool I wrote call [Dockerleaks](https://github.com/bthuilot/dockerleaks), so be sure to check it out on [my GitHub](https://github.com/bthuilot)
  
# Unveiling Secrets: 
<br/>
## A dive into passwords and sensitive sata within built Docker images
<hr/>
<br/>
Over the past few years, the increasing importance of CI/CD security has led to a significant rise in the development and utilization of secret detection tools. From TruffleHog to GitLeaks, these instruments play a crucial role in protecting sensitive data from exposure, thereby ensuring the security and integrity of our digital spaces. These powerful tools, predominantly designed for scanning codebases and repositories, search diligently for sensitive information that may inadvertently be embedded within the digital fabric.

However, while these tools are adept at unraveling secrets in code repositories, there is a noticeable gap (at least in my opinion) in their abilities: Docker images. Despite the ubiquity of Docker in modern DevOps environments, very few secret detection tools offer support for scanning Docker images. As Docker's popularity continues to surge, and the number of Docker images multiplies, the urgency to address this vulnerability increases exponentially.

In this blog post, I'm going to discuss the common ways that secrest end up in built docker images, and what tools exist to combat this (one of them maybe written by yours truly).

<!--more-->

## How are secrets leaked?

The main way developes leak secrets in docker are:
- **Environment variables**: Set use the `ENV` instruction in a Dockerfile
- **Build arguments**: Set using the `ARG` instruction in a Dockerfile, and supplied by the user
- **File content**: Contents of a file that is contained within a built image's layer.

<br />
### Environment variables

Environment variables are the most obviousy place to find leaked credentials, since they are configured statically or from the result of a build argument. Environment variables set to a static credential during build time are inspectable by the docker agent. 

In the example below, we create a docker image with a password written in plaintext

```Dockerfile
FROM scratch

# This should never be done!
ENV MY_PASSWORD=notverysecure
```

after building this image (`docker build -t env-leak .`),, we can inspect all environment variables and see our secret.

```bash
docker  inspect --type image env-leak --format='\{\{.Config.Env}}'
# [... MY_PASSWORD=notverysecure]
```

This is obviously extremely bad, since if this image was published to a public registry like [Docker Hub](https://hub.docker.com/) it would be viewable by everyone!

<br/> 
### Build arguments

Build arguments are an area that many developers do not realize is **not a secure method for loading in secrets during build time**.
Build arguments are meant to serve as a way to inject data into an image during buildtime (i.e. version numbers, build date, etc.) but instead tends to be used for build time secrets (i.e. git/artifact server access tokens).

In the example below, we create a docker image with a build argument for a git access token that is used to clone down the source repository inside the docker image.
However, even in the case where this build argument isn't used, the secret will be leaked. So long as there is `RUN` command after the build argument, the argument will be visible in an inspected image


```Dockerfile
FROM alpine:latest

ARG GIT_ACCESS_TOKEN

# ... Do something with git token
RUN git clone https://docker:$GIT_ACCESS_TOKEN@example-repo.com

RUN ["echo", "hello"]
```

after building this image with a build argument (`docker build --build-arg GIT_ACCESS_TOKEN=leaked -t build-arg-leak .`),
we can inspect the history of an image and see that the build argument is given as an enviornment variable to our `RUN` command in plain-text!

```bash
docker history build-arg-leak
# IMAGE          CREATED         CREATED BY                                      SIZE      COMMENT
#6e558799f7ba   4 seconds ago   |1 GIT_ACCESS_TOKEN=leaked /bin/sh -c echo "…   0B
#996dea2565f0   5 seconds ago   /bin/sh -c #(nop)  ARG GIT_ACCESS_TOKEN         0B
#... Other layers from base image
```

As stated above this is very bad for images pushed to a public registry, since now that build-time credential can be recovered by anyone.

<br/>
### Files

'Files' is pretty self explanitory, any public image can be pulled down and have their file-system accessed so any sensitive information contained within a file on the file-system is publically viewable.
This can come from either copying in data that exists on your local filesytem or a result of any tools that run on a docker image during build that produces sensitive files.
Files and folders like `terraform.tfstate` or `.git/` can be extracted to find more secrets.

In the example below, we have a file system with 3 files: `main.go`, `.env` and `Dockerfile`.
Their contents are shown below:

#### `main.go`

```golang
// Contents of main.go
package main

import (
	"fmt"
	"os"
)

func main() {
	var userApiKey string
	
	apiKey := os.Getenv("MY_API_KEY")
	fmt.Print("Enter API Key: ")
	fmt.Scan(&userApiKey)
	
	if apiKey == userApiKey {
		fmt.Println("Correct!")
	} else {
		fmt.Println("Wrong!!")
		os.Exit(1)
	}
}
```

#### `.env`

```bash
# .env contents
MY_API_KEY=testing
```

#### `Dockerfile`

```Dockerfile
# Dockerfile contents
FROM golang:latest

WORKDIR /app
# Copy all files into docker images
# this really should only 'main.go'
COPY . .

RUN go build -o main main.go 
ENTRYPOINT ["/app/main"]
```

The following bash snippet will show that an image built from the following files will contain the leaked `.env` file.

```bash
# run in the same directory as the files listed above
docker build -t env-leak .

docker run --rm -it --entrypoint /bin/bash env-leak -c "cat /app/.env"
# MY_API_KEY=testing
```

This happens because line 7 of the Dockerfile copies all contents of the current directory into the docker images.
The remediation for this would be to be carefully about your `COPY` statements, and leverage [`.dockerignore` files ](https://docs.docker.com/build/building/context/#dockerignore-files) (similar to a .gitignore file). In the example above, the user could add the line `.env` to your `.dockerignore` file, or changing the copy to `COPY main.go /app/main.go` to only include the source code.

Additionally the user should be careful of the output of any commands (i.e. running a build step that produces artifacts with sensitive values).
Leveraging [Multi-Stage builds](https://docs.docker.com/build/building/multi-stage/) is a good way to ensure that only the nessecary artifacts are copied to the released docker image after build.

Refactoring the `Dockerfile` above to leverage multi build would look like the following

```Dockerfile
# Dockerfile contents
FROM golang:latest as build

# Here we will copy everything in (including the .env)
# and build the application in the 'build' stage to /build/main
WORKDIR /build
COPY . .
RUN go build -o main main.go 

# Now we change the stage to the 'release' stage
# Since this is the last stage, it will be the one that will be published
FROM golang:latest as release

# Copy only the `/build/main` 
COPY --from=build /build/main /app/main

ENTRYPOINT ["/app/main"]
```


<br/>
## How can secret data be loaded and used securely?

*NOTE*: this requires [Docker BuildKit](https://docs.docker.com/build/buildkit/)

 Docker provides a feature called "Docker Secrets" primarily aimed at securely managing secrets in Docker Swarm services. Additionally, since Docker 18.09, the concept of "BuildKit" has been introduced to support using secrets during the build process

Docker has its own version of secrets built in, it works by mounting a volume that contains the secret as the content of a file that can be used during specified `RUN` commands, then unmounted after the command completes.

### During build-time

BuildKit allows for secrets to be accessed surely during build time, without revealing the secret plain text in the images history.

in the example below, we access a secret mount to perform the git clone.

```Dockerfile
FROM alpine:latest

# Use a private tokent to clone a repo into the docker images
# We mount the secret 'gitlab_token', meaning it the secret value will
# be stored as the contents of a file with the same name in `/var/secrets`
RUN --mount=type=secret,id=git_token git clone https://docker:$(cat /var/secrets/git_token)@example-repo.com
```
To then build this docker image with a secret from an environment variable or a file on the local system,
the following command-line parameter should be added (*note*: the `id` must match between the `RUN` line and the CLI parameter) :

```bash
# For a secret for a local file
--secret id=gitlab_token,src=$HOME/.git-token

# For an environment variables
--secret id=git_token,env=GIT_ACCESS_TOKEN
```

### During run-time

Just like described in the [`During build-time`](#during-build-time) section above, 
secrets can be leveraged as well for runtime.

This feature is beyond the scope of this post, but be sure to read about secret use within docker [here](https://docs.docker.com/engine/swarm/secrets/)

## How can I know if my secret has been leaked?

And now a word from our sponsor... me.

As a side project I've written [Dockerleaks](https://github.com/bthuilot/dockerleaks),
a CLI tool that will connect to the docker daemon and attempt to search the built image for
secret regular expression matches and high entropy strings. Be sure to check it out on my GitHub!

