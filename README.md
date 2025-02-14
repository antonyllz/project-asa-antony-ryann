# Projeto de Infraestrutura como Código (IaC)

## Objetivo do Projeto
Desenvolver competências práticas em DevOps e Infraestrutura como Código (IaC) utilizando as ferramentas Vagrant e Ansible. O objetivo é provisionar uma infraestrutura virtual e automatizar a configuração do sistema operacional e serviços essenciais.

## Integrantes da Equipe
- Antony César Pereira de Araújo - 20231380013
- Jose Ryann Ferreira de Brito - 20231380032

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
   
Configuração do Provider VirtualBox: Especificamos o nome da VM (p01_Ryann_Antony) e a memória alocada (1024 MB).
```ruby

config.vm.provider "virtualbox" do |vb|
  vb.name = "p01_Ryann_Antony"
  vb.memory = 1024
end
````
Discos adicionais: Três discos virtuais de 15 GB são adicionados à máquina virtual, para simular um ambiente com mais de um disco.

```ruby
config.vm.disk :disk, size: "15GB", name: "disk1"
config.vm.disk :disk, size: "15GB", name: "disk2"
config.vm.disk :disk, size: "15GB", name: "disk3"
````
Configuração de Rede: Definimos uma rede privada com o IP 192.168.57.10.

   ```ruby
config.vm.network "private_network", ip: "192.168.57.10"
````
Provisionamento com Ansible: O playbook do Ansible será executado automaticamente após o provisionamento da máquina.

   ```ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
end

````
Playbook Ansible
O playbook.yml realiza a configuração automatizada da máquina virtual. Abaixo estão as tarefas principais.

Descrição das tarefas no playbook

**Atualização do Sistema Operacional: Atualiza todos os pacotes do sistema.**
````
- name: Realizar atualização completa do sistema operacional
  apt:
    update_cache: yes
    upgrade: dist

````
**Configuração do Hostname**
```yml
- name: Configurar o hostname
  hostname:
    name: "p01_Ryann_Antony"
````
**Criação dos usuários**
   ```yaml
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
````
**Criação de Banner de Acesso: Cria um banner para exibição em acessos via SSH.**

   ```yaml
- name: Criar arquivo de banner de acesso restrito
  copy:
    dest: /etc/issue.net
    content: |
      Acesso restrito apenas à pessoas com autorização expressa
      Seu acesso está sendo monitorado !!!
    owner: root
    group: root
    mode: '0644'
````
**Configuração do SSH: Ajusta as configurações de segurança do SSH.**

   ```yaml

    - name: Permitir autenticação apenas por chave pública
      ansible.builtin.replace:
        path: /etc/ssh/sshd_config
        regexp: "^#?PasswordAuthentication.*"
        replace: "PasswordAuthentication no"
      notify: Reiniciar SSH

    - name: Bloquear acesso root via SSH
      ansible.builtin.replace:
        path: /etc/ssh/sshd_config
        regexp: "^#?PermitRootLogin.*"
        replace: "PermitRootLogin no"
      notify: Reiniciar SSH

    - name: Permitir acesso ao grupo acesso_ssh
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?AllowGroups.*"
        line: "AllowGroups acesso_ssh"
        state: present
      notify: Reiniciar SSH


`````
Configuração de LVM: Configura Logical Volume Management.

   ```yaml
 - name: Instalar o LVM2
      ansible.builtin.apt:
        name: lvm2
        state: present
        update_cache: true

    - name: Criar Volume Group
      community.general.lvg:
        vg: dados
        pvs:
          - /dev/sdb
          - /dev/sdc
          - /dev/sdd

    - name: Criar Logical Volume
      community.general.lvol:
        vg: dados
        lv: sistema
        size: 15g

    - name: Formatar o Logical Volume
      community.general.filesystem:
        fstype: ext4
        dev: /dev/dados/sistema

    - name: Criar diretório /dados
      ansible.builtin.file:
        path: /dados
        state: directory
        mode: "0755"

    - name: Adicionar entrada ao fstab
      ansible.builtin.lineinfile:
        path: /etc/fstab
        line: "/dev/dados/sistema /dados ext4 defaults 0 0"
        state: present

    - name: Montar partição /dados
      ansible.posix.mount:
        path: /dados
        src: /dev/dados/sistema
        fstype: ext4
        state: mounted

