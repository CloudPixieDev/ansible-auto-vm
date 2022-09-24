# Ansible for Rapid VMs and Deployment

Simple goals of this project:

* Minimally modify non-linux host platform (macOS)
* Rapidly configure a guested VM (hosted by Fusion or VirtualBox)
* Update and install dependencies on the guest VM OS
* Create a standard 'auto' user account
* Provide password-less ssh access to the VM for auto user using host user's pub ssh ID
* Configure auto user with  password-less sudo access on the VM
* Configure tmux and screen for persistent console sessions (and integration with iTerm2)
* Configure vagrant-libvirt on the VM
* Configure k3s on the VM

To get started we need Ansible on MacOS

## Ansible on MacOS

This is just a note on getting Ansible on a MacOS Monterey host quick as possible.

### Install Homebrew

According to the web page https://brew.sh/

	/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

#### tmux for iterm2

Quick side note for iTerm2 users...

	brew install tmux

and do more with iTerm and remote sessions following these write ups:

	https://gitlab.com/gnachman/iterm2/-/wikis/TmuxIntegration
	https://gitlab.com/gnachman/iterm2/-/wikis/tmux-Integration-Best-Practices

If you follow steps above you'll have lines like this at the end of your ~/.zshrc file

	export ITERM_ENABLE_SHELL_INTEGRATION_WITH_TMUX=YES
	test -e "${HOME}/.iterm2_shell_integration.zsh" && source "${HOME}/.iterm2_shell_integration.zsh"

#### pyenv for latest python3 and pip3

Monterey doesn't have the latest of either, but don't stomp the system python or pip. Add pyenv

	brew install pyenv xz

#### brew results:

	brew list

Example output

	==> Formulae
	autoconf	ca-certificates	libevent	m4		ncurses		openssl@1.1	pkg-config	pyenv		readline	tmux		utf8proc	xz
	
	==> Casks
	rectangle

Run this command to modify the env

	echo 'eval "$(pyenv init --path)"' >> ~/.zprofile
	echo 'eval "$(pyenv init -)"' >> ~/.zshrc
	
now check for this at end of ~/.zprofile

	eval "$(pyenv init --path)"

and at end of ~/.zshrc:

	eval "$(pyenv init -)"
	
#### brew maintance

Update all package definitions (formulae) and Homebrew itself

	brew update

List installed packages (kegs) which are outdated with

	brew outdated

Upgrade outdated casks with

	brew upgrade

### Use pyenv to install latest python and pip

#### Find the latest

	pyenv install --list | grep -e '^[ \t]*3.' | grep -v -e rc -e dev | tail -n 1

Example output

	3.10.6

#### Install and activate it

	pyenv install 3.10.6
	pyenv global 3.10.6

#### pyenv results:

	pyenv versions

Example output

	  system
	* 3.10.6 (set by /Users/.../.pyenv/version)
  
### Use pip to upgrade pip 

	pip install --upgrade pip
	
### Install ansible

	pip install --user ansible

#### prepend to path in .zprofile

Add these to the ond of your .zprofile and then source it

	echo 'path=("/Users/$USER/.local/bin" $path)' >> ~/.zprofile
	echo 'export PATH' >> ~/.zprofile
	source ~/.zprofile

#### which ansible

	which ansible
	ansible --version


## Deploy VM with Host platform

Using VMware Workstation or Fusion or Oracle VirtualBox create a Linux Guest VM.

I use openSUSE Leap from their appliance builds:

	https://download.opensuse.org/distribution/leap/15.4/appliances/

If you create a new VM using the custom option, then you can have it copy the uncompressed VMDK from here.

When booting this runs a setup dialoge ans asks for the 'root' password. 


## Run the playbooks

Boot the VM and get the IP

### ssh-copy-id

Copy your host user public ssh key over to guest VM installed user authorized_keys with:

	ssh-copy-id root@<vm-ip-addr>

Enter the root password provided to the dialoge earlier.

### Configure ansible vault for guest_sudo_pass

	ansible-vault create vault.yml

When prompted enter your vault password and in the editor add

	---
	guest_sudo_pass: <guest_sudo_password>

Where `<guest_sudo_password>` is the password provided to the dialoge earlier.


### Create a vault password file

To keep from being asked for the vault password with every ansible playbook invocation

