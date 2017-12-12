FROM debian:stretch

MAINTAINER innoQ iQπ team <iqpi@innoq.com>

RUN set -e -x && \
  apt-get update && apt-get -y upgrade && \
  apt-get install -y python3 openssl && \
  mkdir -p /data

COPY kubernetes_make_user /usr/local/bin
VOLUME /data
WORKDIR /data
