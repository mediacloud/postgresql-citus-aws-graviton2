# PostgreSQL + Citus Data built for AWS Graviton2

1. Launch a new ARM64 instance:
    * Ubuntu Server 20.04 LTS (HVM), SSD Volume Type
    * 64-bit (Arm)
    * t4g.2xlarge
    * 40 GB EBS volume
    * tags:
        * `Name: mediacloud-herewegoagain-build-arm64`
        * `project: mediacloud-herewegoagain`

2. SSH into the instance and run:

```bash
sudo hostnamectl set-hostname build-arm64

sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt-get -y install gcc-11 g++-11

sudo update-alternatives --remove-all gcc
sudo update-alternatives --remove-all g++
sudo update-alternatives --remove-all cc
sudo update-alternatives --remove-all c++
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 10
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-11 10
sudo update-alternatives --set cc /usr/bin/gcc
sudo update-alternatives --install /usr/bin/cc cc /usr/bin/gcc 10
sudo update-alternatives --set cc /usr/bin/gcc
sudo update-alternatives --install /usr/bin/c++ c++ /usr/bin/g++ 10
sudo update-alternatives --set c++ /usr/bin/g++

export CFLAGS="-march=armv8.2-a+fp16+rcpc+dotprod+crypto -mcpu=neoverse-n1 -fsigned-char"
export CXXFLAGS="$CFLAGS"
export CPPFLAGS="$CFLAGS"
export DEB_CFLAGS_SET="$CFLAGS"
export DEB_CPPFLAGS_SET="$CFLAGS"
export DEB_CXXFLAGS_SET="$CFLAGS"

sudo apt install curl ca-certificates gnupg
curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/apt.postgresql.org.gpg >/dev/null
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo sh -c 'echo "deb-src http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo apt-get -y update

sudo apt-get -y install dpkg-dev
```

3. Build PostgreSQL:

```bash
sudo apt-get -y install autoconf bison clang debhelper-compat dh-exec docbook-xml docbook-xsl flex gdb gettext libicu-dev libio-pty-perl libipc-run-perl libkrb5-dev libldap2-dev liblz4-dev libpam0g-dev libpam-dev libperl-dev libreadline-dev libselinux1-dev libssl-dev libsystemd-dev libxml2-dev libxml2-utils libxslt1-dev llvm-dev pkg-config postgresql-common python3-dev systemtap-sdt-dev tcl-dev uuid-dev xsltproc zlib1g-dev libz-dev clang-11 llvm-11-dev

apt-get source postgresql-14

git clone https://salsa.debian.org/postgresql/postgresql.git -b 14
cd postgresql/
git checkout debian/14.1-1
cp -R debian/ ../postgresql-14-14.1/debian/
cd ../postgresql-14-14.1/

sed -i 's/debhelper-compat (= 13)/debhelper, dh-exec (>= 0.13~)/' debian/control
sed -i 's/clang-9/clang-11/' debian/control
sed -i 's/llvm-9/llvm-11/' debian/control
sed -i 's/clang-9/clang-11/' debian/rules
sed -i 's/llvm-config-9/llvm-config-11/' debian/rules
echo 9 > debian/compat

dpkg-buildpackage -rfakeroot -b -uc -us

# Ensure that PostgreSQL got built with LSE (has to be non-zero)
objdump -d ./build/src/backend/postgres | grep -i 'cas\|casp\|swp\|ldadd\|stadd\|ldclr\|stclr\|ldeor\|steor\|ldset\|stset\|ldsmax\|stsmax\|ldsmin\|stsmin\|ldumax\|stumax\|ldumin\|stumin' | wc -l

# Ensure that PostgreSQL didn't get built with load-store exclusives (has to be zero)
objdump -d ./build/src/backend/postgres | grep -i 'ldxr\|ldaxr\|stxr\|stlxr' | wc -l
```

4. Build PgBouncer:

```bash
sudo apt-get -y install libevent-dev libc-ares-dev pandoc

# Install PostgreSQL packages that we've just built
sudo dpkg -i ~/*.deb

apt-get source pgbouncer

cd pgbouncer-1.16.1/

dpkg-buildpackage -rfakeroot -b -uc -us -d

sudo dpkg -i pgbouncer_1.16.1-1.pgdg20.04+1_arm64.deb
```


5. Build Citus extension:

```bash
sudo apt-get -y install autoconf flex git libcurl4-gnutls-dev libicu-dev \
                        libkrb5-dev liblz4-dev libpam0g-dev libreadline-dev \
                        libselinux1-dev libssl-dev libxslt1-dev libzstd-dev \
                        make uuid-dev

git clone https://github.com/citusdata/citus.git
cd citus/
git checkout v10.2.3

./configure --with-security-flags
make -j$(nproc)
make install-all
make install

cd src/tests/regression/
sudo chmod 777 /var/run/postgresql/
make check

# If upgrade / downgrade tests fail, that's probably fine

mkdir ~/citus-install/
make DESTDIR=~/citus-install install-all
make DESTDIR=~/citus-install install

cd ~/
mkdir citus-install/DEBIAN/
cat << EOF > citus-install/DEBIAN/control
Package: postgresql-14-citus-10.2
Source: citus
Version: 10.2.3.citus-1
Architecture: arm64
Maintainer: Linas Valiukas <linas@mediacloud.org>
Depends: libc6 (>= 2.17), libcurl4 (>= 7.16.2), libcurl4-gnutls-dev, liblz4-1 (>= 0.0~r130), libpq5 (>= 9.2~beta3), libssl1.1 (>= 1.1.0), libzstd1 (>= 1.3.2), postgresql-14
Conflicts: postgresql-14-citus
Provides: postgresql-14-citus
Section: database
Priority: optional
Homepage: https://github.com/mediacloud/postgresql-aws-graviton2
Description: sharding and distributed joins for PostgreSQL (built for AWS Graviton2)
 Citus is a distributed database implemented as a PostgreSQL extension. It
 provides functions to easily split a PostgreSQL table into shards to be
 placed on remote worker nodes. Citus can replicate shards, update their
 schemas, and keep track of shard health. An advanced distributed planner
 is included which handles queries and modifications against sharded tables.

EOF

dpkg-deb --build --root-owner-group citus-install/
mv citus-install.deb postgresql-14-citus-10.2_10.2.3.citus-1_arm64.deb
sudo dpkg -i postgresql-14-citus-10.2_10.2.3.citus-1_arm64.deb
```


6. Copy binaries to local machine:

```bash
scp mediacloud-herewegoagain-build-arm64:'*.deb' .
scp mediacloud-herewegoagain-build-arm64:'*.ddeb' .
```


7. Create a new GitHub release



## Links

* https://blog.dbi-services.com/postgresql-on-aws-graviton2-cflags/
* https://dev.to/aws-heroes/aws-postgresql-on-graviton2-with-newer-gcc-3aha
* https://www.postgresql.org/message-id/flat/099F69EE-51D3-4214-934A-1F28C0A1A7A7%40amazon.com
* https://www.percona.com/blog/2021/01/28/how-to-create-postgresql-custom-builds-and-debian-packages/
* https://github.com/aws/aws-graviton-getting-started/blob/main/c-c++.md
* https://github.com/citusdata/citus/blob/master/CONTRIBUTING.md
* https://www.internalpointers.com/post/build-binary-deb-package-practical-guide
