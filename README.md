## Overview

The Lab is a customizable Ansible playbook used to create, deploy and manage servers on AWS EC2 and your local machine. It supports multiple servers and environments with the use of variables.

## Stack

* AWS EC2
* Ubuntu 18.04
* RVM
* Apache/Passenger
* MySQL
* Solr
* RabbitMQ
* Redis
* Java
* Nodejs
* Prometheus
* Netdata

Of course, anything can easily be added as a role or task.

## Features

* Start to finish creation and provisioning of servers on AWS EC2
* Handles multiple servers
* Handles multiple environments
* Local environment with Vagrant
* Ansible testing with github actions
* Build and update playbooks
* Easy control with aliases

## Index

* [Local Usage](Local-Usage)
* [Web Usage](Web-Usage)
* [How It Works](How-It-Works)
* [Creating New Servers](Creating-New-Servers)
* [Dependencies](Dependencies)
* [Building Local Server](Building-Local-Server)
* [Updating](Updating)
* [Using Ansible Vault](Using-Ansible-Vault)
* [Testing](Testing)
* [Known Issues](Known-Issues)

## Local Usage

**Vagrant Commands:**

* **Build:** `vagrant up` or `lab-up`
* **Connect:** `vagrant ssh` or `lab-ssh`
* **Reload:** `vagrant reload` or `lab-reload`
* **Re-provision:** `vagrant provision` or `lab-provision`
* **Stop:** `vagrant halt` or `lab-halt`
* **Delete:** `vagrant destroy -f` or `lab-destroy`

_**Note:** All `vagrant` commands must be run in the `lab/local` directory._

**VM Tips & Tricks:**

* **Go to app directory:** `app` or `apps`
* **Go to enabled apache sites:** `vhosts`
* **View login message:** `motd`
* **File browser:** `ranger`
* **Reload dotfiles:** `sauce`
* **System Monitor:** `htop`
* **Netdata** real-time performance monitor in the browser at `<host_name>:19999`

**Resource Allocation:**

You can easily change resource allocation in the `lab/local/Vagrantfile`.

* `v.memory = <memory>` = Memory / RAM allocated to the VM
* `v.cpus = <cpu_cores>` = CPU Cores allocated to the VM

## Web Usage

## Web Usage

***

**Building Servers:**

`lab/web/main.yml` is used to build and manage servers and dependencies.

```
lab-build --extra-vars "server_name=<server_name> app_env=<app_env>"
```

or for UK servers (using UK server key pair location in alias):

```
lab-build-uk --extra-vars "server_name=<server_name> app_env=<app_env>"
```

*Or manually set variables in playbook:*
1. Set variables in `lab/web/vars.yml`
1. Set or uncomment host in `lab/web/hosts` under `[web]`
1. Run the playbook: `lab-build`

**Updating Servers:**

`lab/web/update.yml` can be edited to reprovision specific tasks.

1. Add `import_tasks` in `lab/web/update.yml`
1. Add `vars_files` to include in `lab/web/update.yml` if neccissary
1. Edit relevant variable files
1. Add host(s) to `lab/web/hosts` under `[web]`
1. Run the playbook:

```
lab-update -e "server_name=<server_name> app_env=<app_env>"
````

**Using Tags:**

Tags can be used to include or exclude specific commands within tasks.

1. Add `tags: <tag>` to tasks within `.yml` files.

```
- name: Include apache config vars
  include_vars:
    file: vars/configs/bashrc/bashrc_{{ app_env }}.yml
  tags: fix

- name: Remove original bashrc
  file:
    path: /home/{{ username | lower }}/.bashrc
    state: absent
  tags: fix
