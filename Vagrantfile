# WSUWP Indie Vagrant Configuration
#
# This is the development Vagrantfile for the WSUWP Indie project. This
# Vagrant setup helps to describe an environment for local development that
# matches the WSUWP Indie production setup as closely as possible.
#
# We recommend Vagrant 1.4.x and Virtualbox 4.3.x
#
# -*- mode: ruby -*-
# vi: set ft=ruby :
vagrant_dir = File.expand_path(File.dirname(__FILE__))

Vagrant.configure("2") do |config|

  # Virtualbox specific settings to allocate memory to the virtual machine and to
  # provide a name for easier tracking.
  config.vm.provider :virtualbox do |v|
    v.customize ["modifyvm", :id, "--memory", 512]
    v.name = "wsuwp-indie-dev"
  end

  # CentOS 6.4, 64 bit release
  #
  # Provides a fairly bare-bones CentOS box created by Puppet Labs.
  config.vm.box     = "centos-64-x64-puppetlabs"
  config.vm.box_url = "http://puppet-vagrant-boxes.puppetlabs.com/centos-64-x64-vbox4210-nocm.box"

  # Set the default hostname and IP address for the virtual machine. If you have any other
  # Vagrant environments on the 10.10.30.x subnet, you may want to consider modifying this.
  config.vm.hostname = "wsuwp-indie-dev"
  config.vm.network :private_network, ip: "10.10.40.40"

  # The default SSH port in Vagrant is 2222. This helps avoid conflicts, even though Vagrant
  # does what it can to auto correct those conflicts.
  config.vm.network "forwarded_port", guest: 22, host: 2201, auto_correct: true

  # Mount the local project's www/ directory as /var/www inside the virtual machine. This will
  # be mounted as the 'vagrant' user at first, then unmounted and mounted again as 'www-data'
  # during provisioning.
  config.vm.synced_folder "www", "/var/www", :mount_options => [ "uid=510,gid=510", "dmode=775", "fmode=774" ]

  # Mount the local project's pillar/ directory as /srv/pillar inside the virtual machine. This allows
  # us to pass arbitrary data to Salt during provisioning.
  config.vm.synced_folder "pillar", "/srv/pillar", :mount_options => [ "dmode=775", "fmode=664" ]

  #############################################################################
  # Automatic Hosts Entries
  #
  # In the following section, we make use of two plugins for Vagrant to add network
  # hosts entries to both your local (host) machine and the virtual (guest) machine.
  #
  # Add a `hosts` file to each of your project directories to provide additional
  # hosts when required. These should be added as one host per line.
  #############################################################################

  # Parse through the project level `hosts` files and put the hosts that are found
  # into a single array.
  paths = []
  Dir.glob(vagrant_dir + '/www/**/hosts').each do |path|
    paths << path
  end

  hosts = []
  paths.each do |path|
    new_hosts = []
    file_hosts = IO.read(path).split( "\n" )
    file_hosts.each do |line|
      if line[0..0] != "#"
        new_hosts << line
      end
    end
    hosts.concat new_hosts
  end

  # Local Machine Hosts (/etc/hosts on the host)
  #
  # If the Vagrant plugin hostsupdater is installed, each project's `hosts` files will
  # be parsed and the entries found will be added to your local machine's hosts file
  # so that you are able to access the server configured inside the guest machine.
  #
  # vagrant-hostsupdater https://github.com/cogitatio/vagrant-hostsupdater
  #
  # This may require the entry of a password in OSX or Linux, and the acceptance of a UAC
  # prompt in Windows.
  if defined? VagrantPlugins::HostsUpdater
    config.hostsupdater.aliases = hosts
  end

  # Virtual Machine Hosts (/etc/hosts on the guest)
  #
  # If the Vagrant plugin vagrant-hosts is installed, each project's `hosts` files will
  # be parsed and the entries found will be added to the virtual machine's hosts file
  # so that it is able to access itself at those network addresses.
  #
  # vagrant-hosts https://github.com/adrienthebo/vagrant-hosts
  #
  # This will only run during provisioning.
  if Vagrant.has_plugin?("vagrant-hosts")
    config.vm.provision :hosts do |provisioner|
      provisioner.add_host '127.0.0.1', hosts
    end
  else
    $error_msg = <<ERRORSS

    WARNING

    The vagrant-hosts plugin is recommended to ensure proper functionality with WSUWP Indie. Use the
    following command to install this plugin before continuing:

    vagrant plugin install vagrant-hosts

ERRORSS
    puts $error_msg
    abort()
  end

  $script =<<SCRIPT
    cd /tmp && rm -fr wsu-web
    cd /tmp && curl -o wsu-web.zip -L https://github.com/washingtonstateuniversity/wsu-web-provisioner/archive/master.zip
    cd /tmp && unzip wsu-web.zip
    cd /tmp && mv WSU-Web-Provisioner-master wsu-web
    cp -fr /tmp/wsu-web/provision/salt /srv/
    cp /tmp/wsu-web/provision/salt/config/local.yum.conf /etc/yum.conf
    rpm -Uvh --force http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
    sed -i 's/mirrorlist=https/mirrorlist=http/' /etc/yum.repos.d/epel.repo
    sh /tmp/wsu-web/provision/bootstrap_salt.sh -K stable
    rm /etc/salt/minion.d/*.conf
    rm /etc/salt/minion_id
    echo "wsuwp-indie-dev" > /etc/salt/minion_id
    cp /srv/salt/minions/wsuwp-indie.conf /etc/salt/minion.d/
    salt-call --local --log-level=info --config-dir=/etc/salt state.highstate
SCRIPT

  config.vm.provision "shell", inline: $script

end