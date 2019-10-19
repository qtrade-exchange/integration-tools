# Why Docker & Docker Compose

qTrade containerizes all node and wallet software we run. This has many advantages regarding easily building/running/deploying/securing software developed by third parties, which are often developed in different environments.

If your team can provide us with Docker and Docker compose files that save us time, this can greatly speed up our listing process, and therefore means we're more likely to be able to list your coin.

## Notes about substantial time saving components

It is fairly unlikely we will be able to use your Docker/compose files as-is, due to our security requirements and architecture differences. The main advantage they provide us is the ability to rapidly create our own instead of starting from scratch. Here are a few of the biggest areas we can save time by looking at your docker setup:

### Configuration management

We handle configuration entirely through environment variables, but rarely does node or wallet software support this. Typically they are designed to run from a config file. There are many ways to handle this, but here are a couple of the ways we've used:

 - Modify node/wallet software to accept config variables as runtime arguments
 - Run an entrypoint script that pulls environment variables and updates a config file before starting the node software

### Data persistance

We only want to mount one persisted volume per node, and this is typically best accomplished by placing all data that needs to be persisted in a single (configurable) folder. Considerable maintenance issues can be caused if the node generates stateful files outside that directory

### Key/Wallet loading (extra credit)

We often have a single seed/key that is used to generate an account, or derive user addresses, and this may be passed in via an environment variable. If there is wallet software bundled it would ideally use this key to bootstrap a wallet file, or import the key (as appropriate). This step reduces the burden on our integration software to maintain the wallet state.


# Operationalizing your software

Now that we've covered the biggest time saving features we're looking for, lets discuss some design decisions for building your own Dockerfile and Docker Compose setup, what that might actually look like, and why we think these are good ideas.

## Dockerfile suggestions:

For compiled code there should be two docker stages:
 1. The builder stage. This should include all necessary build dependencies and compile the binary files
 2. The container image. This image should have binaries copied from the builder stage, and all dependencies necessary to run those binaries

The goal of the two stage system is to reduce image size and complexity by not having a bunch of extra packages and build artifacts. For uncompiled languages the two-step process step isn't necessary

### Notes
 - Dockerfile build process should pull the git repository, **and not assume the repo is already present**. This allows easier CI and build automation
 - Configuration of the node (& wallet) must be possible via environment variables or runtime CLI arguments. If your software only supports loading configuration from a config file this can be done by using Docker's entrypoint system to write/modify a config file before the daemon is started.
 - Blockchain data, wallets, and all data persisted between restarts needs to be in a single root directory (`/home/docker/data`). Binaries and other files that don't change should be below this directory. This allows easily mounting a volume for persisted data
 - Node/wallet should be able to run in an 'offline' mode and still perform signing/transaction creation/etc.
 - A good system for automatically loading wallets/keys into the container is a plus
 - Ideally caching would be reasonably optimized. Build steps that change more often should be as late as possible, while core dependencies should be setup first. See https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache for more information.

## Docker compose suggestions:

Our system requires that secret keys be network isolated, and in practice this means there are two main peices of software we run.

 1. A `watcher` that is a network connected node. It needs to have the ability to import public keys to check for incoming deposits, and broadcast withdraws to the network
 2. A `signer` that is network isolated software. It has secret keys, and is primarily used to sign withdraw transactions. 

 **If you can provide an example python script demonstrating how to generate addresses, key pairs, and sign transactions without connecting to a node/wallet API there is no need to containerize software for the signer, and this is preferred.**

We use docker compose to orchestrate tests for this two part configuration. Essentially, it should define 1-4 services:

 1. A node service 
 2. A wallet service (if not included in the node software)
 3. A signer node service (if required to sign transactions or generate addresses)
 4. A signer wallet service (if #3 is required and wallet software isn't in the node)
 
 These sevices should be configured by providing the proper arguments or environment variables to run it in the expected mode. For example, the `signer` service should be configured to run in an `offline` mode and not bind any ports or make any outgoing connections.

### Notes:
 - Mount points/volumes for persisted block chain data/config files should be specified
 - Should run signer in offline/isolated mode
 - Should pass in environment variables to configure
 - API Ports should be mapped to local ports

# Examples

We've provided an example Docker and Docker Compose file which should allow you to easily build and run a bitcoin node on your local system. Try it out:

```
docker build -t qtrade/csbuild/bitcoin:v18 . -f Dockerfile
mkdir ~/bitcoin-pv && chmod -R 777 ~/bitcoin-pv
docker-compose up
```
