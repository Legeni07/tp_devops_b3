###### ERROR #####"
minikube start
😄  minikube v1.34.0 on Ubuntu 24.04 (amd64)
👎  Unable to pick a default driver. Here is what was considered, in preference order:
    ▪ docker: Not healthy: "docker version --format {{.Server.Os}}-{{.Server.Version}}:{{.Server.Platform.Name}}" exit status 1: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.47/version": dial unix /var/run/docker.sock: connect: permission denied
    ▪ docker: Suggestion: Add your user to the 'docker' group: 'sudo usermod -aG docker $USER && newgrp docker' <https://docs.docker.com/engine/install/linux-postinstall/>
    ▪ kvm2: Not healthy: /usr/bin/virsh domcapabilities --virttype kvm failed:
error: failed to connect to the hypervisor
error: Failed to connect socket to '/var/run/libvirt/libvirt-sock': Permission denied
exit status 1
    ▪ kvm2: Suggestion: Follow your Linux distribution instructions for configuring KVM <https://minikube.sigs.k8s.io/docs/reference/drivers/kvm2/>
💡  Alternatively you could install one of these drivers:
    ▪ podman: Not installed: exec: "podman": executable file not found in $PATH
    ▪ qemu2: Not installed: stat /usr/share/OVMF/OVMF_CODE.fd: no such file or directory
    ▪ virtualbox: Not installed: unable to find VBoxManage in $PATH

❌  Exiting due to DRV_NOT_HEALTHY: Found driver(s) but none were healthy. See above for suggestions how to fix installed drivers.

##### SOLUTION #####
sudo usermod -aG docker $USER
newgrp docker

sudo usermod -aG libvirt $USER
newgrp libvirt

#####  #################
Cette erreur indique que votre cluster Kubernetes n'est pas correctement initialisé ou configuré. Voici quelques étapes de diagnostic et de résolution :

Vérifier le statut de Minikube

minikube status

Redémarrer Minikube

minikube stop
minikube start --driver=docker

Vérifier la configuration kubectl

kubectl config view

Réinitialiser la configuration Kubernetes

minikube delete
minikube start --driver=docker

Vérifier les pods système

kubectl get pods -A

Installer le plugin de réseau (CNI)

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

Vérifier les logs de Minikube

minikube logs

Réinitialisation complète

sudo systemctl stop kubelet
sudo systemctl stop docker
sudo rm -rf /var/lib/cni /etc/cni /var/lib/kubelet /*
sudo systemctl start docker
sudo systemctl start kubelet
minikube start --driver=docker