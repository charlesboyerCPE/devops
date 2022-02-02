# TP3 : Ansible

## BOYER Charles

---

## Intro

---
Fichiers de base :

- /etc/ansible/hosts : Fichier permettant de spécifier les hosts
- inventories/setup.yml : Fichier permettant de créer un inventaire

Commande de bases :

- "--become" permet de devenir super-user
- ansible all -m -ping : Permet de pinger toutes les machines
- ansible all -m yum -a “name=<SERVICE>” --private-key=<path_to_your_ssh_key> -u centos : Permet d'installer un services sur une machine
- ansible all --list-hosts : Permet de lister tous les hosts présents
- ansible all -i inventories/setup.yml -m ping : Permet lancer le test de l'inventaire en pingant la machine esclave

---

## Playbooks

---

### First playbook

Contenue playbook.yml :

```yaml
- hosts: all
  gather_facts: false
  become: yes

  tasks:
    - name: Test connection
      ping:
```

Commandes réalisés :

- --syntax-check : "Permet de checker la syntaxe"
- ansible-playbook -i inventories/setup.yml playbook.yml : Permet de lancer un playbook

### Advanced playbook

```yaml
- hosts: all
  gather_facts: false
  become: yes

# Install Docker
  tasks:
  - name: Clean packages
    command:
      cmd: dnf clean -y packages

  - name: Install device-mapper-persistent-data
    dnf:
      name: device-mapper-persistent-data
      state: latest
  
  - name: Install lvm2
    dnf:
      name: lvm2
      state: latest

  - name: add repo docker
    command:
      cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker
    dnf:
      name: docker-ce
      state: present

  - name: Install Python3
    dnf:
      name: python3

  - name: Pip Install
    pip:
      name: docker

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker
```

Commandes réalisés :

- ansible-playbook -i inventories/setup.yml playbook.yml : Permet de lancer un playbook
- ansible-galaxy init roles/docker : Permet de récupérer le module : rôle Docker

---

## Deploy your app

---

Création des différents rôles :

- ansible-galaxy init roles/proxy
- ansible-galaxy init roles/app
- ansible-galaxy init roles/network
- ansible-galaxy init roles/database

Playbook.yml final :

```yaml
- hosts: all
  gather_facts: false
  become: yes
  roles:
    - docker
    - network
    - database
    - app
    - proxy
```
