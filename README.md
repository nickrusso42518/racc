[![Build Status](
https://travis-ci.com/nickrusso42518/racc.svg?branch=master)](
https://travis-ci.com/nickrusso42518/racc)

[![published](
http://cs.co/codeex-badge)](
https://developer.cisco.com/codeexchange/github/repo/nickrusso42518/racc)

# Run Arbitrary CLI Commands (racc)
This playbook runs arbitrary commands on a given node, stores the
output in a flat text file, and archives the entire "run" in an archive
file for easy copying to management stations via SCP. It is generally used
for configuration backups, hardware inventory, and license information.
The tool also supports automatic generation of file content hashes, helping
to determine configuration drift across collection batches.

> Contact information:\
> Email:    njrusmc@gmail.com\
> Twitter:  @nickrusso42518

  * [Supported Platforms](#supported-platforms)
  * [Getting Started](#supported-platforms)
  * [Variables](#variables)
  * [Task Summary](#task-summary)
  * [Output Files](#output-files)

## Supported Platforms
Any network device with a corresponding Ansible `*_command` module can be
supported. The playbook currently provides Ansible task files for
the various network operation systems shown in the list below.
To add a new device type, create a new task file in the
`tasks/` folder along with a corresponding `group_vars/` and inventory entry.
These are all easy tasks given the abstract architecture of this playbook.

Testing was conducted on the following platforms and versions:
  * Cisco Catalyst 8000v, version 17.9.2, running in Cisco DevNet Sandbox
  * Cisco CSR1000v, version 17.3.3, running in AWS
  * Cisco CSR1000v, version 16.12.01a, running in AWS
  * Cisco CSR1000v, version 16.9.3, running in Cisco DevNet Sandbox
  * Cisco XRv9000, version 6.3.1, running in AWS
  * Cisco XRv9000, version 7.3.2, running in Cisco DevNet Sandbox
  * Cisco ASAv, version 9.9.1, running in AWS
  * Cisco ASAv, version 9.16.1, running in AWS
  * Cisco Nexus 3172T, version 6.0.2.U6.4a, hardware appliance
  * Cisco Nexus 9000v, version 9.3(3), running on VMware ESXi
  * Cisco Nexus 9000v, version 9.3(3), running in Cisco DevNet Sandbox
  * Cisco AireOS vWLC, version 8.3.143.0, running on VMware ESXi
  * Juniper vMX, version 18.4R1, running in AWS
  * Arista vEOS, version 4.22.1FX, running in AWS
  * Mikrotik RouterOS, version 6.44.3, running in AWS
  * F5 BIGIP, version 16.0.1.1, running in AWS
```
$ cat /etc/redhat-release
Red Hat Enterprise Linux Server release 7.4 (Maipo)

$ uname -a
Linux ip-10-125-0-100.ec2.internal 3.10.0-693.el7.x86_64 #1 SMP
  Thu Jul 6 19:56:57 EDT 2017 x86_64 x86_64 x86_64 GNU/Linux

$ ansible --version
ansible [core 2.11.12] 
  config file = /home/ec2-user/racc/ansible.cfg
  configured module search path = ['/home/ec2-user/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/ec2-user/environments/racc/lib/python3.7/site-packages/ansible
  ansible collection location = /home/ec2-user/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/ec2-user/environments/racc/bin/ansible
  python version = 3.7.3 (default, Aug 27 2019, 16:56:53) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
  jinja version = 3.1.2
  libyaml = True
```

## Getting Started
Follow these instructions to quickly get started with `racc`. This
README assumes you have Python and `pip` installed:
  1. Install Python packages: `pip install requirements.txt`
  2. Install Ansible collections: `ansible-galaxy collection install -r requirements.yml`
  3. Edit `hosts.yml`, the inventory file to suit your network
  4. Edit the relevant `group_vars/` files based on your devices
  5. Run the playbook: `ansible-playbook racc_playbook.yml`

## Variables
The `group_vars/all.yml` file contains connectivity parameters common to all
network modules. These typically use `network_cli`. Note that if the legacy method
`provider` method is used for some devices, it must first be defined. Then,
individual task files for each platform must set `ansible_connection: local`
in addition to manually specifying `provider: "{{ provider }}"` as a task
suboption in most cases.

There are some minor variables that control the playbooks output products
and formatting. These are not often modified:
  * `remove_files`: A boolean true/false question that governs whether items
    in the `files/` folder are removed after a playbook run. Normally, this
    is set to `true` because the final archive is what really matter, and
    retaining all the uncompressed text files on the control machine is not
    a long-term solution. Setting this to `false` is useful for quick tests
    or troubleshooting.
  * `hash_type`: A string that specifies what hash algorithm to use when
    computing the hash of the contents of each file. These are written to
    a consolidated CSV file on a per-device basis. The algorithms available
    vary based on your installed Python `hashlib` version, but simple
    algorithms like `"md5"` and `"sha1"` are recommended. You can set this
    value to `false` to skip the procecss of generating hashes, which will
    cause the playbook to run a little faster.
  * `newline_sequence`: Determines what kind of line terminator to use for
    output files. See the Ansible documentation for a full list of options.
  * `archive_format`: Determines what kind of archive to create when the
    `archive` moduel is called. Any option supported by the version of Ansible
    in use can be used here. Some examples include zip, gz, bz2, xz, and tar
    as of Ansible 2.5. The zip format is usually appropriate when transferring
    to Windows-based SCP servers, and that is the default.
  * `scp`: Nested dictionary containing two subkeys below.
    * `user`: The SCP username that can write files to the SCP server.
    * `host`: The SCP server FQDN/IP address. Under its root directory, a
      directory called `racc/` should be created. This is where the archives
      generated by this playbook are copied. This playdoes __does not__
      automatically perform the copying.

There are independent `group_vars/` files for each network device type:
routers, switches, and firewalls, and wireless LAN controllers for example.
Each one has a key of `command_list` which specifies a list of commands
specific to that product's CLI. After those commands are executed on
a device, a filename equal to the command (spaces are replaced with
underscores) is generated. CLI output redirection (pipes) are supported.
It is recommended to type the entire command into the `command` value
since these are printed to stdout during execution. Using abbreviated
commands may confuse operators less familiar with some network CLIs.

Note that these are just examples and it is common to adjust the
`command_list` on a per-run basis. For example, if collecting all routing
tables is important to quickly troubleshoot a routing loop, simply add
`show ip route` to the list, and run the playbook.

## Task Summary
To make the playbook easier to read and troubleshoot, whenever a command is
issued on a device, both the inventory hostname and the command being issued
are printed to stdout. The example below shows a sample run with a variety
of devices.

```
TASK [ASA >> Gather Cisco ASA information]
ok: [asav1]

TASK [IOSXR >> Gather Cisco IOS-XR information]
ok: [xrv1]

TASK [IOS >> Gather Cisco IOS/IOS-XE information]
ok: [csr1]
ok: [csr2]
```

## Output Files
At the end of the playbook, assuming that `remove_files` is set to
`false` for the purpose of discussion, the following filesystem
components are created.

For each host against which this playbook runs, N files are generated,
where N is the number of elements in the relevant `command_list`.
Ansible inventory hostname and the date/time group (DTG) in UTC are
included in the file names for easy identification. All files are written
to `files/` directory and are ignored by git. The format of all text files is
`<command_that_was_run>.txt` while the parent directory format is
`<hostname>_<dtg>/`.

```
$ tree --charset=asci files/
files/
|-- asav1_20210629T142431
|   |-- show_inventory.txt
|   |-- show_running-config.txt
|   `-- show_version.txt
|-- chr1_20210629T142431
|   |-- export.txt
|   |-- system_license_print.txt
|   `-- system_resource_print.txt
|-- csr1_20210629T142431
|   |-- show_inventory.txt
|   |-- show_license_all.txt
|   |-- show_running-config.txt
|   `-- show_version.txt
|-- csr2_20210629T142431
|   |-- show_inventory.txt
|   |-- show_license_all.txt
|   |-- show_running-config.txt
|   `-- show_version.txt
|-- n9kv1_20210629T142431
|   |-- show_install_active.txt
|   |-- show_inventory.txt
|   |-- show_license_usage.txt
|   `-- show_version.txt
|-- veos1_20210629T142431
|   |-- show_inventory.txt
|   |-- show_license_all.txt
|   |-- show_running-config.txt
|   `-- show_version.txt
|-- vmx1_20210629T142431
|   |-- show_configuration.txt
|   |-- show_interfaces_brief.txt
|   |-- show_system_errors_count.txt
|   `-- show_system_license.txt
`-- xrv1_20210629T142431
    |-- show_inventory.txt
    |-- show_license_all.txt
    |-- show_running-config.txt
    `-- show_version.txt
```

The actual text output is shown below. You can use the `head -n-0`
command to view all the outputs from a given device, which prints
the filename at the start of each output, making it easy to determine
where one output ends and another begins.

```
$ head -n-0 samples_nohash/csr1_20210629T142431/*
==> samples_nohash/csr1_20210629T142431/show_inventory.txt <==
NAME: "Chassis", DESCR: "Cisco CSR1000V Chassis"
PID: CSR1000V          , VID: V00  , SN: 92ASWZPKBOY

NAME: "module R0", DESCR: "Cisco CSR1000V Route Processor"
PID: CSR1000V          , VID: V00  , SN: JAB1303001C

NAME: "module F0", DESCR: "Cisco CSR1000V Embedded Services Processor"
PID: CSR1000V          , VID:      , SN:

==> samples_nohash/csr1_20210629T142431/show_license_all.txt <==
Smart Licensing Status
======================

Smart Licensing is ENABLED

Registration:
  Status: UNREGISTERED
  Export-Controlled Functionality: NOT ALLOWED

License Authorization:
  Status: No Licenses in Use

Export Authorization Key:
  Features Authorized:
    <none>

Utility:
  Status: DISABLED

Data Privacy:
  Sending Hostname: yes
    Callhome hostname privacy: DISABLED
    Smart Licensing hostname privacy: DISABLED
  Version privacy: DISABLED

Transport:
  Type: Callhome

Miscellaneous:
  Custom Id: <empty>

License Usage
=============

No licenses in use

Product Information
===================
UDI: PID:CSR1000V,SN:92ASWZPKBOY

Agent Version
=============
Smart Agent for Licensing: 5.0.6_rel/47

Reservation Info
================
License reservation: DISABLED

==> samples_nohash/csr1_20210629T142431/show_running-config.txt <==
Building configuration...

Current configuration : 10447 bytes
!
! Last configuration change at 12:00:08 UTC Mon Jun 28 2021 by admin
!
version 17.3
service timestamps debug datetime msec
service timestamps log datetime msec
[snip, running-config output omitted for brevity]
```

If `hash_type` is not a false-y value (such as Boolean `false` or `null`),
you'll see a `*_hashes.txt` file as well. This file contains a two-column
CSV file with the hash first and the filename second. You can double-check
the hash computations using CLI commands such as `md5sum`, which should
match the hashes computed by Ansible.

```
$ cat md5_hashes.txt
764c8f490b9038afe5a618854bb21852,files/csr1_20230811T074823/show_running-config.txt
fcd7bfac12290c1d7bd94bc195504d37,files/csr1_20230811T074823/show_inventory.txt
d3eaa829011e49cf04603ee926208a17,files/csr1_20230811T074823/show_license_all.txt
705640cd7519ce509b373f3243c45688,files/csr1_20230811T074823/show_version.txt

$ column -s, -t md5_hashes.txt
764c8f490b9038afe5a618854bb21852  files/csr1_20230811T074823/show_running-config.txt
fcd7bfac12290c1d7bd94bc195504d37  files/csr1_20230811T074823/show_inventory.txt
d3eaa829011e49cf04603ee926208a17  files/csr1_20230811T074823/show_license_all.txt
705640cd7519ce509b373f3243c45688  files/csr1_20230811T074823/show_version.txt

$ md5sum show*
fcd7bfac12290c1d7bd94bc195504d37  show_inventory.txt
d3eaa829011e49cf04603ee926208a17  show_license_all.txt
764c8f490b9038afe5a618854bb21852  show_running-config.txt
705640cd7519ce509b373f3243c45688  show_version.txt
```

Finally, the playbook prints out a shell command that can be copy/pasted by
the user to quickly SCP the archive to an SCP server, assuming an FQDN/IP
has been specified for it. This is useful for those unfamiliar with the
Linux `scp` command.

`scp archives/commands_20180603T070140.zip nick@192.0.2.1:/racc/`