```

_(Don't forget to add tags to the variable include command at the start of the file if neccissary! They should be added in already but if there is a password issue this is the first thing to check...)_

2. Run playbook with `--tags` flag:

	* `lab-build --tags <tag>`
    * `lab-update --tags <tag>`
	* `lab-build --skip-tags <tag>`
    * `lab-update --skip-tags <tag>`

**Built-In Tags:**

`--skip-tags restart`: do not restart server on playbook completion

## Overview

**Environments**

* `lab/local` creates the testing environment on the local machine.

* `lab/web` creates the staging and production environments in AWS EC2.

Each directory has its own set of _Tasks_, _Playbooks_ and _Variables_.

**Tasks**

`lab/{env}/tasks/`

Commands run on the host to install dependencies. Each file represents a different dependency or configuration.

**Playbooks**

`lab/{env}/main.yml`

Connects to the host, reads variable files and imports the relevant tasks sequentially to run on the host.

**Variables**

`lab/{env}/vars.yml` is first variable file called by the ansible playbook. This file is where the user defines top level information that determines how the playbook is run and which variable files are used.

The `app_env` and `server_name` variables specified in `vars.yml` determine which server variable is called, which tasks to import in the `main.yml` playbook and are used throughout the tasks and config files.

`lab/{env}/servers` variables dictate server specific variables including AWS EC2 configuration, host names, app names and app versions. Each server file refers to an individual server or group of idential servers enforced by the `instance_name` variable. The `template.yml` file serves as a point of reference with explanations for each variable.

`lab/{env}/vars` variables are relevant to specific tasks are imported at the beginning of the task file.

`lab/{env}/vars/configs` are variables referring to server config files: apache, motd, bashrc, etc... Editing these and reprovisioning will properly update the host's relevant config file.

Files in the `certs`, `passwords` and `ssh_keys` directories shoudl be encrypted with Ansible Vault and are also gitignored.

**What happens when I run `lab-build`?**

This command will create and provision the relevant server on AWS defined by the user in the `lab/web/vars.yml` file:

1. First, the alias command is read from your local machine's `~/.profile`

```
ansible-playbook --ask-vault-pass -i ~/lab/hosts --key-file "~/key_pairs/<key>.pem" ~/lab/web/main.yml
```

2. This command tells Ansible to ask for the Ansible Vault password (`--ask-vault-pass`), points to the host file (`-i ~/lab/hosts`), points to the AWS key pair (`--key-file "~/key_pairs/<key>.pem"`), and  runs the `~/lab/web/main.yml` playbook.
3. You are asked for the Ansible Vault password. If correct, the `main.yml` playbook is run.
3. The `main.yml` playbook actually includes 3 seperate playbooks within it in order to connect to 3 different hosts.

```
TL;DR

Playbook 1: Creates the EC2 instance
Playbook 2: Installs python on the server
Playbook 3: Provisions the rest of the server dependencies and configs

