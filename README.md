
# Ansible Bootstrapping

This provides an explanation on initially configuring the controller and hosts.

The following playbooks are used:
- bootstrap_controller.yml
- bootstrap_hosts.yml
- configure_hosts.yml

### IMPORTANT NOTE:

This does NOT cover configuring the VM or Network for any VMs.
It assumes all of the VMs are created and connected to the network.

### IMPORTANT SECURITY NOTICE:

The bootstrap playbooks add two users to the controller and hosts.

- The **USER** passed as 'username' - the interactive user account.
- a User called 'ansible' - the account used to run ansible jobs on hosts.

<mark>The **USER** is given the ansible user's private key 'ansible_id_rsa'.</mark>

I did this so it is not necessary to log in as 'ansible' to work with ansible.
The **USER** logs into the hosts as 'ansible' (remote_user) to perform all the actions.

So, the **USER** can use ansible's private key, to run ansible, and not share an account.
Sharing an account is worse than sharing a private key amongst user's on the same host, in my opinion.
But you can easily edit the playbooks to change this configuration.

Frankly, there is no 100% secure way to use the ansible CLI with multiple users.
You are either sharing accounts or sharing a private key.

### Passwords and Passphrases:

ssh key passphrases - The instructions and playbooks are configured to set
passphrases for both users. They are configured as variables in an ansible vault.
There are instruction for how to pass in the ssh key passphrase
when running ansible jobs using an ssh-agent.

The ansible user is not configured with a password, it uses passwordless sudo.

The **USER** is configured with a password that is the same as the ssh key passphrase
Running Playbooks as root user.

## Running bootstrap playbooks as root:

The bootstrap playbooks, and these instructions, are designed to be ran as
the root user. The 'ansible' project directory, and everything in it,
will be created and ran under the root user. After initial configuration,
the 'ansible' project will be transferred to both users.

## Hosts require 'PermitRootLogin yes' for bootstrapping:

Configuring hosts as root requires this setting in /etc/ssh/sshd_config:

    'PermitRootLogin yes'

- Verify this 'PermitRootLogin' setting is configured on new hosts:
      
    grep 'PermitRootLogin yes' /etc/ssh/sshd_config
    
- If nothing is returned, add the setting to the file, and restart sshd:

    echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
    systemctl restart sshd

- All hosts must have the same root password, only one can be used for SSH, when running the playbook.


# Initial Set-up of Ansible Project, and configuring controller:

1. Install ansible:

    yum install ansible-core

2. Create the 'ansible' directory under root account:
    
    mkdir /root/ansible

3. Create an ansible vault:

    ansible-vault create /root/ansible/vault

4. Input a password for the vault

5. Put in two variables::

    ansible_ssh_passphrase: <i>passphrase for ansible ssh key</i>
    username_ssh_passphrase: <i>passphrase for USER ssh key</i>

6. Create a 'vars' directory under /root/ansible:

    mkdir /root/ansible/vars

7. Put "username" variable in a bootstrap_vars file:

    echo "username: <b>USER</b>>" > /root/ansible/vars/bootstrap_vars

8. Create Playbooks directory:

    mkdir /root/ansible/playbooks

9. Add bootstrap_controller.yml and bootstrap_hosts.yml files under playbooks.

10. Create the inventory.yml file, add groups for 'all' and 'bastions'.

<b>NOTE</b>
There many ways to configure an inventory file.
I provided an example inventory file: example_file/inventory.yml
Only the 'all' and 'bastions' are required, as they are used in the playbooks.

11. Verify the correct modules are installed

    ansible-doc -l

<b>NOTE</b> - This is only necessary if 'ansible-core' was installed

12. If ansible.posix is missing run:

     ansible-galaxy collection install ansible.posix

13. Run the playbook 'bootstrap_controller.yml':

    ansible-playbook bootstrap_controller.yml --ask-vault-pass

15. Run the bootstrap_hosts.yml playbook, provided in this repository

    ansible-playbook bootstrap_host.yml -i /root/ansible/inventory.yml --ask-pass --ask-vault-pass

<b>NOTE:</b> -i is required because root ansible.cfg points to <b>USER</b>'s inventory.
This is so, if root is used in the future, it's using an updated inventory.

TODO:

 * Finish change_hostname_and_resubscribe.yml
    Need to re-subscribe host if it is redhat os.

 * Add configure_hosts.yml. This will do all the actual configurations
 
 * Remove PermitRootLogin
 
 * Do all the configurations that should be done on every host
     a. Firewall, copy /etc/hosts, etc..
 
 * test configuring controller from scratch
