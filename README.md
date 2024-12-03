
This provides an explanation on initially configuring the controller and hosts.

It uses:
    * bootstrap_controller.yml
    * bootstrap_hosts.yml

For the rest of the playbooks no explanation will be provided 

NOTE:

This does NOT cover configuring the VM or Network for any VMs.
It assumes all of the VMs are created and connected to the network.

IMPORTANT SECURITY NOTICE:

The bootstrap playbooks add two users to the controller and hosts.
    The <USER> passed as 'username' and a user called 'ansible'.
The <USER> is given the ansible user's private key 'ansible_id_rsa'.
<USER> logs into the hosts as 'ansible' to perform all the actions.
User's can use ansible's private key, to run ansible, and not share an account.
Sharing an account is worse than sharing a private key on one host, imo.
But you can easily edit the playbooks to change this configuration.

<USER> is added to all the hosts as well, so either account can run ansible.
For <USER> just change ansible.cfg for remote_user = {{ username }}.

Frankly, there is no 100% secure way to use the ansible CLI.
You are either sharing accounts or sharing a private key.

IMPORTANT NOTES:

When I refer to User and the "username" variables used in bootstrap playbooks.
It is a regular interactive user account that will be put on all the hosts.
The 'ansible' project directory, and all these, playbooks should be
under that User on the Ansible Controller. So, create that User manually.

THE ABOVE WON'T WORK.

THIS HAS TO START ENTIRELY IN THE ROOT DIRECTORY, AS THE dboyd USER IS 
CREATED DURING BOOTSTRAPPING THE CONTROLLER.

UPDATE THE PLAYBOOKS SO IT ALL RUNS UNDER ROOT TO START, AND THEN IS COPIED OVER TO dboyd USER.

TODO ABOVE *****************************

Running Playbooks as root user:

The bootstrap playbooks, and these instructions, are designed to be ran as
the root user. The 'ansible' project directory, and everything in it,
will be created and ran under the root user. After initial configuration,
the 'ansible' project will be transferred to the User created.

Initial Set-up of Ansible Project, and configuring controller:

The bootstrap_controller.yml playbook uses vault to store the passphrase for
two SSH Keys it creates: ansible and User

1. Install ansible:

    yum install ansible-core

2. Create the 'ansible' directory under root account
    
    mkdir /root/ansible

3. Create an ansible vault in 

    ansible-vault create /root/ansible/vault

4. Input a password for the vault

5. Put in two variables:

    ansible_ssh_passphrase: <passphrase for ansible ssh key>
    username_ssh_passphrase: <passphrase for USER ssh key>

6. Create a 'vars' directory under /root/ansible

    mkdir /root/ansible/vars

7. Put "username" variable in a bootstrap_vars file:

    echo "username: <USER>" > /root/ansible/vars/bootstrap_vars

8. Create Playbooks directory

    mkdir /root/ansible/playbooks

9. Add bootstrap_controller.yml and bootstrap_hosts.yml files under playbooks

10. Create the inventory.yml file, add groups for 'all', 'new', and 'bastions'

    NOTE - There many ways to configure an inventory file.
         - I provided an example_file/inventory.yml
         - The 3 groups: 'all', 'new', and 'bastions' are required
         - as they are used in the playbooks.

11. Verify the correct modules are installed

    ansible-doc -l

    NOTE - This is only necessary if 'ansible-core' was installed

12. If ansible.posix is missing run

     ansible-galaxy collection install ansible.posix

13. Run the playbook 'bootstrap_controller.yml'

    ansible-playbook bootstrap_controller.yml --ask-vault-pass

14. Configuring hosts as root requires this setting in /etc/ssh/sshd_config

    'PermitRootLogin yes'

    - Verify this 'PermitRootLogin' setting is configured on new hosts:
      
        grep 'PermitRootLogin yes' /etc/ssh/sshd_config
    
    - If nothing is returned, add the setting to the file, and restart sshd:

        echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
        systemctl restart sshd

15. Run the bootstrap_hosts.yml playbook, provided in this repository

    ansible-playbook bootstrap_host.yml -i /root/ansible/inventory.yml --ask-pass --ask-vault-pass

    NOTE - This require the root password for all the hosts to be the same.
           Since, it is used to SSH into the hosts as root.
         - -i is required because root ansible.cfg point to USER inventory
            So, if root is used in the future it's using an updated inventory

TODO:

 * Add ansible vault configuration to ansible-controller

 * Finish change_hostname_and_resubscribe.yml
    Need to re-subscribe host if it is redhat os.

 * Add configure_hosts.yml. This will do all the actual configurations
 
 * Remove PermitRootLogin
 
 * Do all the configurations that should be done on every host
     a. Firewall, copy /etc/hosts, etc..
 
 * Configure passwords for users 'ansible' and 'USER'
    Ansible - set an actual password since it shouldn't be used interactively
    USER - set a default password that should be changed.

 * test configuring controller from scratch



 DONE:

 3. Ran as ansible or <USER>  - DONE
    Both work out the gate, but <USER> uses ansible's private key