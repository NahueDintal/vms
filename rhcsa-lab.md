# Lab RHCSA (EX200) en Arch Linux + libvirt/KVM

Guía completa para montar un laboratorio de práctica con máquinas Rocky Linux 9.

---

## 1. Preparación del host (Arch)

```bash
sudo pacman -S libvirt dnsmasq virt-install qemu-full edk2-ovmf
sudo usermod -aG libvirt $USER
sudo systemctl enable --now libvirtd
sudo virsh net-start default   # si no está activa
```

Directorio sugerido para discos e ISOs: `/var/lib/libvirt/images/` (viene por defecto con el pool `default`).

Descargá el ISO de Rocky 9 Minimal:
https://rockylinux.org/download

```bash
sudo mv Rocky-9.*-x86_64-minimal.iso /var/lib/libvirt/images/
```

## 2. Arquitectura del lab

| VM | Rol | vCPU | RAM | Discos | Redes |
|----|-----|------|-----|--------|-------|
| server.example.com | Sistema principal | 2 | 2G | 20G OS + 5G + 5G (extra) | default (NAT/internet) + labnet x2 |
| node2.example.com | Segunda máquina (NFS/SSH/ldap n/a) | 1 | 1G | 10G | default + labnet |

- **default**: red NAT con internet (para dnf, podman pull, etc.)
- **labnet**: red interna aislada 192.168.100.0/24 para prácticas entre VMs, teaming, firewall, etc.
- Los **dos discos extra** en server son para LVM, Stratis, VDO y swap.

## 3. Crear la red interna labnet

Archivo `/tmp/labnet.xml`:

```xml
<network>
  <name>labnet</name>
  <bridge name='virbr1'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.10' end='192.168.100.50'/>
    </dhcp>
  </ip>
</network>
```

```bash
sudo virsh net-define /tmp/labnet.xml
sudo virsh net-start labnet
sudo virsh net-autostart labnet
```

## 4. Crear las VMs

```bash
cd /var/lib/libvirt/images/

# Discos extra del server
sudo qemu-img create -f qcow2 server-lvm1.qcow2 5G
sudo qemu-img create -f qcow2 server-lvm2.qcow2 5G

# Server
sudo virt-install \
  --name server \
  --ram 2048 --vcpus 2 \
  --disk size=20,bus=virtio \
  --disk path=/var/lib/libvirt/images/server-lvm1.qcow2,bus=virtio \
  --disk path=/var/lib/libvirt/images/server-lvm2.qcow2,bus=virtio \
  --network network=default,model=virtio \
  --network network=labnet,model=virtio \
  --network network=labnet,model=virtio \
  --location /var/lib/libvirt/images/Rocky-9.*-x86_64-minimal.iso \
  --extra-args "inst.ks=https://..." \
  --os-variant rocky9.0 \
  --graphics none \
  --console pty,target_type=serial

# Node2
sudo virt-install \
  --name node2 \
  --ram 1024 --vcpus 1 \
  --disk size=10,bus=virtio \
  --network network=default,model=virtio \
  --network network=labnet,model=virtio \
  --location /var/lib/libvirt/images/Rocky-9.*-x86_64-minimal.iso \
  --os-variant rocky9.0 \
  --graphics none \
  --console pty,target_type=serial
```

> Sin `--extra-args` el instalador anaconda queda en modo gráfico. Con `--graphics none` usá la consola serial (VGA serial console). Alternativa sencilla: crear la VM con virt-manager la primera vez e instalar con GUI, y después usar `virsh console` + SSH.

### Truco: instalación automatizada con Kickstart

Creá un archivo `ks.cfg` y servilo por HTTP o incrustalo en el initrd:

```bash
sudo virt-install ... --extra-args "inst.ks=file:///ks.cfg" --initrd-inject /ruta/ks.cfg
```

Kickstart mínimo (particionado LVM, usuario, red DHCP):

```text
lang en_US.UTF-8
keyboard us
timezone America/Argentina/Buenos_Aires --utc
rootpw --plaintext redhat
user --name=student --password=redhat --plaintext --groups=wheel
bootloader --location=mbr
clearpart --all --initlabel
autopart --type=lvm
network --bootproto=dhcp --device=eth0 --activate
firewall --enabled --service=ssh
selinux --enforcing
reboot
%packages
@minimal-environment
bash-completion
vim
tmux
%end
```

