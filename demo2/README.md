# 1. Define a Luet tree
tree .

mkdir alpine

cat <<EOF > alpine/build.yaml
unpack: true
image: "alpine:3.11.3"
EOF

cat <<EOF > alpine/definition.yaml
category: "distro"
name: "alpine"
version: "3.11.3"
EOF

mkdir golang

cat <<EOF > golang/build.yaml
requires:
- category: "distro"
  name: "alpine"
  version: "3.11.3"
prelude:
- apk update
- apk add go=1.13.4-r1
EOF

cat <<EOF > golang/definition.yaml
category: "dev-lang"
name: "go"
version: "1.13.4"
EOF


mkdir gogs

cat <<'EOF' > gogs/build.yaml
requires:
- category: "dev-lang"
  name: "go"
  version: ">=0"
prelude:
- apk update
- apk add git
- git clone https://github.com/gogs/gogs
- cd gogs && git checkout v"${PACKAGE_VERSION}"
steps:
- cd gogs && go mod update && CGO_ENABLED=0 go build && mv gogs /usr/bin/gogs
EOF

cat <<EOF > gogs/definition.yaml
category: "dev-vcs"
name: "gogs"
version: "0.11.91"
requires:
- category: "distro"
  name: "alpine"
  version: "3.11.3"
EOF

tree .

# 2. Build packages

luet build dev-vcs/gogs --concurrency 1
# 3. Create a (local) repository

# 4. Install them!

# Let's create a temporary folder, where we will chroot inside
export tmpdir="$(mktemp -d)"