# Playbook para instalar frigate con ansible desde MOMS

Los bloques de código de este documento conforman un playbook de ansible
que realiza desde el MOMS las tareas de instalación del container podman
de frigate asi como sus configuraciones varias en los servidores DVR de
PRT.

Desglosamos distintas tareas en distintos bloques tratando de explicar
de que va cada bloque/tares

## Seteamos el hostname conforme al inventory.ini

``` yaml
---
- name: INSTALL FRIGATE
  hosts: all
#  become: yes
  become_method: sudo
  vars_prompt:
    - name: "ansible_become_pass"
      prompt: "Enter your sudo password"
      private: yes

  tasks:
    - name: set system hostname to {{ inventory_hostname }}
      become: yes
      command: hostnamectl set-hostname {{ inventory_hostname }}

```

## Nos aseguramos que sice está en el grupo sudo y deshabilitamos password de sudo para grupo sudo (a lo loco).

``` yaml
#---

    - name: Ensure user sice is in the sudo group
      become: yes
      become_method: sudo
      user:
        name: sice
        groups: sudo
        append: yes

    - name: Allow members of the sudo group to run sudo without a password
      become: yes
      become_method: sudo
      lineinfile:
        path: /etc/sudoers
        regexp: '^%sudo'
        line: '%sudo ALL=(ALL:ALL) NOPASSWD: ALL'
        validate: 'visudo -cf %s'

```

### Explanation:

-   **`become: yes`**: Enables privilege escalation.
-   **`tasks`**: Contains the tasks to be performed.
-   **`user` module**: Ensures the user `sice` is added to the `sudo`
    group.

## El siguiente bloque de código hará las siguientes tareas:

1.  Ensure the directory \~/instalacion/FRIGATE exists
2.  copy the local file
    "/home/concesion/instalacion/DVAS/FRIGATE/FRIGATE-INSTALL.tar" into
    the directory "/home/sice/instalacion"
3.  Untar FRIGATE-INSTALL.tar in
    "//home/concesion/instalacion/DVAS/FRIGATE/"
4.  Untar tmux.tar
5.  Prepare tmux.
6.  Load podman image
7.  Execute "dpkg -i \*" in directory
    "//home/concesion/instalacion/DVAS/FRIGATE/PODMAN-PKGS/"

``` yaml
#---

    - name: Ensure the directory ~/instalacion/DVAS/FRIGATE exists
      file:
        path: /home/sice/instalacion/DVAS/FRIGATE
        state: directory
        mode: '0755'  # Optional: Set permissions

    - name: change owner to ~/instalacion/DVAS/FRIGATE
      become: yes
      command: chown -R sice:sice /home/sice/instalacion/DVAS/FRIGATE

    - name: Copy FRIGATE-INSTALL.tar to {{ inventory_hostname }}:~/instalacion/DVAS/FRIGATE
      copy:
        src: /home/concesion/instalacion/DVAS/FRIGATE/FRIGATE-INSTALL.tar
        dest: /home/sice/instalacion

    - name: Untar FRIGATE-INSTALL.tar
      unarchive:
        src: /home/sice/instalacion/FRIGATE-INSTALL.tar
        dest: /home/sice/instalacion/DVAS/FRIGATE/
        remote_src: yes

    - name: Untar tmux.tar
      unarchive:
        src: /home/sice/instalacion/DVAS/FRIGATE/frigate-install/tmux.tar
        dest: /home/sice
        remote_src: yes

    - name: Prepare tmux
      shell: ln -sf .tmux/.tmux.conf
      args:
        chdir: /home/sice

#   - name: Load frigate_0.13.1-sice-20240628-03.bz2 podman image
#     shell: bzcat frigate_0.13.1-sice-20240628-03.bz2 | podman load
#     args:
#       chdir: /home/sice/instalacion/DVAS/FRIGATE/frigate-install # Change to the specified directory before executing the command

#   - name: Install all .deb packages in /path
#     become: yes
#     shell: dpkg -i *.deb
#     args:
#       chdir: /home/sice/instalacion/DVAS/FRIGATE/frigate-install/PODMAN-PKGS/

```

## Zona horaria y sincronización

``` yaml
#---

    - name: Set timezone to America/Puerto_Rico
      become: true
      community.general.timezone:
        name: America/Puerto_Rico

    - name: Comment out line containing 'tos minclock 4 minsane 3'
      become: true
      lineinfile:
        path: /etc/ntpsec/ntp.conf
        regexp: '^tos minclock 4 minsane 3'
        line: '# tos minclock 4 minsane 3'
        state: present

    - name: Comment out existing pool lines
      become: true
      replace:
        path: /etc/ntpsec/ntp.conf
        regexp: '^(pool 0\.debian\.pool\.ntp\.org|pool 1\.debian\.pool\.ntp\.org|pool 2\.debian\.pool\.ntp\.org|pool 3\.debian\.pool\.ntp\.org)'
        replace: '# \g<0>'


    - name: Add TPCC ip as server
      become: true
      lineinfile:
        path: /etc/ntpsec/ntp.conf
        regexp: '^# pool 3.debian.pool.ntp.org'
        line: "server {{ ansible_host.split('.')[0:3] | join('.') }}.1 iburst"
        state: present

    - name: Restart ntpsec service
      become: true
      systemd:
        name: ntpsec
        state: restarted
        enabled: true

```

### Cómo usarlo:

**nota**: MOMS ya tiene ansible instalado.

Todos los bloques anteriores de código yaml ya han sido volcados al
fichero:

**\~/instalacion/DVAS/FRIGATE/prt-frigate-ansible/install<sub>frigate</sub>.yml**

Solo es necesario conectarse a la maquina del MOMS como usuario
*concesion*, posicionarse en el directorio adecuado y llamar al comando
del siguiente bloque:

``` bash
# conectarse a MOMS (172.30.30.12) como usuario concesion y hacer lo que sigue
cd ~/instalacion/DVAS/FRIGATE/prt-frigate-ansible
ansible-playbook ansible/tasks/install_frigate.yml -i inventory.ini -l prt-zm01

```
