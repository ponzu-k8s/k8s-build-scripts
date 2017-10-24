
# Base our image on an official, minimal image of our preferred golang
FROM golang:1.8.1

# Note: The default golang docker image, already has the GOPATH env variable set.
# GOPATH is located at /go
ENV GOSRC $GOPATH/src
ENV PONZU_GITHUB github.com/ponzu-cms/ponzu
ENV PONZU_ROOT $GOSRC/$PONZU_GITHUB

# Consider updating package in the future. For instance ca-certificates etc.
# RUN apt-get update -qq && apt-get install -y build-essential

# get latest ponzu from github & build it
RUN go get $PONZU_GITHUB/...

# this is where to mount the persistent disk for your project
ENV PROJECT_SHARE $GOSRC/project
VOLUME $PROJECT_SHARE
WORKDIR $PROJECT_SHARE

# copy startup script to /usr/local/bin
COPY start_admin /usr/local/bin/start_admin

# expose port 8080 for ponzu admin server
EXPOSE 8080

# its better to define this in deployment yaml
# CMD [ "start_admin" ]
