# importa il file settings.yaml
require "yaml"

# Ottengo il percorso della cartella in cui si trova il file Vagrantfile
vagrant_dir_root = File.dirname(File.expand_path(__FILE__))
settings = YAML.load_file "#{vagrant_dir_root}/settings.yaml"

# Estrae l'indirizzo IP del control plane
IP_SECTIONS = settings["network"]["control_ip"].match(/^([0-9.]+\.)([^.]+)$/)

puts "IP_SECTIONS: #{IP_SECTIONS}"
# Primo tre ottetti dell'indirizzo IP (escluso l'ultimo ottetto)
IP_NW = IP_SECTIONS.captures[0]

# Ottetto dell'indirizzo IP (ultimo ottetto)
IP_START = Integer(IP_SECTIONS.captures[1])

# Numero di nodi worker
NUM_WORKER_NODES = settings["nodes"]["workers"]["count"]

Vagrant.configure("2") do |config|
  # Imposta le variabili d'ambiente per l'indirizzo IP del control plane e dei nodi worker nel file /etc/hosts di ogni macchina virtuale
  config.vm.provision "shell", env: { "IP_NW" => IP_NW, "IP_START" => IP_START, "NUM_WORKER_NODES" => NUM_WORKER_NODES }, inline: <<-SHELL
      apt-get update -y
      echo "$IP_NW$((IP_START)) controlplane" >> /etc/hosts
      for i in `seq 1 ${NUM_WORKER_NODES}`; do
        echo "$IP_NW$((IP_START+i)) node0${i}" >> /etc/hosts
      done
  SHELL

  # se l'architettura è ARM64, utilizza la box ARM64 altrimenti utilizza la box standard
  if `uname -m`.strip == "aarch64"
    config.vm.box = settings["software"]["box"] + "-arm64"
  else
    config.vm.box = settings["software"]["box"]
  end
  # Aggiorna la box
  config.vm.box_check_update = true

  # Definizione del control plane (nodo master)
  config.vm.define "controlplane" do |controlplane|
    # Imposto il nome della macchina virtuale
    controlplane.vm.hostname = "controlplane"
    #imposto l'indirizzo IP della macchina virtuale
    controlplane.vm.network "private_network", ip: settings["network"]["control_ip"]

    # Condivido le cartelle tra host e macchina virtuale
    if settings["shared_folders"]
      settings["shared_folders"].each do |shared_folder|
        controlplane.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
      end
    end

    # Imposto le risorse della macchina virtuale (CPU e memoria)
    controlplane.vm.provider "vmware_desktop" do |vb|
        vb.vmx["virtualHW.version"] = "20"  # Imposta la versione hardware
        vb.vmx["numvcpus"] = settings["nodes"]["control"]["cpu"]
        vb.vmx["memsize"] = settings["nodes"]["control"]["memory"]
        #Abilita la GUI
        vb.gui = true
        # Imposta il gruppo della macchina virtuale (se definito)
        if settings["cluster_name"] && settings["cluster_name"] != ""
          # Crea una variabile d'ambiente per il nome del cluster
          vb.vmx["group"] = settings["cluster_name"]
        end
    end
    # Provisioning della macchina virtuale con gli script common.sh e master.sh
    controlplane.vm.provision "shell",
      env: {
        # Imposto le variabili d'ambiente per lo script common.sh
        "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
        "ENVIRONMENT" => settings["environment"],
        "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
        "KUBERNETES_VERSION_SHORT" => settings["software"]["kubernetes"][0..3],
        "OS" => settings["software"]["os"]
      },
      path: "scripts/common.sh"
    controlplane.vm.provision "shell",
      env: {
        # Imposto le variabili d'ambiente per lo script master.sh
        "CALICO_VERSION" => settings["software"]["calico"],
        "CONTROL_IP" => settings["network"]["control_ip"],
        "POD_CIDR" => settings["network"]["pod_cidr"],
        "SERVICE_CIDR" => settings["network"]["service_cidr"]
      },
      path: "scripts/master.sh"
  end

  (1..NUM_WORKER_NODES).each do |i|

    config.vm.define "node0#{i}" do |node|
      node.vm.hostname = "node0#{i}"
      node.vm.network "private_network", ip: IP_NW + "#{IP_START + i}"
      if settings["shared_folders"]
        settings["shared_folders"].each do |shared_folder|
          node.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
        end
      end
      node.vm.provider "vmware_desktop" do |vb|
          vb.vmx["virtualHW.version"] = "20"  # Imposta la versione hardware
          vb.vmx["numvcpus"] = settings["nodes"]["workers"]["cpu"]
          vb.vmx["memsize"] = settings["nodes"]["workers"]["memory"]
          vb.gui = true # Abilita la GUI anche per i worker
          if settings["cluster_name"] && settings["cluster_name"] != ""
            # Crea una variabile d'ambiente per il nome del cluster
            vb.vmx["group"] = settings["cluster_name"]
          end
      end
      node.vm.provision "shell",
        env: {
          "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
          "ENVIRONMENT" => settings["environment"],
          "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
          "KUBERNETES_VERSION_SHORT" => settings["software"]["kubernetes"][0..3],
          "OS" => settings["software"]["os"]
        },
        path: "scripts/common.sh"
      node.vm.provision "shell", path: "scripts/node.sh"

      # Install the dashboard after provisioning the last worker (if enabled).
      if i == NUM_WORKER_NODES and settings["software"]["dashboard"] and settings["software"]["dashboard"] != ""
        node.vm.provision "shell", path: "scripts/dashboard.sh"
      end
    end

  end
end

