# InterPlanetary Container Registry

*Disclaimer: this project is in the early prototype stage.*

The ultimate goal of this project is to build a fully decentralised version of [Docker Hub](https://hub.docker.com) with the following features:
* Images and their metadata are stored on p2p storage.
* Private images are encrypted.
* Decentralised identification.
* Web frontend is running on p2p network.

You can check a prototype of the web frontend in [ipcr/web](https://github.com/ipcr/web). It looks like an extremely simplified version of Docker Hub. Live demo is running [on Fleek](https://ipcr.on.fleek.co).

The project was started at [HackFS](https://hackfs.com). HackFS is a virtual Hackathon by ETHGlobal and Protocol Labs.

The following sections of the document describe how to build a rudimentary public container registry using [IPFS](https://ipfs.io), [IPNS](https://docs.ipfs.io/concepts/ipns/) and [DNSLink](https://dnslink.io).

In the future a tool called `ipcr` will be built.
So that you can simply `ipcr push image:tag` to publish an image.
And `ipcr pull some.domain/image:tag` to pull an image.

## Prerequisites

All commands below were tested on macOS version 10.15.5 running Docker Desktop 2.3.0.3.

### Install IPFS

You can follow the [official guide](https://docs.ipfs.io/install/command-line-quick-start/#install-ipfs).
But, using Homebrew is quicker:

	$ brew install ipfs

### Initialize IPFS repository

To initialize the repository, run the following command:

	$ ipfs init

For more details, see [IPFS documentation](https://docs.ipfs.io/install/command-line-quick-start/#initialize-the-repository).

### Take IPFS node online

To join your node to the public network, run the ipfs daemon:

	$ ipfs daemon

## Initialize container registry

To create an empty container registry and enable [IPNS](https://docs.ipfs.io/concepts/ipns/#example-ipns-setup) for it, run the following commands:

```console
$ mkdir .ipcr
$ ipfs add -Qr .ipcr
QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn
$ ipfs name publish QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn
Published to QmTw9u8R3FCh2DR1t6PvkWibVnZXJfujzEKV2o4SRrTPLV: /ipfs/QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn
```

Note, that publishing to IPNS takes some time.

The last hash, next to the folder CID, is the one to remember, call it `$PEER_ID` for now:

	$ PEER_ID=QmTw9u8R3FCh2DR1t6PvkWibVnZXJfujzEKV2o4SRrTPLV

## Edit your DNS records

Assume you have the domain name `your.domain` and can access your registrar's control panel to manage DNS entries for it.

Create a DNS TXT record ([DNSLink](https://docs.ipfs.io/concepts/dnslink/)), with the key `your.domain`. and the value `dnslink=/ipns/$PEER_ID` where `$PEER_ID` is the value from the section above.

Once you've created that record, and it has propagated you should be able to find it.

```console
$ dig +noall +answer TXT your.domain
your.domain.            60      IN      TXT     "dnslink=/ipns/$PEER_ID"
```

You can now list registry contents using `your.domain`:

	$ ipfs ls /ipns/your.domain

Note, that the registry is still empty.
Time to add some images!

## Push Docker image to IPFS

First, you should pull some image from [Docker Hub](https://hub.docker.com):

	$ docker pull -q ipfs/go-ipfs:v0.6.0

Next, save this image to your registry directory:

	$ mkdir -p .ipcr/go-ipfs/tags/v0.6.0
	$ echo 'Go implementation of IPFS, the InterPlanetary FileSystem' > .ipcr/go-ipfs/README-short.txt
	$ docker save ipfs/go-ipfs:v0.6.0 | tar -xf - -C .ipcr/go-ipfs/tags/v0.6.0

Finally, publish the updated version of your container registry to IPFS:

```console
$ ipfs add -Qr .ipcr
QmcZqjtj3QhYEAr8a9Uu1A3vThtn5uee9iMjVJEzMYaY4p
$ ipfs name publish /ipfs/QmcZqjtj3QhYEAr8a9Uu1A3vThtn5uee9iMjVJEzMYaY4p
Published to QmTw9u8R3FCh2DR1t6PvkWibVnZXJfujzEKV2o4SRrTPLV: /ipfs/QmcZqjtj3QhYEAr8a9Uu1A3vThtn5uee9iMjVJEzMYaY4p
```

## Pull Docker image from IPFS

To pull an image you download it from IPFS and then load using Docker:

	$ ipfs get -o go-ipfs:v0.6.0 /ipns/your.domain/go-ipfs/tags/v0.6.0
	$ tar --strip-components 1 -cf - go-ipfs:v0.6.0 | docker load

Note, that downloading the image as a TAR archive from IPFS using `--archive` and loading this archive using Docker doesn't work.
Because the leading path element of the archive will be `v0.6.0` in that case.
This is not expected by Docker and you'll get a non obvious error saying `open /var/lib/docker/tmp/docker-import-787571586/v0.6.0/json: no such file or directory` when loading the image.
Therefore, the image directory is archived with `--strip-components 1` before loading.

## Prior art

The following list contains a few projects that implement a [Docker Registry](https://docs.docker.com/registry) on top of IPFS:

* [Storage Driver: Add an IPFS driver](https://github.com/docker/distribution/pull/2906)
* [IPDR: InterPlanetary Docker Registry](https://github.com/miguelmota/ipdr)
* [Docker Registry for IPFS](https://blog.bonner.is/docker-registry-for-ipfs)

## License

This project is licensed under the MIT License–see the [LICENSE](LICENSE) file for details.