Vokomokum Ansible deployment
================================

- [Vokomokum Ansible deployment](#vokomoum-ansible-deployment)
- [Introduction](#introduction)
- [Roles overview](#roles-overview)
- [Common tasks](#common-tasks)
  - [Check and restart a service](#check-and-restart-a-service)
  - [Adding a new foodcoop](#adding-a-new-foodcoop)
  - [Deleting a foodcoop](#deleting-a-foodcoop)
  - [Adding a member to the hosting team](#adding-a-member-to-the-hosting-team)
  - [Run rails commands](#run-rails-commands)
  - [Recreating the demo database](#recreating-the-demo-database)
  - [List the date of latest activity per instance](#list-the-date-of-latest-activity-per-instance)

# Introduction
In this folder you'll find a couple of [Ansible](https://www.ansible.com) roles to setup and manage the
infrastructure for [Vokomokum platform](https://vokomokum.nl)

To use this roles you have to install these packages:
```Shell
apt install ansible ansible-mitogen
```
Our setup depends on external roles that are defined in this repository's `requirements.yml`. To install them run:
```Shell
ansible-galaxy install -r requirements.yml
```
The external roles are installed with a `galaxy-` prefix. Because there are no automatic updates for external roles you have to do it by yourself:
```Shell
ansible-galaxy install -r requirements.yml --force
```

We don't want to save internal data as clear text in this roles. For data encryption we make use of
[ansible-vault](https://docs.ansible.com/ansible/latest/cli/ansible-vault.html). To complete your Ansible setup just create a file called `.vault_pass` at the same level as this Readme file and include the vault password from our password database in this file. All variables that make use of the vault start with a prefix `vault_`. You will find them in either in the `group_vars` or in a `host_vars` directory in a file named `vault.*.yml`. To edit such a variable use a command like this:
```Shell
ansible-vault edit host_vars/focone/vault_foodcoops.yml
```

Have a look at a role's directory to find out more details on how we implement the global Foodsoft platform.

You can execute a role by using the corresponding [playbook](https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html):
```shell
ansible-playbook playbooks/foodsoft.yml
```
# Roles overview
| Name | Description |
|------|-------------|
| `basic-server` | Initial setup for a new server |
| `configure-foodsoft` | Configuration of the Foodsoft, Add and delete new foodcoops |
| `configure-sharedlists` | Configuration of Sharedlists |
| `galaxy/foodsoft` | Installation and updating of the Foodsoft |
| `galaxy/sharedlists` | Installation of the Sharedlists |
| `galaxy/senselab.borgbackup` | Configure (borg)backups |
| `galaxy/senselab.configure-firewall` | Configure UFW firewall |
| `galaxy/foodsoft` | Role to install the Foodsoft |
| `galaxy/senselab.dehydrated` | Installation and initial setup of dehydrated |
| `galaxy/senselab.mariadb` | Installation and initial setup of MariaDB |
| `galaxy/senselab.nginx` | Installation and configuration of Nginx |
| `galaxy/senselab.postfix` | Installation and configuration of Postfix for use with a real mail domain |
| `galaxy/sharedlists` | Role to install the Sharedlists application |
| `phpmyadmin` | Installation and configuration of phpMyAdmin |

# Common tasks
## Check and restart a service
We use different (systemd) services at focone:

| Name | Description |
|------|-------------|
| `foodsoft-web` | Foodsoft web interface |
| `foodsoft-smtp` | Foodsoft smtp server for incoming emails |
| `foodsoft-resque` | Redis-based [backend](https://github.com/resque/resque) for background jobs |
| `sharedlists-web` | Sharedlists web interface |
| `sharedlists-smtp` | Sharedlists smtp server for article updates by email |

You can check the status of a service with a command like this:
```shell
systemctl status foodsoft-web
```
If something seams to be wrong have a look at the log file, e.g:
```shell
journalctl -u foodsoft-web
```
There is also the [Monit](https://mmonit.com/monit/) output that tells you what's going on:
```Shell
monit status
```
Monit tries to restart a service in case of a failure. You can restart a service manually with systemd:
```shell
systemctl restart foodsoft-web
```

## Adding a new foodcoop
- Gather all [information](https://foodcoops.net/.global-foodsoft-platform/#request-a-new-instance). You need at least the *short name* and the mail addresses of the *contact persons*.
- Unlock the Ansible vault with:
   ```Shell
   ansible-vault edit host_vars/focone/vault_foodcoops.yml
   ```
- Edit the data in the vault:
  ```Shell
  ansible-vault edit host_vars/focone/vault_foodcoops.yml
- Add to the section `vault_foodcoops` the following mandatory settings:
   | Setting | Type | Value |
   |---------|------|-------|
   | `name` | string | short name of the foodcoop |
   | `database` | string | prefix `foodsoft_` + short name |
- For administrational reason we save also the contact persons mail addresses:
   ```YAML
   contacts:
     - name: alice
       mail: alice@example.org
     - name: bob
       mail: bob@example.org
   ```
- Some custom [configuration](roles/configure-foodsoft/Configuration.md) settings are available. At the moment every key-value-pair is supported by this role. Just add them to the `config` dictionary as shown in the example below.
- Example configuration:
   ```YAML
   vault_foodcoops:
     - name: demo
        database: foodsoft_demo
        contacts:
          - name: alice
            mail: alice@example.org
          - name: bob
            mail: bob@example.org
        config:
          stop_ordering_under: 70
          use_apple_points: false
          use_nick: true
          use_messages: false
   ```
- Upload the changes to our Git repository.
- Execute the playbook with:
   ```shell
   ansible-playbook playbooks/foodsoft.yml --tags multi_coop,foodsoft_app_config
   ```
- Immediately login with `admin` / `secret1234` and change the user details and password. The `admin` user should become the user account of the first contact person, so use their email address here. We do not want to encourage an unused `admin` account.
- You may want to pre-set some configuration if you know a bit more about the foodcoop. It's always helpful for new foodcoops to have a setup that already reflects their intended use a bit. At least you should set a time zone.
- Send an email to the foodcoop's contact persons with the url and admin account details.
- Please also communicate that this platform is run by volunteers from participating food cooperatives and depends on donations.
- Add the two contact persons to our foodsoft announce mailing list.

## Deleting a foodcoop
If the deletion of a foocoop is requested follow these steps:

- Find the foodcoops's configuration at `host_vars/focone/vault_foodcoops.yml`. Enter another entry called `deleted: true` to the array:
   ```yaml
   vault_foodcoops:
     - name: mycoop
       database: foodsoft_mycoop
       deleted: true
   ```
- Execute the playbook:
   ```shell
   ansible-playbook playbooks/configure-foodsoft.yml --tags never,foodcoop_delete
   ```
   Removing a Foodcoop will result in a restart of the Foodsoft service. Please execute the playbook in the late evening or preferably during the night to not disturb our users.
- Delete the foodcoop's entry from the vault file.
- Upload the changes to our Git repository.
- Delete the two contact persons from our foodsoft announce mailing list.

## Adding a member to the hosting team
- Add to Github [operations team](https://github.com/orgs/foodcoops/teams/operations)
- Add to relevant [mailing lists](https://lists.foodcoops.net) (support@lists.foodcoops.net and foodsoft-global-hosting@lists.foodcoops.net)
- Obtain user's SSH key and verify it from a Github gist, Keybase or a video call.
- Add SSH key file to `roles/basic-server/ssh_authorized_keys/admin` and start ansible:
    ```shell
    ansible-playbook playbooks/basic-server.yml --limit focone
    ```
- Add the admin's email address to `host_vars/focone/vault_postfix.yml`, save the file and run the postfix playbook:
    ```Shell
    ansible-playbook playbooks/postfix.yml
    ```

## Run rails commands
Usually you have to pass at least these environment variables to make `rails` [commands](https://guides.rubyonrails.org/command_line.html) work:
- `RAILS_ENV`
- `REDIS_URL` (Foodsoft only)
- `SECRET_KEY_BASE`
  
Additionally your need to run `rails` in the context of `rbenv`. This results in a very long command line argument.  You may use the bash [aliases](https://github.com/foodcoops/foodcoops.net/commit/a3b9818644e14eaa51af4a4e5db8d38a4474f59a) `foodsoftctl` and `sharedlistsctl` to shorten the `rails` commands.

## Recreating the demo database
It can sometimes be useful to manually reset the demo instance with a new database, seeded from `small.en`. Run the following command as the Foodsoft system user:
```shell
DISABLE_DATABASE_ENVIRONMENT_CHECK=1 DATABASE_URL=mysql2://foodsoft:$DATABASE_PASSWORD@localhost/foodsoft_demo \  
  foodsoftctl db:purge db:schema:load db:seed:small.en && foodsoftctl tmp:cache:clear
```

## List the date of latest activity per instance
You can intenify the date of the latest user activity of all Foodsoft instances:
```Shell
cd /opt/foodsoft

su _foodsoft

foodsoftctl r 'FoodsoftConfig.foodcoops.each{|s| FoodsoftConfig.select_foodcoop(s) ; \
  puts "#{User.maximum(:last_activity).rfc3339} #{s}"}' | sort
```
