
## Software richiesti
È necessario installare:
- Vagrant
- WMWARE o VIRTUALBOX: 8 Gig + RAM workstation as the Vms use 3 vCPUS and 4+ GB RAM

## Per gli utenti MAC/Linux che usano l'ultima versione di VirtualBox
Se si utilizza l'ultima versione di VirtualBox, per gli utenti Mac/Linux è necessario creare o aggiungere il file "etc/vbox/networks.conf" e aggiungere: "<pre>* 0.0.0.0/0 ::/0</pre>".
È possibile anche eseguire il seguente comando:
```shell
sudo mkdir -p /etc/vbox/
echo "* 0.0.0.0/0 ::/0" | sudo tee -a /etc/vbox/networks.conf
```

## Comandi importanti:
### Avvio
- Avvio con Virtualbox:
  ```bash
  mv VagrantfileVB Vagrantfile"
  vagrant up
  ```
- Avvio con VMWare:
  ```bash
  mv VagrantfileVMWare Vagrantfile
  vagrant up --provider wmware_desktop
  ```
  
### Spegnimento 
  ```bash
  vagrant hant
  ```
### Riavvio 
  ```bash
  vagrant up
  ```
### Eliminare il cluster
  ```bash
  vagrant destroy -f
  ```

## Set Kubeconfig file variable

```shell
cd vagrant-kubeadm-kubernetes
cd configs
export KUBECONFIG=$(pwd)/config
```

or you can copy the config file to .kube directory.

```shell
cp config ~/.kube/
```

## Install Kubernetes Dashboard

The dashboard is automatically installed by default, but it can be skipped by commenting out the dashboard version in _settings.yaml_ before running `vagrant up`.

If you skip the dashboard installation, you can deploy it later by enabling it in _settings.yaml_ and running the following:
```shell
vagrant ssh -c "/vagrant/scripts/dashboard.sh" controlplane
```

## Kubernetes Dashboard Access

To get the login token, copy it from _config/token_ or run the following command:
```shell
kubectl -n kubernetes-dashboard get secret/admin-user -o go-template="{{.data.token | base64decode}}"
```

Make the dashboard accessible:
```shell
kubectl proxy
```

Open the site in your browser:
```shell
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
```
# Network graph

```
                  +-------------------+
                  |    External       |
                  |  Network/Internet |
                  +-------------------+
                           |
                           |
             +-------------+--------------+
             |        Host Machine        |
             |     (Internet Connection)  |
             +-------------+--------------+
                           |
                           | NAT
             +-------------+--------------+
             |    K8s-NATNetwork          |
             |    192.168.99.0/24         |
             +-------------+--------------+
                           |
                           |
             +-------------+--------------+
             |     k8s-Switch (Internal)  |
             |       192.168.99.1/24      |
             +-------------+--------------+
                  |        |        |
                  |        |        |
          +-------+--+ +---+----+ +-+-------+
          |  Master  | | Worker | | Worker  |
          |   Node   | | Node 1 | | Node 2  |
          |192.168.99| |192.168.| |192.168. |
          |   .99    | | 99.81  | | 99.82   |
          +----------+ +--------+ +---------+
```

This network graph shows:

1. The host machine connected to the external network/internet.
2. The NAT network (K8s-NATNetwork) providing a bridge between the internal network and the external network.
3. The internal Hyper-V switch (k8s-Switch) connecting all the Kubernetes nodes.
4. The master node and two worker nodes, each with their specific IP addresses, all connected to the internal switch.

