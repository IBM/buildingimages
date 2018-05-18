# Building Container Images in IBM Cloud

content creation around image building for containers

# Prerequisites

Before you get started you'll need an IBM Cloud account and need to
install
the
[IBM Cloud Development Tools](https://github.com/IBM-Cloud/ibm-cloud-developer-tools).

You should also clone
the [guestbook repository](https://github.com/IBM/guestbook) as all
image builds will be based on that.

Once those are installed, you'll be able to complete the commands
below.

# Basics of Image Building

Kubernetes uses standard container images to deploy applications.

From the application's perspective, a container image is a single
filesystem that is created before the application is started.

From a creation perspective, container images are layers. These layers
are built up using a Docker file. Lets look at the example guestbook
application Docker file and pull it apart.

```
FROM golang as builder
RUN go get github.com/codegangsta/negroni
RUN go get github.com/gorilla/mux github.com/xyproto/simpleredis
COPY main.go .
RUN go build main.go

FROM busybox:ubuntu-14.04

COPY --from=builder /go//main /app/guestbook

ADD public/index.html /app/public/index.html
ADD public/script.js /app/public/script.js
ADD public/style.css /app/public/style.css

WORKDIR /app
CMD ["./guestbook"]
EXPOSE 3000
```

Docker files specific a set of commands, at each command we build a
new layer.

FROM - pulls a layer from a remote repository. The default
repository is the docker image registry. This could be a complete OS
image, or some partial image that used to do something small. In this
example we pull in 2 different base images. One is a named layer for
the golang build environment, so that it can be referenced later. The
second one is the base image with just enough operating system to
provide debug facilities inside the image.

RUN - executes a command, all output files go into the layer. There
are a few `go get` commands that pull in libraries for the main
application. Later the main.go is compiled using this.

COPY - copy a file from the local filesystem into the container. This
can work on a set of files.

WORKDIR - set the working directory for the entrypoint

CMD - what command should be run in this container if no exec is
passed into it. CMD means that the command run is replaceable. There
is another stanza `ENTRYPOINT` which is not replaceable.

EXPOSE - open up a specific port from this container to the
infrastructure. Without this no network traffic will be allowed into
the container.

These commands, plus many more are documented on the [Docker reference
site](https://docs.docker.com/engine/reference/builder/#cmd).

## Best Practices for Building Images

There are a number of best practices when building images.

### Clean up everything you can

Whatever files are created in a layer are stored in the intermediate
layer. Sometimes what gets added in each layer ends up as a
surprise. For instance, when install Linux packages with `apt`, not
only is the software installed, but copies of the packages are stored
in `/var/cache/apt/archives`. This can add significant bloat to your
images, which slows down image upload/download times and thus
container start times.

When using standard install tools, be sure to include clean up
commands as part of the run command. For instance:

```
RUN apt-get update && apt-get install -y python3 python3-pip && apt-get clean
```

Which updates the repos, installs packages, and deletes downloaded
archive files.

# Publishing to IBM Cloud

You can build images with docker and publish them publicly to docker
hub. However, if you are going to be using these images inside of IBM
cloud it's best practice to publish them into the IBM Cloud Container
Registry. This makes the images much more network local to where they
run, and makes them private to your IBM Cloud account.

## Creating your registry

Before you can upload images to the IBM Cloud you must first create a
registry namespace. Namespaces are just used to help you organize your
account. You can have many namespaces in your account, creating one
per project is a useful way of staying organized.

```
> bx cr namespace-add guestbook
```

This will create a registry url of `registry.ng.bluemix.net/guestbook`
that you can use for uploading images.

## Building using bx client

The bx client can be used to build images using the local docker
system. When building with the bx client all the authentication to the
IBM Cloud Container Registry is handled behind the scenes.

We will want to provide a meaningful name for the image, as well as
some kind of version.

```
> bx cr build --tag registry.ng.bluemix.net/guestbook/gbv1:2018-05-18-1 guestbook
Sending build context to Docker daemon  16.38kB
Step 1/13 : FROM golang as builder
 ---> 6b369f7eed80
Step 2/13 : RUN go get github.com/codegangsta/negroni
 ---> Using cache
 ---> 6235665371cf
Step 3/13 : RUN go get github.com/gorilla/mux github.com/xyproto/simpleredis
 ---> Using cache
 ---> c7b784b25fd7
Step 4/13 : COPY main.go .
 ---> 22bb5236a85b
Step 5/13 : RUN go build main.go
 ---> Running in 7809ad57b6b4
 ---> 7b70b540d7d4
Removing intermediate container 7809ad57b6b4
Step 6/13 : FROM busybox:ubuntu-14.04
 ---> d16744963217
Step 7/13 : COPY --from=builder /go//main /app/guestbook
 ---> 8b7eb0124777
Step 8/13 : ADD public/index.html /app/public/index.html
 ---> 9d763716676c
Step 9/13 : ADD public/script.js /app/public/script.js
 ---> 353bca474993
Step 10/13 : ADD public/style.css /app/public/style.css
 ---> 81819bed4684
Step 11/13 : WORKDIR /app
 ---> 1a99524a116a
Removing intermediate container 020c16388591
Step 12/13 : CMD ./guestbook
 ---> Running in bdb88f981da7
 ---> fc76ec26b945
Removing intermediate container bdb88f981da7
Step 13/13 : EXPOSE 3000
 ---> Running in b04f65bdedad
 ---> b63df6f0716e
Removing intermediate container b04f65bdedad
Successfully built b63df6f0716e
Successfully tagged registry.ng.bluemix.net/guestbook/gbv1:2018-05-18-1
The push refers to a repository [registry.ng.bluemix.net/guestbook/gbv1]
cfd8baa1be3c: Pushed
3d8c58449645: Pushed
98a83cd4e753: Pushed
9b5fbbfcbb1d: Pushed
5f70bf18a086: Layer already exists
5dbcf0efe4f2: Layer already exists
4: digest: sha256:cca8cdb32ed616ae13087a9b557696896505bb36241c0f24588987094a687000 size: 1772

OK
```

This builds and pushes all the relevant layers.

The `--tag` field gives the image a name and version, separated by the
`:`. When using the IBM Cloud Container Registry the name is going to
include the full URL to that image.

The version portion can be anything that's meaningful to you. The
important thing is that tags do not get reused. If you rebuild an
image, make sure to change the version before uploading. Many
kubernetes primitives, such as deployments, will perform a graceful
rolling upgrade when you specify a new image version. If image
versions get reused it hampers the effectiveness of container
orchestration systems.

# Retrieving from IBM Cloud

IBM Cloud images are included in kubernetes configuration in the same
way as images from docker hub. The only difference is the full url is
needed as it's the non default image repository.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
  labels:
    app: guestbook
spec:
  replicas: 3
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
      - name: guestbook
        image: registry.ng.bluemix.net/guestbook/gbv1:2018-05-18-1
        ports:
        - name: http-server
          containerPort: 3000
```

# Vulnerability Advisor

An additional feature of using the IBM Container Service is IBM
Vulnerabiliy Advisor. After the image is uploaded it is scanned by a
set of tools that look for common security issues. These scans are
also run regularly as new CVEs are issued.

In the case of the guestbook image it's an extremely small
microservice that doesn't include a traditional OS base, so there is
not much VA can do with it. If we look at a set of images that are
based on an Ubuntu Linux base we can see it in action.

```
> bx cr images --restrict ny-power
Listing images...

REPOSITORY                                            NAMESPACE   TAG   DIGEST         CREATED        SIZE     VULNERABILITY STATUS
registry.ng.bluemix.net/ny-power/ny-power-api         ny-power    5     16041dfa9da4   1 month ago    203 MB   Vulnerable
registry.ng.bluemix.net/ny-power/ny-power-base        ny-power    7     56ef5e28ad5e   1 month ago    208 MB   Vulnerable
registry.ng.bluemix.net/ny-power/ny-power-ibm-cloud   ny-power    5     3e5fbcddf92c   1 month ago    107 MB   OK
registry.ng.bluemix.net/ny-power/ny-power-ibm-cloud   ny-power    6     e735f31e9e81   22 hours ago   107 MB   OK
registry.ng.bluemix.net/ny-power/ny-power-influxdb    ny-power    3     596642b949a6   1 month ago    84 MB    Vulnerable
registry.ng.bluemix.net/ny-power/ny-power-mqtt        ny-power    4     b94db277bbd5   1 month ago    88 MB    OK
registry.ng.bluemix.net/ny-power/ny-power-web         ny-power    10    e2b7fd0b3a44   1 month ago    203 MB   Vulnerable
registry.ng.bluemix.net/ny-power/ny-power-web         ny-power    11    dc56770012f8   1 month ago    203 MB   Vulnerable
registry.ng.bluemix.net/ny-power/ny-power-web         ny-power    12    25bee418debe   1 month ago    203 MB   Vulnerable
registry.ng.bluemix.net/ny-power/ny-power-web         ny-power    15    166225785d06   1 month ago    203 MB   Vulnerable
registry.ng.bluemix.net/ny-power/ny-power-web         ny-power    16    3b861213f07e   23 hours ago   203 MB   OK
registry.ng.bluemix.net/ny-power/ny-power-web         ny-power    7     419f2cf4738a   1 month ago    203 MB   Vulnerable
registry.ng.bluemix.net/ny-power/ny-power-web         ny-power    8     0481120a635d   1 month ago    203 MB   Vulnerable
registry.ng.bluemix.net/ny-power/ny-power-web         ny-power    9     81cdb370bd74   1 month ago    203 MB   Vulnerable

```

We can drill down into an image in question:

```
> bx cr va registry.ng.bluemix.net/ny-power/ny-power-base:7 --extended
Checking vulnerabilities for 'registry.ng.bluemix.net/ny-power/ny-power-base:7'...

Image 'registry.ng.bluemix.net/ny-power/ny-power-base:7' was last scanned on Mon May 14 23:38:51 UTC 2018.
The scan results show the image should be deployed with CAUTION.

VULNERABLE PACKAGES FOUND
=========================

perl
   Corrective action: Upgrade to perl 5.22.1-9ubuntu0.3

   FIX SUMMARY                                OFFICIAL NOTICE                        CVE ID
-  Several security issues were fixed in      http://www.ubuntu.com/usn/usn-3625-1   CVE-2015-8853,CVE-2016-6185,CVE-2017-6512,CVE-2018-6797,CVE-2018-6798,CVE-2018-6913
   Perl.


CONFIGURATION ISSUES FOUND
==========================

SECURITY PRACTICE                                                     CORRECTIVE ACTION
PASS_MAX_DAYS must be set to 90 days                                  Maximum password age must be set to 90 days.
Minimum password length not specified in /etc/pam.d/common-password   Minimum password length must be 8.
PASS_MIN_DAYS must be set to 1                                        Minimum days that must elapse between user-initiated password changes should be 1.
File /var/log/faillog does not exist                                  Permission check of /var/log/faillog
file /var/log/faillog must exist if pam_tally2.so is not being used   faillog file checking

OK
```

This scans inside the image and found that the standard perl package
that was part of the base image has a security notice associated with
it, as well as the CVEs that references. Rebuilding this image with
updates applied would address this issue.

It also scans for potential configuration issues. The configuration
issues listed here aren't really applicable to this image as there is
no login services to the container.
