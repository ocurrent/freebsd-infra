---

During the FreeBSD installation you will have created a user account on the machine.

Make sure you can sudo without a password.

```shell
echo "mte24 ALL=(ALL:ALL) NOPASSWD: ALL" > /usr/local/etc/sudoers.d/mte24
```

FreeBSD doesn't install python by default so this must be installed first before Ansible will work.

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