````
**Configuração de NFS: Configura o compartilhamento via NFS.**
```yaml
   - name: Instalar pacotes necessários para NFS
      ansible.builtin.apt:
        name: nfs-kernel-server
        state: present

    - name: Criar o usuário "nfs-ifpb" com shell desabilitado
      ansible.builtin.user:
        name: nfs-ifpb
        shell: /usr/sbin/nologin
        state: present

    - name: Obter informações do usuário "nfs-ifpb"
      ansible.builtin.getent:
        database: passwd
        key: nfs-ifpb
      register: nfs_user

    - name: Configurar exportação NFS para /dados/nfs
      ansible.builtin.lineinfile:
        path: /etc/exports
        line: /dados/nfs 192.168.57.0/24(rw,sync,no_subtree_check,all_squash,anonuid={{ nfs_user.ansible_facts.getent_passwd['nfs-ifpb'][1] | int }},anongid={{ nfs_user.ansible_facts.getent_passwd['nfs-ifpb'][2] | int }})
        state: present
      notify: Reiniciar NFS

    - name: Garantir que o serviço NFS está habilitado na inicialização
      ansible.builtin.service:
        name: nfs-kernel-server
        enabled: true
      notify: Reiniciar NFS

````
**Monitoramento de Acesso**

```yaml
    - name: Criar o diretório /dados/nfs
      ansible.builtin.file:
        path: /dados/nfs
        state: directory
        owner: nfs-ifpb
        group: nfs-ifpb
        mode: "0755"

    - name: Criar o arquivo de log de acessos
      ansible.builtin.file:
        path: /dados/nfs/acessos
        state: touch
        mode: "0666"

    - name: Criar o script de monitoramento de login
      ansible.builtin.copy:
        dest: /usr/local/bin/log_login.sh
        content: |
          #!/bin/bash
          # Script para monitorar logins
          LOG_FILE="/dados/nfs/acessos"
          TIMESTAMP=$(date '+%Y-%m-%d %H:%M')
          USER=$USER
          TTY=$SSH_TTY
          IP=$(who -m | awk '{print $NF}' | tr -d '()')

        # Escreve no arquivo de log
          echo "${TIMESTAMP}; ${USER}; ${TTY}; ${IP}" >> "$LOG_FILE"
        mode: "0755"

    - name: Configurar para executar o script no login
      ansible.builtin.lineinfile:
        path: /etc/profile
        line: "/usr/local/bin/log_login.sh"
        state: present

     handlers:
    # Handlers de SSH
    - name: Reiniciar SSH
      ansible.builtin.service:
        name: sshd
        state: restarted

    # Handlers de NFS
    - name: Reiniciar NFS
      ansible.builtin.service:
        name: nfs-kernel-server
        state: restarted''

**Como Executar o Projeto**

Instale o Vagrant e o VirtualBox no seu sistema.

Clone este repositório ou baixe os arquivos.

Execute o seguinte comando no diretório onde o Vagrantfile está localizado:

Instale os requerimentos:
```ruby
    ansible-galaxy install -r requirements.yml
````
````
Instalar plugin para manter as VirtualBox Guest Additions atualizadas automaticamente no sistema convidado (guest) ao usar o Vagrant
```ruby
vagrant plugin install vagrant-vbguest
```
bash
   ```ruby
vagrant up
````
Isso provisionará a máquina virtual com as especificações do projeto, e o Ansible será automaticamente invocado para configurar o sistema.

**Para o acessor SSH às máquinas segue as instruções para cada usuário**
```ruby
ssh-keygen -t ed25519 -C "{Nome do usuário}" -f ~/.ssh/{nome_do_usuário}
````
Após isso é somente executar o comando SSH usando a chave criada e a porta associada
```ruby
ssh -i chaves/{nome_do_arquivo} {usuário}@127.0.0.1 -p 2222

````
Após isso é necessário que a máquina conheça o usuário
```ruby
ssh-keygen -f "/home/{usuário_local}/.ssh/known_hosts" -R "[127.0.0.1]:2222"

```
Para ambos os usuário as chaves SSH para conexão foram inseridas com os valores: **123**
