![catclock](https://github.com/BarkyTheDog/catclock/raw/master/catclock.gif)

# Catclock example

In this example, we will build the awesome [CatClock](https://github.com/BarkyTheDog/catclock) on containers we will run it locally in a Luet box. 

We will do this experiment to prove two things: 
1) how we can build a package with Luet and 
2) two packages from different distributions can (sometime) work together.


## 1) Create the package

To prove our point, we will build our package from an opensuse image, and later on we will consume
entropy repositories for runtime dependencies. To note, this is not the main focus of Luet, and this is a restricted example on its features on build-time resolution.  For more syntax examples, see [Build specs](https://luet-lab.github.io/docs/docs/concepts/specfile/#build-specs) and [Package types](https://luet-lab.github.io/docs/docs/concepts/packages/#package-types) in the official docs.


Run this commands in any directory you choose to be your workspace:

```bash

# Let's create a directory to store our package spec:
mkdir -p tree/misc/catclock/

# Create a build file. We use here opensuse/leap to build the package, as an example
cat <<EOF > tree/misc/catclock/build.yaml
image: opensuse/leap

prelude:
- zypper in -y git make libXt-devel xmh gcc motif-devel libXext-devel libpulse-devel libaubio-devel
- git clone https://github.com/BarkyTheDog/catclock

steps:
- cd catclock && make DEFINES="-Wno-incompatible-pointer-types"
- mv catclock/xclock /usr/bin/xclock

#unpack: true
includes:
- /usr/bin/xclock
EOF

# Create a runtime definition. We will leverage packages already present on Entropy repositories
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

## 2) Build it!

```bash
sudo /usr/bin/luet build --pull \
--image-repository sabayonarm/luetcache \
--clean=false --tree=$PWD/tree misc/catclock \
--destination $PWD/build --backend docker --concurrency 1 --compression gzip
sudo chown -R $USER $PWD/build
```
Note, we need sudo to keep the permissions properly mapped in the artifact which is produced
this is not always the case. Depends on the package content.



## 3) Create a local repository
```bash
/usr/bin/luet create-repo --tree "tree" \
--output $PWD/build \
--packages $PWD/build \
--name "test repo" \
--descr "Test Repo" \
--urls "http://localhost:8000" \
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