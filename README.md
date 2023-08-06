---

During the FreeBSD installation:
- Unselect all the optional system components (kernel-dbg and lib32)
- Choose layout `Auto (ZFS)`
- Add users to the system: No
- Manual configuration shell at the end of the installation:

Remove all these (empty) unnecessary ZFS partitions to prevent them from being used

```shell
zfs destroy -r zroot/tmp
zfs destroy -r zroot/usr
zfs destroy -r zroot/var
mkdir -p /tmp
chmod 1777 /tmp
```

Create the ZFS pool for obuilder (typicaly on a second disk or partition)

```shell
zpool create obuilder /dev/da1
```

Add your SSH key, allow root to login and disable password authentication

```shell
mkdir -m 700 /root/.ssh
fetch https://github.com/mtelvers.keys -o authorized_keys
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

FreeBSD doesn't install Python by default so this must be installed first before Ansible will work.

```shell
pkg install -y python
```

And on the machine you will run Ansible _from_ install the community.general collection

```shell
ansible-galaxy collection install community.general
```

Then create the base images on your FreeBSD machine:

```shell
ansible-playbook -i hosts playbook.yml
```

Once install the worker logs to syslog and can be stopped and started with

```
/etc/rc.d/worker start
/etc/rc.d/worker stop
```
