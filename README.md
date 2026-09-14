# Linode Marketplace Helper Functions

The Linode Helper functions are static roles that can be imported into Marketplace playbooks to accomplish a particular system tasks. Examples of these include creating sudo users, generating SSL certificates and enabling a firewall.

## Helper Functions

| Name | Description | Actions
| :--- | :---        | :---
| UFW   | Add UFW firewalls to the Linode  | The UFW module will import a `ufw_rules.yml` provided in `roles/$APP/tasks` and enables the service.  |
| Certbot SSL   | Generates and sets auto-renew for Certbot SSL certificates  | The Certbot module installs Certbot Python plugin and certificates based on the webserver detected by Ansible. The default renewal cron runs Mondays at 00:00AM and can be manually edited. |
| Fail2Ban   | Installs, activates and enables Fail2Ban  | The Fail2Ban module installs, activates and enables the Fail2Ban service.   |
| Hostname   | Assigns a hostname to the Linode based on domains provided via UDF or uses default rDNS. | The Hostname module accepts a UDF to assign a FQDN and write to the `/etc/hosts` file. If no domain is provided the default `ip.linodeusercontent.com` rDNS will be used. For consistency, DNS and SSL configurations should use the Hostname generated `_domain` var when possible. |
| Secure MySQL  | Generates a root password for MySQL and performs standard hardening.  | The Secure MySQL module will use `passgen.yml` to generate a secure root password and write to `group_vars/linode/vars`. It will then update MySQL to be accessible by local socket or root password, and remove anonymous users, test databases and remote access.  |  
| Secure SSH   | Performs standard SSH hardening.  | The Secure SSH module writes to `/etc/ssh/sshd_config` to prevent password authentication and enable public key authentication for all users, including root.  |  
| Sudo User  | Creates limited `sudo` user with variable supplied username and password.  | Creates limited user from UDF supplied `username` and `password.` Note that usernames containing illegal characters will cause the play to fail. |
| SSH Key   | Writes SSH pubkey to `sudo` user's `authorized_keys`.  | Writes UDF supplied `pubkey` to `/home/$username/.ssh/authorized_keys`. To add a SSH key to `root` please use [Cloud Manager SSH Keys](https://www.linode.com/docs/products/tools/cloud-manager/guides/manage-ssh-keys/).   |
| Update Packages   | Performs standard apt updates and upgrades. | The Update Packages module performs apt update and upgrade actions as root.  |
| Data Exporter | Prometheus exporters that allow OS-level metrics collection | This modules allows the installation of several Promtheus exporters on the system by capturing UDF input as a list. This module allows the installation of `node_exporter` and `mysqld_exporter`. This modules creates a `prometheus` system user to run the service as. |
| Docker | Installs the latest version of Docker Edition | Use the `import_role` to use this module in your tasks. |
| Add-ons | Installs and manages optional Marketplace add-ons. | The Add-ons module creates `/etc/profile.d/addons.sh` from a template and includes task files for each supported add-on. Add-ons are enabled by listing them in the `add_ons` variable. Supported add-ons include `newrelic`, `node_exporter`, and `mysqld_exporter`. |
| gpu_utils | Installs Nvidia drivers on applications that require GPUs. This writes `gpu_devices`, `gpu_count`, and `_has_gpu` to the `group_vars/linode/vars`. |
| Certbot SSL Cluster | Configures SSL for a web HTTP endpoint in a clustered environment. | Delegated to a single host to perform domain validation (DV). Requires `host` and `webserver_task` vars; installs the Certbot Nginx plugin when `webserver_stack` is `lemp`. |
| CI Log | Reports deployment status to an external CI logging endpoint. | Authenticates against a Keycloak server using env-supplied credentials, then POSTs deployment status (app label, branch, image, region, etc.) to a data endpoint. |
| Create DNS Record | Creates a Linode DNS zone and A records for a domain/subdomain. | Uses the `linode.cloud` collection to create the zone (if missing) and A records pointing at the host's default IPv4 address, then waits for and verifies DNS propagation. |
| Database | Installs and configures a database engine based on the `database` variable. | Includes task files to install MySQL, MariaDB, PostgreSQL, or Redis depending on the selected `database` value. |
| Install DNS Utils | Installs the `dnsutils` package. | Ensures `dnsutils` (e.g. `dig`) is present on the system. |
| MongoDB | Installs and enables MongoDB. | Adds the MongoDB apt repository and GPG key for the `mongo_version` UDF, installs `mongodb-org`, and starts/enables the `mongod` service. |
| Open WebUI | Creates the initial admin account for Open WebUI. | Waits for the Open WebUI service to become healthy, then creates the admin account via the signup API using `openwebui_login_name`, `openwebui_login_email`, and `openwebui_login_password`. |
| Remove Provision Keys | Removes provisioning SSH keys after deployment. | Removes the provisioner entry from `/root/.ssh/authorized_keys` and deletes the local `id_ansible_ed25519` key pair, unless `debug` is defined. |
| Set Root Password | Propagates the root password from the provisioner to remote nodes. | Reads the root password hash from the provisioner's `/etc/shadow` and writes it to `/etc/shadow` on remote nodes. |
| SSH Key Cluster | Writes account SSH keys to root and sudo users across a cluster. | Copies the provisioner's `authorized_keys` file to `/root/.ssh/authorized_keys` and `/home/$username/.ssh/authorized_keys` when present. |

## Add-ons: Usage and Extending

The Add-ons module allows Marketplace apps to enable optional integrations (monitoring agents, exporters, etc.) in a modular way.

### How it Works
1. If the `add_ons` variable is defined (and not `"none"`), the `addons.sh` profile script is created from a template.  
2. The playbook loops through the defined add-ons and includes their respective task files.  
3. Each add-on only runs if its name is present in the `add_ons` list.  

### Adding a New Add-on
1. Create a new task file (e.g. `myaddon.yml`) inside the role’s `tasks/` directory.  
2. Add an entry to the add-ons loop in the playbook:  
   ```yaml
   - { name: "myaddon", file: "myaddon.yml" }

### Post Add-on installation steps

Some add-ons may require additional configuration after installation, such as providing API tokens, server-specific details, or other user inputs. To handle this, the deployment process creates a script at /etc/profile.d/addons.sh, which runs automatically on the first login after the Ansible playbook completes. This script is designed to support multiple add-ons by allowing configuration snippets to be appended as needed.

For example, you can insert a new add-on snippet into the existing file like this:
```
- name: Insert New add-on line after "# BEGIN ADDONS"
  lineinfile:
    path: /etc/profile.d/addons.sh
    insertafter: '^# BEGIN ADDONS'
    line: "{{ lookup('template', 'myaddon.sh.j2') }}"
    create: yes
    owner: root
    group: root
    mode: '0755'
```

## Creating Your Own

Additional Linode Helpers can be added while respecting [Ansible common practice](https://docs.ansible.com/ansible/2.8/user_guide/playbooks_best_practices.html) and directory structure. Linode Helpers should perform common, repeatable system configuration tasks with minimal dependancies. Linode Helper functions can be imported as roles as needed in playbooks. Please see [DEVELOPMENT.md](docs/DEVELOPMENT.md) for more detailed standards.
