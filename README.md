# Ansible_LAB

This project consists in a Vagrant + Ansible project that deploys a centralized authentication system based on OpenLDAP + MIT Kerberos. I developed it with the purpose of learning Ansible and prepare myself for the Red Hat EX294 exam.

The virtual machines can be launched using the following steps:

* Install Ansible, Vagrant and VirtualBox on the control machine.
* Have a CentOS 7 DVD ISO image downloaded at the path `${HOME}/Downloads/CentOS-7-x86_64.iso` (`$ wget --continue --timeout=30 --tries=0 -O ~/Downloads/CentOS-7-x86_64.iso http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Everything-1908.iso`).
* Clone this repository.
* Run `vagrant up` command from the local repository copy.

Network users created by playbook are named using [NATO phonetic alphabet](https://en.wikipedia.org/wiki/NATO_phonetic_alphabet). Their passwords are `student`.

The DVD ISO image is required because my Ansible playbook disables network Yum repositories in order to save bandwidth. Packages required from my Ansible playbook will be retrieved from it.

