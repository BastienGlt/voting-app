# -*- mode: ruby -*-
# vi: set ft=ruby :

NODES = {
  "manager1" => "192.168.99.100",
  "worker1" => "192.168.99.101",
  "worker2" => "192.168.99.102",
}

Vagrant.configure("2") do |config|
  NODES.each do |(node_name, ip_address)|
    config.vm.define node_name do |node|
      node.vm.box = "bento/ubuntu-24.04"
      node.vm.hostname = node_name
      node.vm.network "private_network", ip: ip_address

      # Configuration des ports pour le Manager
      if node_name == "manager1"
        node.vm.network "forwarded_port", guest: 5000, host: 5000 # Vote App
        node.vm.network "forwarded_port", guest: 5001, host: 5001 # Result App
        node.vm.network "forwarded_port", guest: 8080, host: 8080 # Visualizer (optionnel)
      end

      node.vm.provider "virtualbox" do |v|
        v.name = node_name
        v.memory = "2048"
        v.cpus = "2"
      end

      # Script d'installation automatisé
      node.vm.provision "shell", inline: <<-SHELL
        set -e
        
        # 1. Configuration DNS locale (pour que les machines se reconnaissent par nom)
        #{NODES.map { |n_name, ip| "echo '#{ip} #{n_name}' >> /etc/hosts" }.join("\n        ")}

        echo "==> Mise à jour des paquets..."
        sudo apt-get update -y

        echo "==> Installation de Docker (Version Ubuntu - Stable)..."
        # On installe docker.io (le moteur) ET docker-compose-v2 (le plugin pour 'docker compose')
        sudo apt-get install -y docker.io docker-compose-v2

        echo "==> Configuration des permissions..."
        sudo usermod -aG docker vagrant
        sudo systemctl enable docker
        sudo systemctl start docker

        # 2. Configuration automatique du Cluster Swarm
        # On quitte le swarm s'il existe déjà pour éviter les erreurs au redémarrage
        sudo docker swarm leave --force || true

        if [ "#{node_name}" = "manager1" ]; then
          echo "==> Initialisation du Swarm (Manager)..."
          sudo docker swarm init --advertise-addr #{ip_address}
          # On sauvegarde le token dans le dossier partagé /vagrant pour que les workers le voient
          sudo docker swarm join-token worker -q > /vagrant/swarm-worker-token
          echo "==> Token généré."
        else
          echo "==> Attente du token du Manager..."
          # On attend que le fichier token soit créé par le manager
          while [ ! -f /vagrant/swarm-worker-token ]; do sleep 2; done
          
          echo "==> Jonction au Swarm..."
          sudo docker swarm join --token $(cat /vagrant/swarm-worker-token) #{NODES["manager1"]}:2377
        fi
      SHELL
    end
  end
end