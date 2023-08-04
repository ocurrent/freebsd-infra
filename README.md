
FreeBSD doesn't install python by default

```shell
pkg install -y python
```

Make sure you can sudo without a password
```
echo "mte24 ALL=(ALL:ALL) NOPASSWD: ALL" > /usr/local/etc/sudoers.d/mte24
```
