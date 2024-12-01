
This provides an explanation on initially configuring the controller and hosts.

It uses:
    * bootstrap_controller.yml
    * bootstrap_hosts.yml

For the rest of the playbooks no explanation will be provided 

NOTE:   This does NOT cover configuring the VM or Network for any VMs.
        It assumes all of the VMs are created and connected to the network.

Configuring Ansible Controller:

1. Launch a VM.
    - This example uses a RHEL 9 VM that is
        * registered (to redhat)
        * networked - It assumes the other VMs are already networked as well.
    - Other than that, it assumes it is a fresh VM with only the 'root' user.

2. Install ansible:

    yum install ansible-core

3. Copy the bootstrap_controller.yml file to the VM, and run the below command

    ansible-playbook bootstrap_controller.yml -e "username=<USER>"

    - Replace <USER> with name of the interactive account that will be added
    - NOT the 'ansible' user, that will be configured separately
    - This is, obviously, ran as the root user

4. Configure the password for <USER> and ansible, if desired

    passwd <USER>
    passwd ansible

6. The Controller is configured. Update the inventory file with managed VMs

    echo -e "[all]\n<IP>\n<IP>" > ~/ansible/inventory
    EX: echo -e "[all]\n192.168.1.36\n192.168.1.37" > inventory

    - This is something that can be hand-jammed manually.

TODO:

1. controller should be configured with ansible-vault


Configuring New Hosts:

IMPORTANT:

There are multiple ways to configure new hosts

This example will use 'root', this requires root to ssh into the VM

    - Verify this 'PermitRootLogin' setting is configured on new hosts:
      
        grep 'PermitRootLogin yes' /etc/ssh/sshd_config
    
    - If nothing is returned, add the setting to the file, and restart sshd:

        echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
        systemctl restart sshd

    - The other option is to add a User to the Host that has sudo permissions
        This is a sloppy method, when you can just turn on/off PermitRootLogin

1. Verify the correct modules are installed

    ansible-doc -l

    NOTE - This is only necessary if 'ansible-core' was installed, instead of the full 'ansible' package

2. If ansible.posix is missing run

     ansible-galaxy collection install ansible.posix

3. Run the bootstrap_hosts.yml playbook, provided in this repository

    ansible-playbook bootstrap_host.yml -i /home/<USER>/ansible/inventory --ask-pass -e "username=<USER>"

TODO:

 1. Add configure_hosts.yml. This will do all the actual configurations
 2. Remove PermitRootLogin
 3. Ran as ansible or <USER>
 4. Do all the configurations that should be done on every host
     a. Firewall, copy /etc/hosts, etc..
 5. Configure passwords for users 'ansible' and 'USER'
    Ansible - set an actual password since it shouldn't be used interactively
    USER - set a default password that should be changed.

 6. test adding 'sudo usermod -s /usr/sbin/nologin ansible' to bootstrap_controller.yml

    - name: Set shell for ansible user
      ansible.builtin.user:
        name: ansible
        shell: /usr/sbin/nologin

 7. See if you can:
    a. run playbooks as USER with 'become: ansible'
    b. run playboos as ansible.

 8. test configuring controller from scratch