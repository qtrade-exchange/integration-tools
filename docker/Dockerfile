FROM ubuntu:bionic as builder

WORKDIR /home/docker/

ARG COIN_BIN=bitcoind
ARG COIN_CLI_BIN=bitcoin-cli
ARG GIT_REPO=https://github.com/bitcoin/bitcoin.git
ARG GIT_CHECKOUT=v0.18.0

RUN apt-get update -y \
  && apt-get install software-properties-common python-dev -y \
  && add-apt-repository ppa:bitcoin/bitcoin -y \
  && apt-get update -y \
  && apt-get install -y git build-essential libtool autoconf-archive autotools-dev automake pkg-config libssl-dev libevent-dev bsdmainutils libboost-system-dev libboost-filesystem-dev  libboost-chrono-dev libboost-program-options-dev libboost-test-dev libboost-thread-dev libdb4.8-dev libdb4.8++-dev libzmq3-dev libgmp-dev \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN git clone ${GIT_REPO} src
WORKDIR /home/docker/src
RUN ./autogen.sh && ./configure --without-gui && make -j 4
RUN strip src/${COIN_BIN} src/${COIN_CLI_BIN} \
   && mv src/${COIN_BIN} /home/docker/coinserverd \
   && mv src/${COIN_CLI_BIN} /home/docker/coinserver-cli


FROM ubuntu:bionic

ENV COIN_DATA=/home/docker/data

RUN useradd -ms /bin/bash docker \
  && apt-get update -y \
  && apt-get install software-properties-common -y \
  && add-apt-repository ppa:bitcoin/bitcoin -y \
  && apt-get update -y \
  && apt-get install -y curl gnupg gosu jq \
  && apt-get install -y libssl-dev libevent-dev libboost-system-dev libboost-filesystem-dev  libboost-chrono-dev libboost-program-options-dev libboost-test-dev libboost-thread-dev libdb4.8-dev libdb4.8++-dev libzmq3-dev \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

USER docker
WORKDIR /home/docker

COPY --from=builder --chown=docker:docker /home/docker/coinserverd /home/docker/
COPY --from=builder --chown=docker:docker /home/docker/coinserver-cli /home/docker/
RUN chmod 777 /home/docker/coinserverd \
    && mkdir -p ${COIN_DATA} \
    && chown -R docker ${COIN_DATA} \
    && chmod 700 ${COIN_DATA} \
    && ls -la

VOLUME ["/home/docker/data"]
CMD ["/home/docker/coinserverd", "-datadir", "/home/docker/data/"]
