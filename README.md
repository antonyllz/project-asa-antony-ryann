# Projeto de Infraestrutura como Código (IaC)

## Objetivo do Projeto
Desenvolver competências práticas em DevOps e Infraestrutura como Código (IaC) utilizando as ferramentas Vagrant e Ansible. O objetivo é provisionar uma infraestrutura virtual e automatizar a configuração do sistema operacional e serviços essenciais.

## Integrantes da Equipe
- Ryann
- Antony

## Professor
- Pedro Batista de Carvalho Filho

## Disciplina
- Administração de Sistemas Abertos

---

## Descrição do Projeto

Este projeto visa automatizar a configuração de uma infraestrutura virtual utilizando o Vagrant para provisionamento de máquinas virtuais e o Ansible para a configuração do sistema operacional e serviços. O processo de desenvolvimento envolveu a criação de um **Vagrantfile** e **playbook** do Ansible para atender aos requisitos de configuração.

---

## Vagrantfile

O `Vagrantfile` é responsável por definir a configuração da máquina virtual e como ela será provisionada. Ele especifica a box a ser utilizada, a rede, os recursos alocados (memória, discos) e a execução do playbook do Ansible após a criação da máquina.

### Descrição das seções do Vagrantfile

1. **Box e Provider**:
   Definimos a box `generic/debian12`, uma imagem minimalista do Debian 12, e o provider `virtualbox`, que é o VirtualBox. Isso indica que as máquinas virtuais serão executadas no VirtualBox.

   ```ruby
   config.vm.box = "generic/debian12"
Configuração do Provider VirtualBox: Aqui definimos as configurações do provider. Especificamos o nome da VM (p01_Ryann_Antony) e a memória alocada (1024 MB).


config.vm.provider "virtualbox" do |vb|
  vb.name = "p01_Ryann_Antony"
  vb.memory = 1024
end
Discos adicionais: Três discos virtuais de 15 GB são adicionados à máquina virtual, para simular um ambiente com mais de um disco. Esses discos serão utilizados para a configuração de LVM mais adiante no playbook.



config.vm.disk :disk, size: "15GB", name: "disk1"
config.vm.disk :disk, size: "15GB", name: "disk2"
config.vm.disk :disk, size: "15GB", name: "disk3"
Configuração de Rede: Definimos uma rede privada com o IP 192.168.57.10, que permite que a máquina virtual se conecte a outras máquinas na rede interna.



config.vm.network "private_network", ip: "192.168.57.10"
Provisionamento com Ansible: O playbook do Ansible será executado automaticamente após o provisionamento da máquina. O arquivo do playbook é playbook.yml.



config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
end
Playbook Ansible
O playbook.yml realiza a configuração automatizada da máquina virtual. Abaixo, explico cada parte do playbook.

Descrição das tarefas no playbook
Atualização do Sistema Operacional: Esta tarefa atualiza todos os pacotes do sistema, garantindo que a máquina virtual esteja com o sistema mais recente.



- name: Realizar atualização completa do sistema operacional
  apt:
    update_cache: yes
    upgrade: dist
Configuração do Hostname: A tarefa altera o hostname da máquina virtual para p01_Ryann_Antony, garantindo que a VM tenha um nome claro e reconhecível.



- name: Configurar o hostname
  hostname:
    name: "p01_Ryann_Antony"
Criação de Usuários: Cria os usuários Antony e Ryann com o shell /bin/bash, permitindo que ambos acessem a máquina virtual.



- name: Criar usuário Antony
  user:
    name: "Antony"
    state: present
    shell: /bin/bash

- name: Criar usuário Ryann
  user:
    name: "Ryann"
    state: present
    shell: /bin/bash
Criação de Banner de Acesso: Um banner de acesso é criado no arquivo /etc/issue.net, que será exibido sempre que um usuário tentar acessar a máquina via SSH. Isso informa aos usuários que o acesso é monitorado.



- name: Criar arquivo de banner de acesso restrito
  copy:
    dest: /etc/issue.net
    content: |
      Acesso restrito apenas à pessoas com autorização expressa
      Seu acesso está sendo monitorado !!!
    owner: root
    group: root
    mode: '0644'
Configuração do SSH: Configurações de segurança do SSH são aplicadas:

Exibição do banner no SSH.
Bloqueio do login do usuário root via SSH.
Desativação de autenticação por senha, permitindo apenas autenticação via chave pública.

- name: Configurar o SSH para usar o banner
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?Banner'
    line: 'Banner /etc/issue.net'

- name: Bloquear o acesso SSH para o usuário root
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin no'

- name: Permitir apenas autenticação por chave pública no SSH
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PasswordAuthentication'
    line: 'PasswordAuthentication no'

- name: Permitir acesso apenas para usuários do grupo acesso_ssh
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?AllowGroups'
    line: 'AllowGroups acesso_ssh'
Configuração de LVM (Logical Volume Management): LVM é configurado para criar volumes lógicos (LV) usando três discos de 10GB. O volume lógico sistema é criado com 15GB e montado no diretório /dados.



- name: Criar Physical Volume (PV) nos três discos
  lvol:
    pv: "{{ item }}"
    state: present
  loop:
    - /dev/sda
    - /dev/sdb
    - /dev/sdc

- name: Criar Volume Group "dados"
  lvol:
    vg: dados
    pvs:
      - /dev/sda
      - /dev/sdb
      - /dev/sdc
    state: present

- name: Criar Logical Volume "sistema" com 15GB
  lvol:
    vg: dados
    lv: sistema
    size: 15G
    state: present

- name: Formatar o Logical Volume "sistema" no formato ext4
  filesystem:
    fstype: ext4
    dev: /dev/dados/sistema

- name: Montar o Logical Volume "sistema" no diretório /dados
  mount:
    name: /dados
    src: /dev/dados/sistema
    fstype: ext4
    state: mounted
Configuração de NFS: O diretório /dados/nfs é configurado para compartilhamento via NFS, permitindo que outros sistemas possam montar este diretório como uma rede compartilhada.



- name: Instalar pacotes necessários para o NFS
  apt:
    name: nfs-kernel-server
    state: present

- name: Configurar as permissões do diretório /dados/nfs
  file:
    path: /dados/nfs
    owner: nfs-ifpb
    group: nfs-ifpb
    mode: '0770'

- name: Configurar exportação NFS
  lineinfile:
    path: /etc/exports
    regexp: '^/dados/nfs'
    line: '/dados/nfs 192.168.57.0/24(rw,sync,no_subtree_check,all_squash,anonuid=1001,anongid=1001)'

- name: Exportar as novas configurações NFS
  command: exportfs -a
Monitoramento de Acesso: A tarefa cria um script para monitorar e registrar os acessos aos diretórios NFS. Isso é feito utilizando a configuração de exportação NFS.

Como Executar o Projeto
Instale o Vagrant e o VirtualBox no seu sistema.
Clone este repositório ou baixe os arquivos.
Execute o seguinte comando no diretório onde o Vagrantfile está localizado:
bash
'''
vagrant up

Isso provisionará a máquina virtual com as especificações do projeto, e o Ansible será automaticamente invocado para configurar o sistema.