Create a `~/.vault_pass` text file with a single line with the password used in `create` step above and save it.

Make sure the filename is as the ansible.cfg file expects or change it from current setting:

	vault_password_file = ~/.vault_pass

### Configure `hosts` file with guest VM IP

If <vm-ip-addr> is 192.168.108.133 we'll use that after `[guest]` at
the end of the `hosts` file used by ansible

	[guest]
	192.168.108.133

### Test access

	ansible -u root -m ping guest

Expected example output

	192.168.108.133 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}

### Configure `hosts` file with hostname and user become 

This is also where you would change the desired hostname of the guest
VM from `playground` to any valid DNS hostname:

	[guest:vars]
	guest_hostname=playground

### group_vars/guest.yml

This vars file has some settings for non-root become settings that use the password stored in the vault

	become_user=root
	become_method=sudo
	become_pass='{{ guest_sudo_pass }}'

### Run the `playground` playbook

	ansible-playbook -u root playground.yml


#### Ansible access as user `auto` and privleged ansible

Now ansible access to the guest VM can be done without password, vault or otherwise.

For example:

	ansible -m ping guest
	
	192.168.108.133 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}

and privleged:

	ansible -b -m ping guest
	
	192.168.108.133 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}

#### Non-Ansible access:

for `auto`:

	ssh auto@playground.local
	Have a lot of fun...

for privleged access:
	
	auto@playground:~> ls -al /root
	ls: cannot open directory '/root': Permission denied
	
	auto@playground:~> sudo ls -al /root
	total 4
	drwx------ 1 root root  76 Sep 19 13:36 .
	drwxr-xr-x 1 root root 156 Sep 19 12:47 ..
	drwx------ 1 root root   6 Sep 19 13:12 .ansible
	-rw------- 1 root root  61 Sep 19 13:36 .bash_history
	drwxr-xr-x 1 root root   0 Mar 15  2022 bin
	drwx------ 1 root root   0 Mar 15  2022 .gnupg
	drwxr-xr-x 1 root root  36 Sep 19 08:37 inst-sys

## Starting with a VM Laboratory

Assuming you have a VM playground as a 'lab,' make sure the hypervisor has the following setting under `Processors` in the `Advanced optinos`

	 Enable hypervisor applications in this virtual machine

Proceed to install [Golang](https://go.dev/) and [Boots](https://docs.tinkerbell.org/services/boots/), the PXE utilities platform for [Tinkerbell](https://tinkerbell.org/) project from [Equinix Metal](https://metal.equinix.com/product/tinkerbell-oss/), on an openSUSE Leap VM.


### Disable mitigatinos for this VM LAB

Introduce 'mitigations=off' to /etc/default/grub so that we can have KVM instances in the VM (and even nested for more play).

	ansible-playbook kickboots.yml -t mitigations-off
	

### Add Golang

Using zypper [devel:languages:go](https://download.opensuse.org/repositories/devel:/languages:/go/) repo from [OBS](https://build.opensuse.org/project/show/devel:languages:go)

We'll use ansible to add the repo and install go.

#### Manual version

	sudo zypper ar -f https://download.opensuse.org/repositories/devel:/languages:/go/15.4/devel:languages:go.repo
	sudo zypper --gpg-auto-import-keys -n ref
	sudo zypper in -y go

Results in something like this if you're curious

	The following 25 NEW packages are going to be installed:
	  binutils cpp cpp7 gcc gcc7 glibc-devel go go1.19 libasan4
	  libatomic1 libcilkrts5 libctf0 libctf-nobfd0 libgomp1
	  libisl15 libitm1 liblsan0 libmpc3 libmpfr6 libmpx2
	  libmpxwrappers2 libtsan0 libubsan0 libxcrypt-devel
	  linux-glibc-devel

	Overall download size: 185.6 MiB. Already cached: 0 B. After the operation, additional 835.6 MiB will be used.

#### Ansible golang role and tag

	ansible-playbook kickboots.yml -t golang

### Add Boots

Github [latest](https://github.com/tinkerbell/boots/releases/latest) release is here. But it's only released as a tarball.

### Add libvirt

Similar to golang using OBS repos

	https://download.opensuse.org/repositories/Virtualization/15.4/Virtualization.repo