you can now skip to vagrant...
```

For a more granular view... `main.yml`:

1. Connects to the local machine
1. Imports the `vars.yml` variable file
1. The `app_env` and `server_name` variables specified in `vars.yml` are used determine which server variable in `lab/web/servers` to import based on its file name.
1. Imports and runs the `ec2_create.yml` task which creates the relevant AWS EC2 instance with AWS CLI and adds it's IP address to the `lab/web/hosts` file under `[web]`.
1. The second playbook then connects to the host specified in the host file and installs python.
1. The third playbook connects to the host specified in the host file and imports the variables in the same way as the first playbook.
1. Tasks and roles are imported and ran sequentially on the host based on conditionals referring to the `server_name` variable in the `vars.yml` file.

**What happens when I run `lab-up`?**

1. The alias command is read from your local machine's `~/.profile`

```
cd ~/lab/local && vagrant up
```

2. This command `cd`s to the `lab/local` directory and tells Vagrant to run the `Vagrantfile`.

`vagrant up` can be split into two commands that refer to different parts of the `Vagrantfile`...

Part I: `vagrant reload`

3. Defines the virtual machine image
4. Defines the virtual networking
5. Defines the shared folders to sync: `~/apps`, `~/.ssh` & `~/dev_cert`
6. Defines the system resources
7. Creates the virtual machine in Virtualbox

Part II: `vagrant provision`

8. Reads the `lab/local/main.yml` Ansible playbook which imports the relevant task blocks and variables and provisions the virtual machine.

## Creating New Servers

1. Create file in `web/servers/` with the name `<app_env>_<server_name>.yml`.
1. Copy & paste the text in the `web/servers/template.yml` file and add the relevant variables for the new server.
1. Add server to conditionals in `web/main.yml` for relevant tasks.
1. Add server to `web/vars/configs/` files as neccissary.
1. Use [Ascii Maker](http://patorjk.com/software/taag/#p=display&f=Graffiti&t=Type%20Something%20) for the `web/vars/configs/motd/*.yml`

## Dependencies

### Setup SSH on your Local Machine

Generate ~/.ssh folder with id_rsa and id_rsa.pub files:

```
ssh-keygen
```

Add passphrase (if it exists) to keychain:

```
ssh-add -K
```

Copy public key:

```
cat ~/.ssh/id_rsa.pub | pbcopy
```

Create `~/.ssh/authorized_keys` and paste public key.

Create `~/.ssh/config` file and paste the text below:

```
Host *
AddKeysToAgent yes
IdentityFile ~/.ssh/id_rsa
```

### Setup Local Host File to View Local Environment in Browser

Copy the text below to `/etc/hosts` with app `<host_name>`. Feel free to change the ip address.

```
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1       localhost
255.255.255.255 broadcasthost
::1             localhost

192.168.0.42 <host_name>
```

### Setup Self-Signed SSL Cert for Local Environment HTTPS

Generate the key

```
mkdir ~/dev_cert
cd ~/dev_cert
openssl genrsa -des3 -passout pass:x -out server.pass.key 2048
openssl rsa -passin pass:x -in server.pass.key -out server.key
rm server.pass.key
openssl req -new -key server.key -out server.csr
```

Enter info prompted.. Enter the host name for common name.

Create a text file called `v3.ext`

```
vim v3.ext
```

Paste the following content into the file:
```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = %%DOMAIN%%
```
Change `%%DOMAIN%%` to be the _exact_ same value you used for the common name (AKA the hostname) above.

Now generate the certificate:
```
openssl x509 -req -sha256 -extfile v3.ext -days 365 -in server.csr -signkey server.key -out server.crt
```

Add certificate to the OSX keychain:

1. In Chrome, open the VM by navigating to the relevant hostname
1. `Option-Command-I` to open the Chrome Developer tools
1. Click the security tab.
1. Click the "View Certificate" button.
1. This opens a tab at the top of the browser where the certificate is shown. Drag the little picture of the certificate to your desktop. Close the tab.
1. Double-click the certificate icon now on your desktop. This will open the Keychain Access utility.
1. Be sure you add the certificate to the **System** keychain, not the login keychain.
   - If there are other "<host_name>" certificates in your keychain, you may want to delete them all first. Enter your OS X password to unlock it.
1. Once you are sure the new certificate is installed in the **System** keychain, double click it's row to open the details window, and under "Trust" set "When using this certificate" to "Always Trust". The other dropdown selectors will also show "Always Trust".
1. Close this details window. You may have to authenticate again.
1. Close Keychain Access program
1. Restart Chrome `chrome://restart`, and your self-signed certificate should be recognized now by the browser.

### Install Vagrant

[Download and install from website.](https://www.vagrantup.com/downloads.html)

Install vagrant plugins

```
vagrant plugin install vagrant-faster
```

### Install Virtualbox

[Download and install from website.](https://www.virtualbox.org/wiki/Downloads)


If having issues, follow this guide: [http://wiki.tldev2.com/books/devops/page/%F0%9F%92%8E-the-lab-%F0%9F%92%8E#bkmrk-vagrant-up-and-virtu](http://wiki.tldev2.com/books/devops/page/%F0%9F%92%8E-the-lab-%F0%9F%92%8E#bkmrk-vagrant-up-and-virtu)

### Install Ansible

```
sudo easy_install pip
sudo pip install ansible
sudo mkdir /etc/ansible
sudo touch /etc/ansible/hosts
sudo touch /etc/ansible/ansible.cfg
```

Install rvm role for lab ansible playbook.

```
ansible-galaxy install rvm.ruby
```

Add the text below to /etc/ansible/ansible.cfg. This enables clean and legible logs when running Ansible. _(optional)_

```
[defaults]
# Use the YAML callback plugin
stdout_callback = yaml
# Use the stdout_callback when running ad-hoc commands
bin_ansible_callbacks = True
# Disable host key checking
#host_key_checking = False
```

Add the text below to /etc/ansible/hosts

```
[local]
localhost
```

### AWS CLI

Create ~/.aws directory and files:

```
mkdir ~/.aws
vim ~/.aws/config
```

Install pip3:

```
brew install python3
brew postinstall python3

#If you encounter permissions issues do this:
sudo mkdir /usr/local/Frameworks
sudo chown $(whoami):admin /usr/local/Frameworks
```

Install aws-cli:

```
pip3 install awscli --upgrade --user
echo "export PATH=~/Library/Python/3.7/bin:$PATH" >> ~/.profile
```

Go to AWS dashboard > Create User > CLI Keys

Create ~/.aws directory and files:

```
mkdir ~/.aws
touch ~/.aws/config
touch ~/.aws/credentials
```

Add the text below to ~/.aws/config:

```
[default]
region = us-east-2
```

Add the text below to ~/.aws/credentials:

```
[default]
aws_access_key_id = <aws access key>
aws_secret_access_key = <aws secret key>
```

Install boto:

```
sudo pip install boto
```

### Install other app dependencies

Any other app dependencies like search and databases should be installed on your local machine and referred to in the config files or added as ansible tasks to `local/tasks`.

**Config File Notes:**

* Use this _magic_ ip for for any dependencies installed on the local machine: `10.0.2.2`
* App default location is `/var/www/vhosts/<app_name>`

### Git Clone the Repo

```
cd ~
git clone https://github.com/dop-amine/lab-template
mkdir ~/apps
cd ~/apps
git clone https://github.com/<username>/<app1>
git clone https://github.com/<username>/<app2>
```

### Add aliases to `~/.profile`

**For Developers:**

```
alias lab-up="cd ~/lab/local && vagrant up"
alias lab-ssh="cd ~/lab/local && vagrant ssh"
alias lab-reload="cd ~/lab/local && vagrant reload"
alias lab-provision="cd ~/lab/local && vagrant provision"
alias lab-halt="cd ~/lab/local && vagrant halt"
alias lab-destroy="cd ~/lab/local && vagrant destroy -f"
```

**For Sysadmin:**

```
alias lab-build="ansible-playbook --ask-vault-pass -i ~/lab/hosts --key-file "~/key_pairs/<key>.pem" ~/lab/web/main.yml"
alias lab-update="ansible-playbook --ask-vault-pass -i ~/lab/hosts --key-file "~/key_pairs/<key>.pem" ~/lab/web/update.yml"
```

## Building Local Server

Build and connect:

```
lab-up
lab-ssh
```

Thn bundle install ruby apps:

```
cd /var/www/vhosts/{{ app_name }}
bundle install
```

Restart apache:

```
sudo service apache2 restart
```

You can now access the local server in a web browser at the specified `host_name`.

## Using [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)

* **Creating encrypted files:** `ansible-vault create file.yml`
* **Editing encrypted files:** `ansible-vault edit file.yml`
* **Encrypting files:** `ansible-vault encrypt file1.yml file2.yml file3.yml`
* **Decrypting files:** `ansible-vault decrypt file.yml`

## Testing

### yamllint

Install

```
brew install yamllint
```

Config

```
vim ~/.yamllint
```

Copy and paste the text below:

```
---
rules:
  braces: enable
  brackets: enable
  colons: enable
  commas: enable
  comments: disable
  comments-indentation:
    level: warning
  document-end: disable
  document-start: disable
  empty-lines: enable
  empty-values: enable
  hyphens: enable
  indentation: enable
  key-duplicates: enable
  key-ordering: disable
  line-length: disable
  new-line-at-end-of-file: enable
  new-lines: enable
  octal-values: enable
  quoted-strings: disable
  trailing-spaces: enable
  truthy: disable
```

Run

```
cd ~/lab
yamllint -c ~/.yamllint .
```

### ansible-lint

Install

```
brew install ansible-lint
```

Run

```
ansible-lint ~/lab/web/main.yml
ansible-lint ~/lab/local/main.yml
```

## Web Server Manual Setup

### For ruby apps

Generate gemsets for app:

cd into directory or:

```
rvm use ruby-{{ ruby_version }} && rvm gemset create {{ app_name }} && rvm gemset use {{ app_name }}
```

Deploy with capistrano while testing (no elastic ip):

```
# Edit capistrano file to specify server ip
vim /home/deploy/{{ app_name }}/config/deploy/{{ app_env }}.rb
# change server to {{ username }}@<ip_address>
cd /home/deploy/{{ app_name}}
bundle exec cap {{ app_env }} deploy
```

Precomile assets (sometimes):

```
app
bundle exec rake clobber
bundle exec precompile:assets
```

server_name == ""
server_name != "" and server_name != ""
server_name == "" or server_name == ""
