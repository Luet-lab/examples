![catclock](https://github.com/BarkyTheDog/catclock/raw/master/catclock.gif)

# Catclock example

In this example, we will build the awesome [CatClock](https://github.com/BarkyTheDog/catclock) on containers we will run it locally in a Luet box. 

We will do this experiment to prove two things: 
1) how we can build a package with Luet and 
2) two packages from different distributions can (sometime) work together.

## Prerequisites

To build packages with Luet, you must have installed Docker and container-diff, follow our [setup guide](https://luet-lab.github.io/docs/docs/getting-started/#setup) and [build requirements](https://luet-lab.github.io/docs/docs/getting-started/requirements/)


## 1) Create the package

To prove our point, we will build our package from an OpenSUSE image, and later on we will consume
entropy repositories for runtime dependencies. To note, this is not the main focus of Luet, and this is a restricted example on its features on build-time resolution.  For more syntax examples, see [Build specs](https://luet-lab.github.io/docs/docs/concepts/specfile/#build-specs) and [Package types](https://luet-lab.github.io/docs/docs/concepts/packages/#package-types) in the official docs.


Run this commands in any directory you choose to be your workspace:

```bash

# Let's create a directory to store our package spec:
mkdir -p tree/misc/catclock/
```

### 1.1) Build spec

Now, let's generate our **build** spec:

```bash
# Create a build file. We use here opensuse/leap to build the package, as an example
cat <<EOF > tree/misc/catclock/build.yaml
image: opensuse/leap

# Preparation phase
prelude:
- zypper in -y git make libXt-devel xmh gcc motif-devel libXext-devel libpulse-devel libaubio-devel
- git clone https://github.com/BarkyTheDog/catclock

# Here we define the steps that Luet will follow
steps:
- cd catclock && make DEFINES="-Wno-incompatible-pointer-types"
- mv catclock/xclock /usr/bin/xclock

# (optional) File list that will be included in the final package
# Luet will filter out files that won't match any entry in the list (regex syntax IS supported)
includes:
- /usr/bin/xclock
EOF

```

`build.yaml` is what an ebuild is for Gentoo and for e.g. what PKGBUILD is for Arch.

- *image: opensuse/leap* tells luet to use opensuse/leap as a build image. We collect the build time dependencies with `zypper` (the openSUSE package manager), and the [CatClock](https://github.com/BarkyTheDog/catclock) with `git`. When we declare an `image` keyword in a spec, it becomes a *seed* package ( see [Package types](https://luet-lab.github.io/docs/docs/concepts/packages/#package-types) ) as doesn't depend on any package in build time, we will cover more use cases in other examples.
- *prelude* is a list of commands that will happen during the build phase.
They might generate binaries, or download sources, but those are not took into consideration when generating the final package.
- *steps* is a list of commands that will happen during the build phase.
Luet will execute those commands and all the binaries generated from them become part of the final package
- *includes* is a (optional) list of regex that tells to Luet what files to filter out from the final artifact.

### 1.2) Runtime spec

Now we generate the runtime spec, it's the part about the binary end which will be installed in the system. It also holds the metadata relative to the package definition (`name`, `category`, `version`).

```bash
# Create a runtime definition.
# We will leverage packages already present on Sabayon Entropy repositories
# the end-system needs to have the Luet Sabayon Entropy repositories enabled.
cat <<EOF > tree/misc/catclock/definition.yaml
category: "misc"
name: "catclock"
version: "0.20200318"
requires:
- category: meta
  name: users
  version: ">=0"
- category: x11-libs
  name: motif
  version: ">=0.1"
- category: media-libs
  name: libjpeg-turbo
  version: ">=0.1"
EOF
```

- *category*, *name*, and *version*: identifies the package in a Luet tree. This is the unique identifier for a package.
- *requires* it's a list of packages which our **catclock** depends on during runtime (when we will execute catclock inside a small-container!). To find out what's required by your binaries it can be a try-learn-fail effort. If the package you wish to build is specifying the deps it requires, and those are available in a Luet repository, you are all set, just point them there. Otherwise you have to figure out after you build the binary the first time (for example, with `ldd`) to which libraries it depends on.
In this example we consume the dependencies from the [Luet Entropy Repo](https://github.com/Luet-lab/luet-entropy-repo), that we will enable on the following steps.

## 2) Build it!

```bash
sudo /usr/bin/luet build \
--tree=$PWD/tree misc/catclock \
--destination $PWD/build \
--compression gzip

sudo chown -R $USER $PWD/build # So later on, we can access to the repository with our user
```

We are building the specs in this step.

- *tree*: is the path where our specs are, in our case it's `tree`.
- *destination*: is the path where our packages will be stored, in our case this is `build`.
- *compression*: is the compression algorithm used to compress the final artifacts

Note, we need *sudo* to keep the permissions properly mapped in the artifact which is produced
this is not always the case. Depends on the package content.


## 3) Create a local repository

```bash
/usr/bin/luet create-repo --tree "tree" \
--output $PWD/build \
--packages $PWD/build \
--name "test repo" \
--descr "Test Repo" \
--tree-compression gzip \
--meta-compression gzip
```

## 4) Let's test it now!!!

```bash
# Let's create a directory for our "fake" rootfilesystem
# it will be populated with a minimal set of packages needed to run 
# our amazing catclock
mkdir -p $PWD/rootfs

# Let's also create a directory to store our config files
mkdir -p $PWD/conf
```

```bash

# We create here a config file which references the rootfs.
# In this way, luet instead installing packages to your host system, will populate the rootfs
# (note, all the steps are run by a user here, no root required!)
cat <<EOF > conf/luet-dso-local.yaml
general:
  debug: true
system:
  rootfs: $PWD/rootfs # ROOTFS FOLDER
  database_path: "/"
  database_engine: "boltdb"
repositories:
   - name: "main"
     type: "disk"
     priority: 3
     enable: true
     urls:
       - "$PWD/build" # BUILD FOLDER
   - name: "sabayonlinux.org"
     description: "Sabayon Linux Repository"
     type: "http"
     enable: true
     cached: true
     priority: 2
     urls:
       - "https://dispatcher.sabayon.org/sbi/namespace/luet-entropy-repo"
   - name: "luet-repo"
     description: "Luet Official Repository"
     type: "http"
     enable: true
     cached: true
     priority: 1
     urls:
       - "https://raw.githubusercontent.com/Luet-lab/luet-repo/gh-pages"
EOF
# we have specified an additional repository, one that is luet-entropy-repo (which contains
# the runtime dependencies we specified in our package)
```

```bash
# Let's populate our rootfs with some minimal things: base-gcc, and bash
export LUET_NOLOCK=true
luet install \
--config $PWD/conf/luet-dso-local.yaml \
meta/users
```

```bash
# catclock is a X11 app! we want to be able to play with it locally from our host :)
# Let's copy the .Xauthority file to allow the X app to communicate with our X server
# Note: This can be achieved in other ways (set up a tcp X server, and so on)
cp -rfv $HOME/.Xauthority $PWD/rootfs/                                                        
```

```bash
luet install \
--config $PWD/conf/luet-dso-local.yaml \
misc/catclock
```

```bash
# Let's run our beautiful catclock :)
luet box exec --rootfs $PWD/rootfs \
--stdin --stdout --stderr --env DISPLAY=$DISPLAY \
--env XAUTHORITY=/.Xauthority --mount /tmp --entrypoint /usr/bin/xclock
```

Someone said bash me?

```bash
luet box exec --rootfs $PWD/rootfs \
--stdin --stdout --stderr --env DISPLAY=$DISPLAY \
--env XAUTHORITY=/.Xauthority --mount /tmp --entrypoint /bin/bash
```