## 5. Snapshots por tema (flujo de práctica)

Cloná la VM base o usá snapshots para poder volver atrás:

```bash
sudo virsh snapshot-create-as server limpio --description "Base instalada"
sudo virsh snapshot-list server
sudo virsh snapshot-revert server limpio
```

Para iterar rápido, antes de cada bloque de ejercicios:

```bash
sudo virsh snapshot-create-as server tema-lvm
# ... practicás ...
sudo virsh snapshot-revert server tema-lvm --running
```

## 6. Ejercicios por objetivo del EX200 v9

### 6.1 Operar sistemas en ejecución
- Cambiar el target por defecto: `systemctl set-default multi-user.target`
- Bootear a rescue/emergency: editar la línea del kernel con `e` en GRUB (`systemd.unit=rescue.target`)
- Recuperar root con `rd.break`: `mount -o remount,rw /sysroot; chroot /sysroot; passwd; touch /.autorelabel`
- Perfiles de tuning: `tuned-adm list`, `tuned-adm profile latency-performance`, `tuned-adm active`
- Procesos: `ps`, `top/htop`, `kill`, `nice/renice`, `systemctl kill`
- Logs: `journalctl -u sshd --since "1 hour ago"`, `journalctl -b -p err`

### 6.2 Usuarios y grupos
- Crear usuario con caducidad: `useradd -e 2027-01-01 -G wheel alumno`
- `chage`, `passwd -l/-u`, `usermod -aG`
- `visudo`: dar a `%wheel` NOPASSWD o a un usuario comandos específicos
- ACLs: `setfacl -m u:alumno:rwx /proyecto`, `getfacl`, máscara por defecto `setfacl -m d:u:alumno:rwx`
- Umask persistente: `/etc/profile.d/` o `~/.bashrc` según pida el enunciado

### 6.3 Scripts de shell
- Script con condicionales, bucles y código de salida:

```bash
#!/bin/bash
# /usr/local/bin/mkusers.sh: crea usuarios de un archivo, uno por línea
while read -r user; do
  [ -z "$user" ] && continue
  id "$user" &>/dev/null && { echo "existe: $user"; continue; }
  useradd "$user" && echo "creado: $user"
done < "$1"
exit 0
```

- `chmod +x`, ejecutar, verificar `echo $?`
- Programar: `crontab -e` (`*/10 * * * * /usr/local/bin/mkusers.sh /opt/usuarios.txt`) y timer systemd equivalente

### 6.4 Almacenamiento local (el tema más pesado)
- Particionar los discos extra: `parted /dev/vdb mklabel gpt`, `parted /dev/vdb mkpart primary 1MiB 100%`
- LVM:
  ```bash
  pvcreate /dev/vdb1 /dev/vdc1
  vgcreate datos /dev/vdb1 /dev/vdc1
  lvcreate -n ventas -L 2G datos
  mkfs.xfs /dev/datos/ventas
  mkdir /ventas && mount /dev/datos/ventas /ventas
  # extender:
  lvextend -L +1G /dev/datos/ventas && xfs_growfs /ventas
  ```
- Swap: `mkswap /dev/vdb2`, `swapon`, persistir en `/etc/fstab` con UUID
- Stratis: `dnf install stratisd stratis-cli`, `systemctl enable --now stratisd`, `stratis pool create p1 /dev/vdb`, `stratis fs create p1 fs1`, montar `/dev/stratis/p1/fs1`
- VDO: `dnf install vdo kmod-kvdo`, `vdo create --name=vdo1 --device=/dev/vdc --vdoLogicalSize=50G`, `mkfs.xfs -K /dev/mapper/vdo1`
- fstab: montar por **UUID** (`blkid`), validar con `mount -a` antes de reiniciar (¡nunca reinicies sin este paso!)
- NFS + autofs:
  ```bash
  # en node2 (servidor):
  dnf install nfs-utils && systemctl enable --now nfs-server
  echo '/compartido 192.168.100.0/24(rw,sync,no_subtree_check)' >> /etc/exports
  exportfs -rav
  # en server (cliente):
  dnf install autofs nfs-utils
  echo '/misc  /etc/auto.misc' >> /etc/auto.master.d/misc.autofs
  echo 'compartido -rw,soft node2:/compartido' >> /etc/auto.misc
  systemctl enable --now autofs
  ls /misc/compartido   # aparece al acceder
  ```

