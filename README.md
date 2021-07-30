# PostgreSQL built for AWS Graviton2

PostgreSQL rebuilt and optimized for AWS Graviton2 processors (`armv8.2` with LSE atomics):

```bash
DEB_CFLAGS_SET="-march=armv8.2-a+fp16+rcpc+dotprod+crypto -mcpu=neoverse-n1 -fsigned-char" \
    dpkg-buildpackage -rfakeroot -b -uc -us
```

Used `gcc-11-11.1.0-1ubuntu1~20.04` from [`ppa:ubuntu-toolchain-r/test`](https://launchpad.net/%7Eubuntu-toolchain-r/+archive/ubuntu/test).


## Links

* https://blog.dbi-services.com/postgresql-on-aws-graviton2-cflags/
* https://dev.to/aws-heroes/aws-postgresql-on-graviton2-with-newer-gcc-3aha
* https://www.postgresql.org/message-id/flat/099F69EE-51D3-4214-934A-1F28C0A1A7A7%40amazon.com
* https://www.percona.com/blog/2021/01/28/how-to-create-postgresql-custom-builds-and-debian-packages/
* https://github.com/aws/aws-graviton-getting-started/blob/main/c-c++.md
