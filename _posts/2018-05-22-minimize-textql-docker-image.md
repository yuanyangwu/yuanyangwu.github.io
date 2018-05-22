---
layout: post
title: Minimize TextQL docker image by 93%
tags: docker textql
comment: true
image: 
published: true
---
# Minimize TextQL docker image by 93%

The document explains how to reduce the TextQL docker image by 93%. That is from 300MB to 19MB.

[TextQL](https://github.com/dinedal/textql) is Golang-based console command, which can easily execute SQL against structured text like CSV or TSV.

The console command binary size is ~9MB. There are 2 public related docker images. But both are much larger than the console command binary.

- [danstreeter/textql](https://hub.docker.com/r/danstreeter/textql/tags/), 314MB
- [jpierre03/textql](https://hub.docker.com/r/jpierre03/textql/tags/), 260MB

## Default Dockerfile

TextQL default Dockerfile is

```text
FROM golang:1.10

WORKDIR /go/src/app
COPY . .

RUN go get -d -v ./...
RUN go install -v ./...

WORKDIR /tmp

ENTRYPOINT ["textql"]
```

Build docker image

```console
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ sudo docker build -t textql .
```

Local docker images show

- golang:1.10 size is 794MB (>301MB on <https://hub.docker.com/r/library/golang/tags/>)
- textql image size is 887MB. The increased size is mainly build intermediate data because final binary is ~9MB.

```console
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ sudo docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
textql                                     latest              8a486bb61bf2        2 hours ago         887 MB
golang                                     1.10                6b369f7eed80        2 weeks ago         794 MB
```

## Optimization 1: Reduce image by golang:1.10-alpine3.7

golang:1.10-alpine3.7 is based on smallest alpine docker image. Its local size is 376MB (>116MB on <https://hub.docker.com/r/library/golang/tags/>). It is half of golang:1.0.

```console
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ sudo docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
golang                                     1.10                6b369f7eed80        2 weeks ago         794 MB
golang                                     1.10-alpine3.7      05fe62871090        2 weeks ago         376 MB
```

Build docker image with Dockerfile using golang:1.10-alpine3.7

note: Dockerfile add "apk add" commands to install git and gcc.

```console
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ cat Dockerfile.golang-alpine
FROM golang:1.10-alpine3.7

# "build-base" for gcc
RUN apk update && apk add git && apk add build-base
WORKDIR /go/src/app
COPY . .

RUN go get -d -v ./...
RUN go install -v ./...

WORKDIR /tmp

ENTRYPOINT ["textql"]
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ sudo docker build -f Dockerfile.golang-alpine -t textql:golang-alpine .
```

New image "textql:golang-alpine" is 626MB, which is a little smaller than default image. But the size is not that good because it is much bigger than "golang:1.10-alpine3.7".

```console
REPOSITORY                                 TAG                 IMAGE ID            CREATED              SIZE
textql                                     golang-alpine       e260eab8a790        About a minute ago   626 MB
textql                                     latest              8a486bb61bf2        3 hours ago          887 MB
golang                                     1.10                6b369f7eed80        2 weeks ago          794 MB
golang                                     1.10-alpine3.7      05fe62871090        2 weeks ago          376 MB
```

## Optimization 2: Reduce image by multi-stage builds

Optimization 1's image size is big for 2 reasons

- Base image is big. "golang:1.10-alpine3.7" is for build, most of which is not needed for production execution.
- Build intermediate data is needed for production execution.

The further optimization is to build in 2 stages.

- Stage 1: same as Optimization 1, build with base image "golang:1.10-alpine3.7"
- Stage 2: create new image with base image "alpine:3.7", which only includes stage 1's binary

Dockerfile best practice recommends this way in [Use multi-stage builds](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#use-multi-stage-builds). With Docker 17.05 or higher, you can use multi-stage builds to drastically reduce the size of your final image.

Dockerfile using multi-stage builds. The 1st image is named as "build". The 2nd image copies binary from "build".

```console
vagrant@ubuntu-xenial:~/textql$ cat Dockerfile.alpine
FROM golang:1.10-alpine3.7 AS build

# "build-base" for gcc
RUN apk update && apk add git && apk add build-base
WORKDIR /go/src/app
COPY . .

RUN go get -d -v ./...
RUN go install -v ./...

FROM alpine:3.7
COPY --from=build /go/bin/textql /usr/bin
WORKDIR /tmp
ENTRYPOINT ["textql"]
```

Build docker image

Note: if your docker is old, please install docker-ce with instructions on <https://docs.docker.com/install/linux/docker-ce/ubuntu/>

```console
vagrant@ubuntu-xenial:~/textql$ sudo docker build -f Dockerfile.alpine -t textql:alpine .
```

Bingo! New image "textql:alpine" is 13.9MB. It is only 5% of current image size.

note: Image b289c7251afb is the intermediate image for building textql binary.

```console
vagrant@ubuntu-xenial:~/textql$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
textql              alpine              23086cf60fec        About a minute ago   13.9MB
<none>              <none>              b289c7251afb        About a minute ago   584MB
golang              1.10-alpine3.7      05fe62871090        2 weeks ago          376MB
alpine              3.7                 3fd9065eaf02        4 months ago         4.15MB
```

Test new image

```console
vagrant@ubuntu-xenial:~/textql$ cat test.csv
first_name,last_name,ssn
John,Barry,123456
Kathy,Smith,687987
Bob,McCornick,3979870
vagrant@ubuntu-xenial:~/textql$ sudo docker run --rm -it -v $(pwd):/tmp textql:alpine -header -sql 'SELECT * FROM test' test.csv
John,Barry,123456
Kathy,Smith,687987
Bob,McCornick,3979870
```

## Bugfix: Console requested but unable to locate `sqlite3` program

The new image fails to run with "-console". Same problem is reported on current Dockerfile (<https://github.com/dinedal/textql/issues/93>). The root cause is that the docker image does NOT include sqlite3.

```console
vagrant@ubuntu-xenial:~/textql$ sudo docker run --rm -it -v $(pwd):/tmp textql:alpine -header -console test.csv
2018/05/22 08:39:49 Console requested but unable to locate `sqlite3` program on $PATH
```

Update Dockerfile to add sqlite3 to final docker image

```console
vagrant@ubuntu-xenial:~/textql$ cat Dockerfile.alpine
FROM golang:1.10-alpine3.7 AS build

# "build-base" for gcc
RUN apk update && apk add git && apk add build-base
WORKDIR /go/src/app
COPY . .

RUN go get -d -v ./...
RUN go install -v ./...

FROM alpine:3.7
RUN apk add --update-cache sqlite
COPY --from=build /go/bin/textql /usr/bin
WORKDIR /tmp
ENTRYPOINT ["textql"]
```

Build and then test passes!

```console
vagrant@ubuntu-xenial:~/textql$ sudo docker build -f Dockerfile.alpine -t textql:alpine .
vagrant@ubuntu-xenial:~/textql$ sudo docker run --rm -it -v $(pwd):/tmp textql:alpine -header -console test.csv
SQLite version 3.21.0 2017-10-24 18:55:49
Enter ".help" for usage hints.
sqlite> SELECT * FROM test;
John|Barry|123456
Kathy|Smith|687987
Bob|McCornick|3979870
sqlite> SELECT * FROM test WHERE ssn>200000;
Kathy|Smith|687987
Bob|McCornick|3979870
sqlite>
```

Final docker image increases to 18.8MB with sqlite3 installed. The size is pretty small.

```console
vagrant@ubuntu-xenial:~/textql$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
textql              alpine              f800704c4c9d        3 minutes ago       18.8MB
golang              1.10-alpine3.7      05fe62871090        2 weeks ago         376MB
alpine              3.7                 3fd9065eaf02        4 months ago        4.15MB
```

## Summary

With docker multi-stage build and alpine, TextQL docker image is reduced by 93% (from 300MB to 19MB). You can use it as a regular command.

```console
# add alias
alias textql='docker run --rm -it -v $(pwd):/tmp textql:alpine '

# usage example
textql --header -console test.csv
```
