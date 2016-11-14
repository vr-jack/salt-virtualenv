# salt-virtualenv

## Who is responsible for this?!?!

My name is Erik Johnson, and I'm a Senior Engineer on the core development team at [SaltStack](https://saltstack.com). If you've participated in the community between early 2012 and today, then there's a good chance we've interacted in some way. Hopefully I haven't been a jerk to you.

## What is it?

A proof-of-concept for a potential method of distributing [SaltStack](https://saltstack.com) with all Python dependencies bundled within a virtualenv.

## Why?

[SaltStack](https://saltstack.com) releases starting with 2015.8.0 rely on [PyCrypto](https://www.dlitz.net/software/pycrypto/) 2.6.1 and [tornado](http://www.tornadoweb.org/) 4.2.1, which are not available in the repositories for most Linux distributions (especially LTS releases). This is part of the reason why we took steps to host our own [repository](https://repo.saltstack.com/), to provide these newer versions of needed dependencies.

However, [SaltStack](https://saltstack.com) obviously isn't the only project that uses these two modules, and upgrading them via our repositories can be problematic if you happen to:

1. Use other software which relies on the earlier versions of [PyCrypto](https://www.dlitz.net/software/pycrypto/) and [tornado](http://www.tornadoweb.org/) available in your distro's official repositories.

2. Have a company policy against non-essential upgrades.

This POC is an attempt to resolve these conflicts by distributing all Python dependencies in a [virtualenv](https://virtualenv.pypa.io/en/stable/). All of the Python bits get installed under ``/opt/salt``.

## Wait, RHEL/CentOS only? What about Debian/Ubuntu/SUSE/etc...?

Like it says above, this is a proof-of-concept. RHEL/CentOS is in my wheelhouse, Debian/Ubuntu/SUSE not so much. If we at [SaltStack](https://saltstack.com) decide to move forward with this, then our packaging team will expand this POC to other distros. However, for now this is RHEL/CentOS only. Sorry.

## Awesome! How the heck do I build this?

If you are so inclined, and already have a Fedora box with mock installed, the source RPM and the two RPMs that you need to pass to the mock command to provide two of the necessary ``BuildRequires`` can be found in the [rpmbuild-minimal](https://github.com/terminalmage/salt-virtualenv/tree/master/rpmbuild-minimal) directory.

However, to make it as easy as possible, this repo contains everything needed to build a Docker image which contains all the components necessary for building RPMs, and is pre-configured and ready-to-use. Additionally, within the container the spec and source files are expanded from the source RPM, for those interested in inspecting them.

Here's how to set up the Docker-based build environment and build the RPMs:

### 1) Clone this repo

```bash
% git clone https://github.com/terminalmage/salt-virtualenv
```

### 2) Create a data-only container

This provides persistent storage for the mock cache. We'll use it later on.

```bash
% docker run -v /var/cache/mock --name mock-cache busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
56bec22e3559: Pull complete
Digest: sha256:29f5d56d12684887bdfa50dcd29fc31eea4aaf4ad3bec43daf19026a7ce69912
Status: Downloaded newer image for busybox:latest
```

### 3) Build the Docker image

cd to the ``rpmbuild-minimal`` directory within the git clone, and use ``docker build`` to build the image. In his case I'm simply naming the image ``rpmbuild-minimal``.

```bash
% cd salt-virtualenv/rpmbuild-minimal
% sudo docker build -t rpmbuild-minimal .
```

### 4) Create a local destination for the packages we'll be building

This will be bound to ``/home/builder/mock`` in the docker image when we run it.

```bash
% mkdir ~/mock
```

You don't have to use ``~/mock``, but if you do use something different, then you'll of course need to replace occurrences of ``~/mock`` with the location you choose.

### 5) Launch an instance of the build image

```bash
% sudo docker run --cap-add=SYS_ADMIN --privileged --volumes-from=mock-cache --rm -it -h rpmbuild -v ~/mock:/home/builder/mock rpmbuild-minimal
```

- ``--cap-add=SYS_ADMIN`` is needed by mock to chroot within the container
- ``--privileged`` is needed by mock to install the base ``filesystem`` RPM within the chroot (this fails if ``/sys`` is mounted read-only, which is the default in unprivileged containers).
- ``--volumes-from=mock-cache`` ensures that we use the data-only container we created in step 2 to provide persistent storage for mock's rootfs and yum/dnf cache, keeping you from needing to re-download a bunch of stuff the next time you launch the container.

### 6) Use the included ``mockbuild`` helper script to build the package

When the Docker image starts, example commands for building packages RHEL/CentOS 5, 6, and 7 will be displayed. These commands use a helper script I wrote a while back to make mock a little more user-friendly.

These example commands can be viewed [here](https://github.com/terminalmage/salt-virtualenv/tree/master/rpmbuild-minimal/motd).

### 7) Back on the host machine, retrieve packages from the local ``mock`` directory you created in Step 4

```
% cd ~/mock/salt/2016.3.4-2.el5_erik/x86_64
% ls -l
total 26436
-rw-rw-r-- 1 erik erik   566803 Nov 13 16:32 build.log
-rw-rw-r-- 1 erik erik    48242 Nov 13 16:32 root.log
-rw-rw-r-- 1 erik kdm   9489548 Nov 13 16:31 salt-2016.3.4-2.el5_erik.src.rpm
-rw-rw-r-- 1 erik kdm  15298701 Nov 13 16:32 salt-2016.3.4-2.el5_erik.x86_64.rpm
-rw-rw-r-- 1 erik kdm     14358 Nov 13 16:32 salt-api-2016.3.4-2.el5_erik.x86_64.rpm
-rw-rw-r-- 1 erik kdm     17127 Nov 13 16:32 salt-cloud-2016.3.4-2.el5_erik.x86_64.rpm
-rw-rw-r-- 1 erik kdm   1544552 Nov 13 16:32 salt-master-2016.3.4-2.el5_erik.x86_64.rpm
-rw-rw-r-- 1 erik kdm     33593 Nov 13 16:32 salt-minion-2016.3.4-2.el5_erik.x86_64.rpm
-rw-rw-r-- 1 erik kdm     14979 Nov 13 16:32 salt-ssh-2016.3.4-2.el5_erik.x86_64.rpm
-rw-rw-r-- 1 erik kdm     14719 Nov 13 16:32 salt-syndic-2016.3.4-2.el5_erik.x86_64.rpm
-rw-rw-r-- 1 erik erik      962 Nov 13 16:32 state.log
```

### 8) Profit!
