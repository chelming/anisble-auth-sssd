auth-sssd
========

Ansible role to configure RHEL and derivates to use Microsoft Active Directory for authentication. 

Requirements
------------

The role joins the machine in AD with Samba to generate the Keytab for GSSAPI authentication so a user with sufficient privileges to join is required.

This has only been tested on RHEL 7. There are updates in the more recent versions of [SSSD (1.14+)][1] that allow for [GPO based access control][2].

[1]: https://fedorahosted.org/sssd/wiki/Releases
[2]: https://fedorahosted.org/sssd/wiki/DesignDocs/ActiveDirectoryGPOIntegration

Role Variables
--------------

The role uses the following variables, which you should override:
* `auth_sssd_domain` - The domain name lowercase.
* `auth_sssd_bind_path` - The bind path in the following format: `Computers/Department/Linux Computers`.
* `ad_gpo_access_control` - Whether to log (`Permissive`) or enforce (`Enforcing`) GPO based access.

Additionally, the credentials used to bind should be encrypted using Ansible-Vault.
* `auth_sssd_user` - The username with join privileges in the OU you are binding to.
* `auth_sssd_password` - The password.

Encrypying Credentials with ansible-vault
-----------------------------------------

Using the `ansible-vault` command you can encrypy the credentials used to bind the machine.

### Creating an Encrypted Variables File
To create an encrypted variables file, run the following command:
```
ansible-vault create bind_creds.yml
```

You will be prompted for a password and again to confirm, after which a file will open in your default editor. Once you are done with the editor session, the file will be saved as encrypted data using AES.

### Editing an Encrypted Variables File
To edit a previously encrypted variables file, use the `edit` command:
```
ansible-value edit bind_cred.yml
```

This will decrypt the file to a temporary file and allow you to edit the file, saving it back when done and removing the temporary file.

### Example bind_creds.yml
```
$ ansible-vault create bind_creds.yml
```
```
---
auth_sssd_user: "adminuser"
auth_sssd_password: "pa$$word"
```
```
$ cat bind_creds.yml
$ANSIBLE_VAULT;1.1;AES256
30343038636333643437383666336133353338636462653761653732373935333361323239303933
3535333232376631633133376334323736383864643835300a396137303462313538306561323464
35623161303336346263653839386361616438363734373939383064343339623064666461656131
3766383634626563370a323531333237336635656637366430386264643438336463363964636231
65313563376366613338383832333166383830343761366634656562613133653439336338353934
6566376666653863616535396232613762356265353635343235
```

### Using the encrypted bind_creds file
If your file is included in either a `site_vars`, `group_vars` or `host_vars` folder you will need to provide the password used to encrypt the file using either `--ask-vault-pass` or `--vault-password-file /path/to/plain_text_vault_pw.txt`

See the [ansible-vault documentation](http://docs.ansible.com/ansible/playbooks_vault.html) for more information.

GPO-based Access Control
------------------------
SSSD can retreive GPOs applicable to host systems and based on the configured logon rights, determine if a user is allowed to log on to a certain host.

### Configuring GPO-based Access Control
The `ad_gpo_access_control` variable specifies the mode in which the GPO-based access control runs. It can be set to the following values:
* `ad_gpo_access_control: "permissive"` - The permissive value specifies that GPO-based access control is evaluated but not enforced; a syslog message is recorded every time access would be denied. This is the default setting.
* `ad_gpo_access_control: "enforcing"` - The enforcing value specifies that GPO-based access control is evaluated and enforced.
* `ad_gpo_access_control: "disabled"` - The disabled value specifies that GPO-based access control is neither evaluated nor enforced.

The following parameters related to the GPO-based access control can also be specified in the `templates/sssd.conf.j2` file:
The `ad_gpo_map_*` options and the `ad_gpo_default_right` option configure which PAM services are mapped to specific Windows logon rights.
To move a service allowed by default to the deny list, remove it from the allow list. For example, to remove the `su` service from the allow list and add `my_pam_service`:
```
ad_gpo_map_interactive = -su, +my_pam_service
```
You can add and remove services from any of the `ad_gpo_map_*` options in the same way, using `+my_pam_service` and `-my_pam_service`.

#### GPMC Options
| SSSD Option   | GPMC value    | Default set of PAM service names  |
| ------------- |:-------------:| -----:|
| `ad_gpo_map_interactive`      | Allow log on locally | `login, su, su-l, gdm-fingerprint, gdm-password, gdm-smartcard, kdm, lightdm, lxdm, sddm, unity, xdm` |
| `ad_gpo_map_remote_interactive` | Allow log on through Remote Desktop Services | `ssh, cockpit` |
| `ad_gpo_map_network` | Access this computer from the network | `ftp, samba` |
| `ad_gpo_map_batch` | Allow log on as a batch job | `crond` |
| `ad_gpo_map_service` | Allow log on as a service | _not set_ |

#### Other Options
| SSSD Option   | Notes    | Default set of PAM service names  |
| ------------- |:-------------:| -----:|
| `ad_gpo_map_permit` | GPO-based access will always be granted, regardless of any GPO Logon Rights. | `polkit-1, sudo, sudo-i, systemd-user` |
| `ad_gpo_map_deny` | GPO-based access will always be denied, regardless of any GPO Logon Rights. | _not set_ |
| `ad_gpo_default_right` | This option defines how access control is evalueated for PAM service names that are not explicitly listed in one of the `ad_gpo_map_*` options. This option can be set in two different manners. First, this option can be set to use a default logon right. For example, if this option is set to `interactive`, it means that unmapped PAM service names will be processed based on the `InteractiveLogonRight` and `DenyInteractiveLogonRight` policy settings. Alternatively, this option can be set to either always permit or always deny access for unmapped PAM service names. | Supported values for this option include: `interactive, remote_interactive, network, batch, service, permit, deny`. Default: `deny`. |

See the [SSSD GPO Integration Design Documents](https://fedorahosted.org/sssd/wiki/DesignDocs/ActiveDirectoryGPOIntegration) for more information.

Example Playbook
-------------------------

    - hosts: workstations
      roles:
         - auth-sssd

