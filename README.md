# Ansible Role: Add Gadget-PPPD systemd units

[![Build Status](https://travis-ci.org/theatro-noc/ansible-role-gadget-pppd-units.svg?branch=master)](https://travis-ci.org/theatro-noc/ansible-role-gadget-pppd-units)

This role installs the systemd units for connecting multiple gadgets via pppd over usb (available on [bitbucket](https://bitbucket.org/theatro-noc/gadget-pppd-units)).

This role clones the repository (and leaves the clone on the system), and then installs or enables the systemd units and files as needed. You can change the clone location if desired (see Role Variables below).

## Requirements

[`cognifloyd.bitbucket-sources`](https://galaxy.ansible.com/cognifloyd/bitbucket-sources/) is a dependency that must already be installed before running this role, as it does not list this dependency in [meta/main.yml](meta/main.yml). Listing it as a dependency causes errors because the role variables are not provided until it is included via an [`include_role`][1] task. Therefore,  make sure to include `cognifloyd.bitbucket-sources` in your playbook's galaxy dependency file (See [stackoverflow][2] and the [`ansible-galaxy` docs][3]).

## Role Variables

To change the clone location, set `gadget_pppd_clone_dest`. I, Jacob Floyd, like to have an extension on source controlled directories to indicate how I cloned them (`git`, `svn`, `git-svn`, `hg`, etc.) If you do not want the .git extension, change this variable.

The clone will be created by `gadget_pppd_clone_owner` with group `gadget_pppd_clone_group`, and will have the permissions of that user/group.

Bitbucket requires some kind of credentials to access a repository, so you'll need to provide a [bitbucket access key][4] in `gadget_pppd_clone_key`. If you need to clone a fork, you can also change `gadget_pppd_clone_repo_account` and/or  `gadget_pppd_clone_repo_name`.

_If you want me to add your access key theatro-noc repo, please email jacob[at]theatro.com, and I'll consider it._

The pppd units use a variety of options that are sourced from the systemd EnvironmentFile: `/etc/pppd-systemd-units.conf` This is a templated file that uses the `gadget_pppd_options*`, `gadget_pppd_eth0_device`, and `gadget_pppd_ip*`. `gadget_pppd_options` will be appended to `gadget_pppd_options_default`, so if you want to get rid of the default options, you must override `gadget_pppd_options_default` with an empty list.

If you want to install any units that should depend on `gadget.target`, list those files (as a path on the ansible controller) in `gadget_pppd_dependent_units`.

### `defaults/main.yml`:

_Note: you must pre-generate the access key._
```yaml
gadget_pppd_clone_repo_type: git
gadget_pppd_clone_repo_account: "theatro-noc"
gadget_pppd_clone_repo_name: "gadget-pppd-units"
gadget_pppd_clone_dest: "/opt/theatro/theatro-noc/gadget-ppd-units.git"
gadget_pppd_clone_owner: "theatro"
gadget_pppd_clone_group: "theatro"
gadget_pppd_clone_key: "~/.ssh/theatro-noc-accesskey"
gadget_pppd_options_default:
  - nodefaultroute
  - lock
gadget_pppd_options:
  - noauth
gadget_pppd_eth0_device: eth0
gadget_pppd_dependent_units: []
```

### `vars/main.yml`:
You should not need to override these.

`gadget_pppd_ip_default` has the default settings for IP configuration of the connected gadgets and their alias IP once connected. To change this, please set `gadget_pppd_ip` as a role var.

This also has a list of the units to install and the binaries used by those units.

```yaml
gadget_pppd_ip_default:
  base: "10.111.111"
  mask: 31
  alias_mask: 32
  gadget: "10.222.222.223"
  ppp: "10.222.222.223"

gadget_pppd_units:
  - <a list of units in the bitbucket repo>
gadget_pppd_default_bin_paths:
  - <a list of binaries used by the units>
```

### `vars/<linux distro or os_family>.yml`:
Binary dependencies are listed here to make sure they are installed. You should not need to override these unless using an unknown distribution. If you do change these, please add an issue mentioning which distribution and what the proper package names are.

Note that `gadget_pppd_required_packages_update` is only needed to support Gentoo with Ansible 2.2. Once 2.3 is released, this will be removed (see [ansible/ansible#21918](https://github.com/ansible/ansible/issues/21918)).

```yaml
gadget_pppd_required_packages_state: latest
#gadget_pppd_required_packages_update:
gadget_pppd_required_packages:
  - <a list of os package dependencies>
```

### role parameters:

These vars must be defined to use this role. They can be defined as role parameters or as vars for an entire play. Note that this role does not `allow_duplicates`, so the vars can be defined anywhere in the playbook as long as they are available before the role runs.

An important part of wiring up gadgets to auto-connect with 	`pppd` is telling udev what to do with them. This role will create `/etc/udev/rules.d/72-<name>-ttyacm.rules` using these vars (where `<name>` refers to `gadget_pppd_udev_rules_name`).

You'll need to look up your usb gadget's `idProduct` and `idVendor`. Make sure to specify them as octal numbers (ie: drop the leading `0x`, but don't drop any other `0`s. There should be four characters in quotation marks.).

  - On Windows, use Device Manager (see [here][5])
  - On MacOS X, use system_profiler (see [here][5])
  - On linux, use tools like [`lsusb`][6] and [`udevadm`][7] (see [here][5], or get more details [here][8] and [here][9]).

Note that `gadget_pppd_udev_rules` is a list of dictionaries where each dictionary has `name`, `idProduct`, and `idVendor`. For every device you add, a udev rule will be created that will trigger the `pppd@.service` via systemd.

`gadget_pppd_ip` specifies the `IP_*` scheme used by the units to setup connections to the gadgets.

```yaml
gadget_pppd_udev_rules_name: "vendor-gadget"
gadget_pppd_udev_rules:
  - name: "Great Descriptive Name of Gadget"
	idProduct: "0000"
	idVendor: "0000"
gadget_pppd_ip:
  # see gadget_pppd_ip_default in vars/main.yml
```

### global scope vars:
`ansible_distribution` and `ansible_os_family` are used to determine how to install some parts of this.

Please note that, on RedHat-family distributions, `/etc/ppp/ip-up.local` and `/etc/ppp/ip-down.local` will be replaced if they exist. The original files will be backed up to `/etc/ppp/<file>.backup-<date>` (where `date` comes from `ansible_date_time.date`). Please move anything that you needed from those `ip-*.local` files into the `/etc/ppp/ip-up.d/` and `/etc/ppp/ip-down.d/` directories. Make sure that any you suffix any scripts in these directories with `.sh`, and note that they will be sourced from within bash. This makes RedHat match the drop-in ppp script pattern that distros like Gentoo, Arch, and Debian all use.

## Dependencies

### `cognifloyd.bitbucket-sources`
This role uses `cognifloyd.bitbucket-sources` via `include_role`. However, `private: yes` is set on the `include_role` task to prevent leaking of bitbucket-sources variables and to `allow_duplicates` of that role.

Please see [Requirements](#requirements) for more about how this role is used.


## Example Playbook

```yaml
- hosts: servers
  vars:
    gadget_pppd_udev_rules_name: "vendor-gadget"
    gadget_pppd_udev_rules:
      - name: "Great Descriptive Name of Gadget"
        idProduct: "0000"
        idVendor: "0000"
  roles:
     - role: theatro.gadget-pppd-units
```

## License

MIT

## Author Information

Written by [Jacob Floyd](https://github.com/cognifloyd), employed by [Theatro](theatro.com), in 2017.

<!-- links -->
[1]: http://docs.ansible.com/ansible/include_role_module.html
[2]: http://stackoverflow.com/a/30176625/1134951
[3]: http://docs.ansible.com/ansible/galaxy.html#installing-multiple-roles-from-a-file
[4]: https://confluence.atlassian.com/bitbucket/use-access-keys-294486051.html
[5]: http://xpenology.me/how-to-see-the-value-of-my-vid-pid-stick/
[6]: https://www.mankier.com/8/lsusb
[7]: https://www.mankier.com/8/udevadm
[8]: http://elinux.org/Lsusb_(Linux)
[9]: http://weininger.net/how-to-write-udev-rules-for-usb-devices.html