### 6.5 Redes
- nmcli es EL comando del examen:
  ```bash
  nmcli con show
  nmcli con mod "Wired connection 1" ipv4.method manual ipv4.addresses 192.168.100.50/24 ipv4.gateway 192.168.100.1 ipv4.dns 8.8.8.8
  nmcli con up "Wired connection 1"
  nmcli con add type team con-name team0 ifname team0 config '{"runner":{"name":"activebackup"}}'
  nmcli con add type team-slave con-name team0-p1 ifname eth1 master team0
  nmcli con add type team-slave con-name team0-p2 ifname eth2 master team0
  nmcli con up team0
  teamdctl team0 state
  ```
- hostname: `hostnamectl set-hostname server.example.com`
- Resolver: nmcli ipv4.dns + dominio de búsqueda (`ipv4.dns-search example.com`)
- Firewall:
  ```bash
  firewall-cmd --permanent --new-zone=interna
  firewall-cmd --permanent --zone=interna --add-source=192.168.100.0/24
  firewall-cmd --permanent --zone=public --add-service=http
  firewall-cmd --permanent --add-port=8080/tcp
  firewall-cmd --reload
  ```

### 6.6 SELinux
- `getenforce`, `setenforce 0/1`, `sestatus`
- Contextos: `ls -Z`, `ps -Z`, `chcon -t httpd_sys_content_t /var/web`, `restorecon -Rv /var/web`
- Persistente: semanage fcontext + restorecon:
  ```bash
  dnf install policycoreutils-python-utils
  semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"
  restorecon -Rv /web
  ```
- Booleanos: `getsebool -a | grep httpd`, `setsebool -P httpd_enable_homedirs on`
- Puertos: `semanage port -a -t http_port_t -p tcp 8081`

### 6.7 Contenedores (Podman) — gran protagonista en RHEL 9
- Registro: configurar `/etc/containers/registries.conf` (o `registries.conf.d/`) para registry corto
- Básico: `podman search`, `podman pull`, `podman images`, `podman run -d --name web -p 8080:80 -v /srv/web:/usr/share/nginx/html:Z nginx`
- `podman exec -it web bash`, `podman logs -f web`, `podman ps -a`
- Volumen persistente y rootless: practicar también como usuario normal (`loginctl enable-linger student`)
- Arranque automático (quadlet, RHEL 9.4+):
  ```bash
  mkdir -p ~/.config/containers/systemd/
  cat > ~/.config/containers/systemd/web.container <<'EOF'
  [Container]
  Image=registry.access.redhat.com/ubi9/httpd-24
  PublishPort=8080:8080
  Volume=/srv/web:/var/www/html:Z
  [Install]
  WantedBy=default.target
  EOF
  systemctl --user daemon-reload && systemctl --user start web
  ```
- Alternativa clásica: `podman generate systemd --new --name web > ~/.config/systemd/user/container-web.service`

### 6.6 Gestión de software
- Repos: `/etc/yum.repos.d/`, `dnf config-manager --add-repo`, `--set-enabled`
- `dnf install/remove/groups install "Development Tools"`, `dnf history undo N`, `dnf update`
- `rpm -qa`, `rpm -qf /bin/ls`, `rpm -qi`

## 7. Simulacros de examen

- Duración: **4 horas**, ~17 tareas, se aprueba con **70%**.
- **Sin internet ni documentación**: todo de memoria.
- Estrategia:
  1. Snapshot `limpio` al terminar la instalación.
  2. Hacé sets de tareas cronometrados (ej. 4 tareas en 60 min).
  3. Revertí y repetí hasta salir fluido.
  4. Revisá al final con `sosreport`? no — con una checklist propia por tarea (servicio activo, enabled, firewall, SELinux, persiste reboot: `systemctl is-enabled X`).
- La regla de oro del EX200: **todo lo que configures tiene que sobrevivir al reboot** (servicios enabled, fstab correcto, firewall permanent, SELinux etiquetado).

## 8. Checklist post-lab

- [ ] server y node2 se ven entre sí por labnet (ping por IP y hostname con dns-search)
- [ ] SSH con claves de student -> root deshabilitado por contraseña
- [ ] chrony sincronizado: `chronyc sources`
- [ ] snapshot base guardado
- [ ] tmux + bash-completion instalados en las VMs para trabajar cómodo
