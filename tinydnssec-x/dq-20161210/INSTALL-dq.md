[dq](https://mojzis.com/software/dq/install.html)

## Unix installation

### download

```
$ wget https://mojzis.com/software/dq/dq-20161210.tar.gz
$ wget https://mojzis.com/software/dq/dq-20161210.tar.gz.asc
$ gpg --verify dq-20161210.tar.gz.asc dq-20161210.tar.gz
$ gunzip < dq-20161210.tar.gz | tar -xf -
$ cd dq-20161210
```

### compile and install binaries

```
$ make
$ sudo make install
```

### run dqcache

under root - create dqcache root directory

```
# mkdir -p /etc/indimail/dqcache/root/servers /etc/indimail/dqcache/env
# echo 10000000 > /etc/indimail/dqcache/env/CACHESIZE
# echo 127.0.0.1 > /etc/indimail/dqcache/env/IP
# echo "/etc/indimail/dqcache/root" > /etc/dqcache/env/ROOT
```

under root - setup dqcache root servers

```
# sh -c '(
echo "198.41.0.4"
echo "2001:503:ba3e::2:30"
echo "192.228.79.201"
echo "2001:500:84::b"
echo "192.33.4.12"
echo "2001:500:2::c"
echo "199.7.91.13"
echo "2001:500:2d::d"
echo "192.203.230.10"
echo "192.5.5.241"
echo "2001:500:2f::f"
echo "192.112.36.4"
echo "198.97.190.53"
echo "2001:500:1::53"
echo "192.36.148.17"
echo "2001:7fe::53"
echo "192.58.128.30"
echo "2001:503:c27::2:30"
echo "193.0.14.129"
echo "2001:7fd::1"
echo "199.7.83.42"
echo "2001:500:9f::42"
echo "202.12.27.33"
echo "2001:dc3::35"
) > /etc/dqcache/root/servers/@'
```

under root - create dqcache user

`# useradd dqcache`

under root - run dqcache server

```
# envuidgid dqcache envdir /etc/indimail/dqcache/env dqcache
```
