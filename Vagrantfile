# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "cloud-image/debian-13"
  config.vm.box_check_update = false

  # Cloud images usually lack 9p/virtiofs guest support; disable the default
  # share so the boot doesn't stall on a failed /vagrant mount.
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider :libvirt do |libvirt|
    libvirt.driver = "kvm"
    libvirt.uri = "qemu:///system"
    libvirt.memory = "2048"
    libvirt.cpus = 2
  end

  config.vm.define "harden" do |h|
    h.vm.hostname = "harden"
    h.vm.network "private_network", ip: "10.99.0.10"
  end
end
