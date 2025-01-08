FROM docker.io/ubuntu:24.04
ARG HUGO_VERSION

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get -qq -y install curl git golang-go \
    && apt-get clean
RUN curl -sL "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb" -o hugo.deb && \
    dpkg -i hugo.deb && \
    rm hugo.deb
