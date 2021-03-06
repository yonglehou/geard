demo
====

These instructions will take you through setting up vagrant and libvirt and using these tools to 
setup and execute a geard demo.  

Host setup
-------------

**NOTE: The following steps should be run as a non-root user.**

1.  Install Vagrant 1.6.2

        $ wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.6.2_x86_64.rpm
        $ yum -y --nogpgcheck localinstall vagrant_1.6.2_x86_64.rpm
        $ vagrant version
        Installed Version: 1.6.2
        Latest Version: 1.6.2

2.  Setup libvirt 

        $ sudo yum install @virtualization
        $ wget http://file.rdu.redhat.com/~decarr/dockercon/libvirt-setup.sh
        $ chmod u+x libvirt-setup.sh
        $ sudo ./libvirt-setup.sh
        $ sudo systemctl start libvirtd.service
        $ sudo systemctl enable libvirtd.service

3. Setup vagrant with libvirt

        $ sudo yum install -y wget tree vim screen mtr nmap telnet tar git
        $ curl -sSL https://get.rvm.io | bash -s stable
        $ sudo yum install -y libvirt-devel libxslt-devel libxml2-devel
        $ gem install nokogiri -v '1.5.11'
        # Make sure that vagrant isn't installed as a gem. If it is, then uninstall it.
        $ vagrant plugin install --plugin-version 0.0.16 vagrant-libvirt

4.  Setup Vagrant Box and review Vagrantfile

        vagrant box add --name=atomic --provider=libvirt http://download.eng.bos.redhat.com/rcm-guest/staging/walters/images/snaps/20140531.1/vagrant-libvirt/rhel-atomic-host.box

    Clone the geard project and check out the `Vagrantfile`:

        $ git clone git://github.com/openshift/geard
        $ cat geard/contrib/demo-multi/Vagrantfile

5.  Pull and start local docker registry.  This will be used to serve the docker images needed for
    the demo to the two VMs as well and to host images built during the demo.

        $ docker pull pmorie/geard-demo-registry
        $ docker run -p 5000:5000 pmorie/geard-demo-registry

    If you're going to demo local development, you should pull down the nodejs images from your local
    registry:

        $ docker pull localhost:5000/openshift/nodejs-0-10-centos
        $ docker tag localhost:5000/openshift/nodejs-0-10-centos openshift/nodejs-0-10-centos
        $ docker tag openshift/nodejs-0-10-centos nodejs-centos

6.  Install geard locally and clone the helloworld repo onto your host machine:

        $ git clone git://github.com/pmorie/nodejs-helloworld
        $ sudo yum -y --enablerepo=updates-testing update geard

7.  Start VMs

        $ vagrant up --provider=libvirt
        Bringing machine 'default' up with 'libvirt' provider...
        ==> default: HandleBoxUrl middleware is deprecated. Use HandleBox instead.
        ==> default: This is a bug with the provider. Please contact the creator
        ==> default: of the provider you use to fix this.
        ==> default: Creating image (snapshot of base box volume).
        ==> default: Creating domain with the following settings...
        ==> default:  -- Name:          dockercon_489e2b0ea1cb9397b6db6b1bd3a89ad8
        ==> default:  -- Domain type:   kvm
        ==> default:  -- Cpus:          4
        ==> default:  -- Memory:        4096M
        ==> default:  -- Base box:      atomic
        ==> default:  -- Storage pool:  default
        ==> default:  -- Image:         /var/lib/libvirt/images/dockercon_489e2b0ea1cb9397b6db6b1bd3a89ad8.img
        ==> default:  -- Volume Cache:  default
        ==> default: Creating shared folders metadata...
        ==> default: Starting domain.
        ==> default: Waiting for domain to get an IP address...
        ==> default: Waiting for SSH to become available...

    If you run into issues, try the following:

        $ rm -fr ./vagrant
        $ vagrant plugin list

    If you have other third-party plug-ins installed, try to remove them.  In particular, we found
    errors when running the following plug-ins with vagrant-libvirt:

    1.  vagrant-aws
    2.  vagrant-openshift

    Be patient. This will bring up two vm instances: `atomic-1` and `atomic-2`.  The provisioner on
    the initial vagrant up will fetch all required docker images to support the  demo from the local
    registry, and git clone required content.

8.  SSH access

    Validate that you can reach the VMs with ssh:

        $ vagrant ssh atomic-1
        $ docker images
        $ cd geard
        $ git branch
        * demo-multi
          master

        $ vagrant ssh atomic-2
        $ docker images
        * demo-multi
          master

    You can now run rpm-ostree commands, etc.  You can also see the vm running by viewing in 
    virt-manager:

        $ virt-manager

    You should see be able to see both instances running.

9.  OSTree Upgrade

    Next, we need to update to the latest packages.  On both VMs, run the following:

        $ rpm-ostree upgrade

    Then reboot the VMs:

        $ vagrant halt
        $ vagrant up --provider=libvirt && vagrant provision

    Note: The `vagrant-libvirt` provider does not support the `vagrant up --provision` flag.  The 
    provision step is required to work around an issue where the private network interface is not
    brought up on boot by vagrant controller for each VM to communicate.

10. Cockpit

    After running `vagrant up`, cockpit should be started and available at: `http://localhost:11001`.
    Use the following credentials:

        user: root
        password: atomic2014

    Once you have logged into cockpit, click the 'Add server' button on the right-hand side of the
    page.  Enter `atomic-2` for the hostname and leave `login with my current credentials` checked.

11. JMeter

    This demo uses JMeter to drive traffic.  Install JMeter onto your host from:

        http://jmeter.apache.org/download_jmeter.cgi

    Choose the binary distribution and unpack them; the dir the binaries are installed to will now
    be referenced in these instruction as `JMETER_HOME`.  Next we need a JMeter plugin:

        $ cd $JMETER_HOME
        $ curl http://rubenlaguna.com/wp/wp-content/uploads/2007/01/stataggvisualizer.zip > stataggvisualizer.zip
        $ unzip stataggvisualizer.zip

    Next you can bring JMeter up with:

        $ bin/jmeter.sh &

    Next you'll need to import the test plan from your local clone of `geard`.  It will be at:

        contrib/demo-multi/testplan.jmx