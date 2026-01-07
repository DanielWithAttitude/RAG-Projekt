VAGRANTFILE_API_VERSION = "2"

NODES = {
  "k3s-master" => { cpus: 2, memory: 2048, mac: "00:15:5d:8b:5e:54", ip: "10.0.0.28", gw: "10.0.0.1", dns: "10.0.0.1" },
  "k3s-worker" => { cpus: 2, memory: 2048, mac: "00:15:5d:8b:5e:55", ip: "10.0.0.29", gw: "10.0.0.1", dns: "10.0.0.1" }
}

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "generic/debian12"
  config.ssh.insert_key = false
  config.vm.synced_folder ".", "/vagrant", disabled: true

  public_key = <<-KEY.strip
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC8evseV+vNAL46w4wMBfLoixKncfBq4p7rnwoswHaNZB+Pf584es5/yEZgjEhYE7QfGmfb7JDYhFspZBNay/F7yuGy7sNx0LCpcmOWDBia4xiD+yYONLyjqQjkzIQ1lnfKzFujQeqBVjBxH12mSNEOX7dtgGOnOim9VBOYwL8RUd6RFDSgxw6+R+B8k5cOwJF5H4Di6/qtUCjr4rLjB/mSoZW9gdVZKRJ0zjkRjsO1r6fLAsCtKJ47kFY06A4WzpAFyoGPMrfm+Hk4a1uY0uo3EeHkMWrLuuxvz8YEKIUn11gCUR5Xvb0uXmJ+wqdtJaWqfwMDYo5zPgp4j2dG9qQeGJ3Tp/yfsR++vJ8j64L3g+ROBWrrt9SZgNeOHSHUmvTLA2kNkurMDC5MJ+UdKxXbMMadEGQ1YvNfr64U3DEHAYCd3Vh4U3s6Rc5bCMNlQu5KcTZrgrlXCzyoEkZZ8kVijJM6loryAt5Dk3sSe0GUjMs//reLCloPzRuoeCwRjnU= pysio@DESKTOP-639E839
  KEY

  NODES.each do |name, opts|
    config.vm.define name do |node|
      node.vm.hostname = name

      node.vm.provider "hyperv" do |hv|
        hv.vmname       = name
        hv.cpus         = opts[:cpus]
        hv.memory       = opts[:memory]
        hv.linked_clone = true
        hv.mac          = opts[:mac]
      end

      # Używamy vSwitch "Internal", ale BEZ autokonfiguracji IP przez Vagranta
      node.vm.network "public_network", bridge: "Internal", auto_config: false

      node.vm.provision "shell", inline: <<-SHELL
        set -euxo pipefail

        # SSH root pubkey (jak u Ciebie)
        install -d -m 700 /root/.ssh
        touch /root/.ssh/authorized_keys
        chmod 600 /root/.ssh/authorized_keys
        grep -qxF '#{public_key}' /root/.ssh/authorized_keys || echo '#{public_key}' >> /root/.ssh/authorized_keys

        if grep -qE '^#?PermitRootLogin' /etc/ssh/sshd_config; then
          sed -i 's/^#\\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
        else
          echo 'PermitRootLogin prohibit-password' >> /etc/ssh/sshd_config
        fi
        systemctl restart ssh

        # === Statyczny adres przez systemd-networkd (pewne na Debian 12) ===
        # Dopasowanie po MAC gwarantuje, że konfiguracja trafi w dobrą kartę Hyper-V.
        cat >/etc/systemd/network/10-hyperv.network <<'EOF'
        [Match]
        MACAddress=#{opts[:mac].downcase}

        [Network]
        Address=#{opts[:ip]}/24
        Gateway=#{opts[:gw]}
        DNS=#{opts[:dns]}
        Domains=home.internal
        IPv6AcceptRA=no
        EOF

        # Wyłącz wszelkie DHCP na tej karcie (jeśli były)
        pkill -f dhclient || true

        # Upewnij się, że networkd i resolved są włączone
        systemctl enable systemd-networkd systemd-resolved
        systemctl restart systemd-networkd

        # (Opcjonalnie) ustaw /etc/resolv.conf na systemd-resolved
        if [ -e /run/systemd/resolve/stub-resolv.conf ]; then
          ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
        else
          printf "nameserver #{opts[:dns]}\\nsearch home.internal\\n" > /etc/resolv.conf
        fi

        (sleep 2 && reboot) &
      SHELL
    end
  end
end
