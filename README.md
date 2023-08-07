---

Install FreeBSD accept the default values.  Choose either UFS or ZFS
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
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
echo "ChallengeResponseAuthentication no" >> /etc/ssh/sshd_config
echo "PasswordAuthentication no" >> /etc/ssh/sshd_config
```

Finish the installation and reboot

```shell
exit
```

Ensure you can SSH to the machine (as root) without a password.

FreeBSD doesn't install Python by default so this must be installed
first before Ansible will work.

```shell
pkg install -y python
```

And on the machine you will run Ansible _from_ install the
community.general collection

```shell
ansible-galaxy collection install community.general
```

Then create the base images on your FreeBSD machine:

```shell
ansible-playbook -i hosts playbook.yml
```

Once install the worker logs to syslog and can be stopped and started with:

```shell
/etc/rc.d/worker start
/etc/rc.d/worker stop
```
