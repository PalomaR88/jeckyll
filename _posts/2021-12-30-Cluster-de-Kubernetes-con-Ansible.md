---
title: Cluster de Kubernetes con Ansible
author: Paloma R. García Campón
date: 2021-12-30 00:00:00 +0800
categories: [Administración de sistemas operativos]
tags: [Ansible, Kubernetes]
---
# Cluster de Kubernetes con Ansible

A raíz del artítuclo [Kubernetes setup using Ansible and Vagrant](https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/) he decidido investigar acerca de la implementación de un cluster de Kubernetes con Ansible. Como anteriormente tenía un escenario con tres máquinas preparadas para ser usadas, omitiré la parte de Vagrant. Los nodos que forman parte de un cluster de Kubernetes deben cumplir unos requisitos mínimos, en mi caso, los 3 nodos cumplen las siguientes características:
- Debian 11
- 2GB de ram
- 2vCPU

## Inicio

El cluster estará formado por un nodo maestro y dos minios. Pero antes, se neccesita la configuración básica para usar Ansible, que es:
- Un anfitrión con Ansible instalado. 
- Confguración de un par de claves SSH.

Para estos pasos iniciales, se puede seguir mi guía [Instalación de Ansible](https://www.palomargc.com/posts/Instalacion-de-Ansible/). La versión de Ansible 2.12.1.


## Cominicación de Ansible para implementar Kubernetes

El primer paso debería ser configurar Ansible para comunicarse con los nodos de Kubernetes. Se crea un directorio y el fichero de hosts:
~~~
mkdir kubernetes
cd kubernetes
touch hosts
~~~

El fichero tendrá el siguiente contenido:
~~~
[masters]
master ansible_host=davinci ansible_user=root

[workers]
worker1 ansible_host=buonarroti ansible_user=root
worker2 ansible_host=sanzio ansible_user=root
~~~

Este ejemplo contiene la información de mis MV. Tanto los valores de host como de user deben cambiarse según el escenario.

Y se comprueba el funcionamiento con un ping de Ansible:
~~~
paloma@davinci:~/kubernetes$ ansible -i hosts all -m ping

master | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
worker1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
worker2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
~~~

## Creación de un usuario de Kubernetes

En primer lugar, hay que crear un nuevo usuario en cada nodo, que no será root pero tendrá privilegios de sudo. Se crea un fichero llamado users.yaml con el siguiente contenido:
~~~
- hosts: 'workers, masters'
  become: yes

  tasks:
    - name: create the kube user account
      user: name=kube append=yes state=present createhome=yes shell=/bin/bash

    - name: allow 'kube' to use sudo without needing a password
      lineinfile:
        dest: /etc/sudoers
        line: 'kube ALL=(ALL) NOPASSWD: ALL'
        validate: 'visudo -cf %s'

    - name: set up authorized keys for the kube user
      authorized_key: user=kube key="{{item}}"
      with_file:
        - ~/.ssh/id_rsa.pub
~~~

Y ejecutamos el playbook:
~~~
ansible-playbook -i hosts users.yaml
~~~

## Implementar Kubernetes

Con el siguiente playbook al que hemos llamado k8s-install.yaml se prepara a los nodos, añade los repositorios necesarios e instala containerd, kubelet, kubeadm y kubectl, entre otras cosas:
~~~
---
- hosts: "masters, workers"
  remote_user: paloma
  become: yes
  become_method: sudo
  become_user: root
  gather_facts: yes
  connection: ssh

  tasks:
     - name: Create containerd config file
       file:
         path: "/etc/modules-load.d/containerd.conf"
         state: "touch"

     - name: Add conf for containerd
       blockinfile:
         path: "/etc/modules-load.d/containerd.conf"
         block: |
               overlay
               br_netfilter

     - name: modprobe
       shell: |
               sudo modprobe overlay
               sudo modprobe br_netfilter


     - name: Set system configurations for Kubernetes networking
       file:
         path: "/etc/sysctl.d/99-kubernetes-cri.conf"
         state: "touch"

     - name: Add conf for containerd
       blockinfile:
         path: "/etc/sysctl.d/99-kubernetes-cri.conf"
         block: |
                net.bridge.bridge-nf-call-iptables = 1
                net.ipv4.ip_forward = 1
                net.bridge.bridge-nf-call-ip6tables = 1

     - name: Apply new settings
       command: sudo sysctl --system

     - name: install containerd
       shell: |
               sudo apt update && sudo apt install -y containerd
               sudo mkdir -p /etc/containerd
               sudo containerd config default | sudo tee /etc/containerd/config.toml
               sudo systemctl restart containerd

     - name: disable swap
       shell: |
               sudo swapoff -a
               sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

     - name: install and configure dependencies
       shell: |
               sudo apt update && sudo apt install -y gnupg apt-transport-https curl
               curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

     - name: Create kubernetes repo file
       file:
         path: "/etc/apt/sources.list.d/kubernetes.list"
         state: "touch"

     - name: Add K8s Source
       blockinfile:
         path: "/etc/apt/sources.list.d/kubernetes.list"
         block: |
               deb https://apt.kubernetes.io/ kubernetes-xenial main

     - name: install kubernetes
       shell: |
               sudo apt update
               sudo apt install -y kubelet=1.20.1-00 kubeadm=1.20.1-00 kubectl=1.20.1-00
               sudo apt-mark hold kubelet kubeadm kubectl
~~~

Se realizan las acciones de este playbook:
~~~
ansible-playbook -i hosts k8s-install.yaml
~~~

## Nodo maestro

Con el fichero master.yaml se va a configurar el nodo maestro. Entre otras cosas, se genera el comando de unión entre el master y los trabajadores que se guardará en un fichero local:
~~~
- hosts: masters
  become: yes
  tasks:
    - name: initialize the cluster
      shell: kubeadm init --pod-network-cidr=10.244.0.0/16
      args:
        chdir: $HOME
        creates: cluster_initialized.txt

    - name: create .kube directory
      become: yes
      become_user: kube
      file:
        path: $HOME/.kube
        state: directory
        mode: 0755

    - name: copies admin.conf to user's kube config
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/kube/.kube/config
        remote_src: yes
        owner: kube

    - name: install Pod network
      become: yes
      become_user: kube
      shell: kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml 
      args:
        chdir: $HOME
        
    - name: Get the token for joining the worker nodes
      become: yes
      become_user: kube
      shell: kubeadm token create  --print-join-command
      register: kubernetes_join_command

    - name: Copy join command to local file.
      become: yes
      local_action: copy content="{{ kubernetes_join_command.stdout_lines[0] }}" dest="./kubernetes_join_command" mode=0777
~~~

Se ejecuta el playbook:
~~~
ansible-playbook -i hosts master.yaml
~~~

Se comprueba el resultado listando los nodos del nuevo cluster:
~~~
kube@davinci:/home/paloma/kubernetes$ kubectl get nodes
NAME      STATUS   ROLES                  AGE    VERSION
davinci   Ready    control-plane,master   5m8s   v1.20.1
~~~

## Unir los minios

Por último, se utiliza el fichero creado anteriormente para añadir los minios al cluster con el fichero minios.yaml:
~~~
- hosts: workers
  become: yes
  gather_facts: yes

  tasks:
   - name: Copy join command from Ansiblehost to the worker nodes.
     become: yes
     copy:
       src: ./kubernetes_join_command
       dest: ./kubernetes_join_command
       mode: 0777

   - name: Join the Worker nodes to the cluster.
     become: yes
     command: sh ./kubernetes_join_command
     register: joined_or_not
~~~

Y se implementa:
~~~
ansible-playbook -i hosts minios.yaml
~~~

Para finalizar, se comprueba que ya aparecen los nodos:
~~~
kube@davinci:/home/paloma/kubernetes$ kubectl get nodes
NAME         STATUS   ROLES                  AGE    VERSION
buonarroti   Ready    <none>                 116s   v1.20.1
davinci      Ready    control-plane,master   13m    v1.20.1
sanzio       Ready    <none>                 116s   v1.20.1
~~~

Y ya tenemos el cluster listo para ser usado.
