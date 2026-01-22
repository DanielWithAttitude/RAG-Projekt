VAGRANTFILE_API_VERSION = "2"

NODES = {
  "k3s-master" => { cpus: 2, memory: 4096 },
  "k3s-worker" => { cpus: 2, memory: 2048 }
}

if !File.exist?("./vagrant_rsa")
  system("ssh-keygen -t rsa -b 2048 -f ./vagrant_rsa -q -N ''")
end
SSH_PUB_KEY = File.read("./vagrant_rsa.pub").strip

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "generic/debian12"
  config.ssh.insert_key = false
  config.ssh.private_key_path = ["./vagrant_rsa", "~/.vagrant.d/insecure_private_key"]
  config.vm.synced_folder ".", "/vagrant", disabled: true

  NODES.each do |name, opts|
    config.vm.define name do |node|
      node.vm.hostname = name

      node.vm.provider "hyperv" do |hv|
        hv.vmname       = name
        hv.cpus         = opts[:cpus]
        hv.memory       = opts[:memory]
        hv.linked_clone = true
        hv.enable_virtualization_extensions = true
      end

      node.vm.provision "shell", inline: <<-SHELL
        set -euxo pipefail

        # Konfiguracja SSH (klucz publiczny)
        install -d -m 700 /root/.ssh
        touch /root/.ssh/authorized_keys
        chmod 600 /root/.ssh/authorized_keys
        grep -qxF '#{SSH_PUB_KEY}' /root/.ssh/authorized_keys || echo '#{SSH_PUB_KEY}' >> /root/.ssh/authorized_keys
        
        if grep -qE '^#?PermitRootLogin' /etc/ssh/sshd_config; then
          sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
        else
          echo 'PermitRootLogin prohibit-password' >> /etc/ssh/sshd_config
        fi
        systemctl restart ssh
      SHELL
    end
  end
end
