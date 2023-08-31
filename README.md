---

Install FreeBSD accepting the default values.  Choose either UFS or ZFS
filesystem as your preference.  No need to add a additional user.  If you
include `kernel-dbg` and `lib32` make sure your partition is about 16G,
otherwise 8G is sufficient.

In the manual configuration shell at the end of setup run the following.

Create the ZFS pool for obuilder (typicaly on a second disk)

```shell
zpool create obuilder /dev/da1
```

Add your SSH key, allow root to login and disable password authentication

```shell
mkdir -m 700 /root/.ssh
fetch https://github.com/mtelvers.keys -o /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authoized_keys
sysrc -x sshd_flags
echo 'sshd_flags="-o ChallengeResponseAuthentication=no -o PermitRootLogin=yes -o PasswordAuthentication=no"' >> /etc/rc.conf
```

FreeBSD doesn't install Python by default so this must be installed
before Ansible will work.

```shell
pkg install -y python
```

Finish the installation and reboot

```shell
exit
```

Ensure you can SSH to the machine (as root) without a password.

Then create the base images on your FreeBSD machine:

```shell
ansible-playbook -i hosts playbook.yml
```

Once installed the worker logs to syslog and can be stopped and started with:

```shell
service worker start
service worker stop
```

## Updating

Use `update.yml` to pause the worker, update it, rebuild the base images, and resume it.

```shell
ansible-playbook -i hosts update.yml
```


## Manual partitioning

If you have a large single disk and need to partition it these commands may be helpful.

```shell
gpart add -t freebsd ada0
gpart show
zpool create obuilder /dev/ada0s2
```
