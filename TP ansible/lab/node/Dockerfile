# Nœud cible du lab Ansible : Ubuntu + systemd + SSH, comme une petite VM
FROM ubuntu:24.04
ENV container=docker DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
 && apt-get install -y --no-install-recommends systemd systemd-sysv openssh-server sudo python3 python3-apt ca-certificates curl iproute2 \
 && rm -rf /var/lib/apt/lists/* \
 && find /etc/systemd/system /lib/systemd/system -path '*.wants/*' \( -name '*getty*' -o -name '*udev*' -o -name 'systemd-logind*' \) -delete \
 && systemctl mask systemd-ask-password-console.path systemd-ask-password-wall.path systemd-binfmt.service \
      dev-hugepages.mount dev-mqueue.mount proc-sys-fs-binfmt_misc.mount sys-fs-fuse-connections.mount \
      sys-kernel-config.mount sys-kernel-debug.mount sys-kernel-tracing.mount \
 && systemctl enable ssh
# Utilisateur "ansible" avec sudo sans mot de passe (comme sur un serveur préparé pour Ansible)
RUN useradd -m -s /bin/bash ansible \
 && echo "ansible ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/ansible \
 && mkdir -p /home/ansible/.ssh && chmod 700 /home/ansible/.ssh
COPY keys/id_ansible.pub /home/ansible/.ssh/authorized_keys
RUN chown -R ansible:ansible /home/ansible/.ssh && chmod 600 /home/ansible/.ssh/authorized_keys
STOPSIGNAL SIGRTMIN+3
CMD ["/lib/systemd/systemd"]
