
<h1>Ansible Bootstrapping</h1>

This repository provides ansible playbooks for initially bootstrapping/configuring the ansible-controller and all existing hosts.

There are two playbooks for the initial set-up of ansible, and they both end in "_initial_setup".

 - bootstrap_controller_initial_setup.yml
 - bootstrap_hosts_initial_setup.yml

 The "bootstrap_controller_initial_setup.yml" configures the ansible-controller, and the "bootstrap_hosts_initial_setup.yml" configures the hosts. 
 
The only difference between these "bootstrap_hosts_initial_setup.yml" and "bootstrap_hosts.yml" is the "initial_setup" playbook is pointing to files in /root and "bootstrap_hosts.yml" is pointing to files in the USER's directory. The "bootstrap_hosts.yml" playbook is meant to be ran to configure new hosts created after the initial configuration of ansible; there is an explanation at the end of this README that explains how to bootstrap/configure new hosts, after the initial setup is complete. I might try to reconcile these two playbooks in the future.

The playbooks below can be used anytime after the initial set-up of ansible to configure new hosts:
- bootstrap_hosts.yml
    - Sets up host to allow ansible to be used on it: add ansible user, add ssh keys, etc...
- configure_hosts.yml
    - Performs basic configurations: install software, adds configurations, etc...
- update_hosts.yml
    - Runs update commands on all hosts. Doesn't do any backup or testing, might add that in the future
- change_hostname_and_resubscribe.yml
    - This playbook is for redhat hosts. It changes the hostname and resubscribes the host to redhat.

<h3>IMPORTANT NOTE:</h3>

This obviously does NOT cover creating the VM or Networking any VMs.
It assumes all of the VMs are created and connected to the network.

<h3>IMPORTANT SECURITY NOTICE:</h3>

The bootstrap playbooks adds two users to the controller and hosts.

- <b>USER</b> - The <b>USER</b> passed as 'username' - the interactive user account.
- ansible - A user called 'ansible' - the account used to run ansible jobs on hosts.

<mark>The **USER** is given the ansible user's private key 'ansible_id_rsa'.</mark>

I did this so it is not necessary to log in as 'ansible' to work with ansible.
The <b>USER</b> logs into the hosts as 'ansible' (remote_user) to perform all the actions.

So, the <b>USER</b> can use ansible's private key, to run ansible, and not share an account.
Sharing an account is worse than sharing a private key amongst users on the same host, in my opinion.
But you can easily edit the playbooks to change this configuration.

Frankly, there is no 100% secure way to use the ansible CLI with multiple users.
You are either sharing accounts or sharing a private key. Or having multiple users
with the 'same' playbooks in different locations, pushing from different files for
the 'same' playbook.

<h3>Passwords and Passphrases:</h3>

The instructions and playbooks are configured to set passphrases for both users.
They are configured as variables in an ansible vault.
There are instruction for how to pass in the ssh key passphrase
when running ansible jobs using an ssh-agent.

The ansible user is not configured with a password, it uses passwordless sudo.

The **USER** is configured with a password that is the same as the ssh key passphrase.


<h4>Saving ssh-key passphrase:</h4>

To save the passphrase for an ssh-key, in this case,saving the passphrase
for ansible user's private key while running as <b>USER</b>,
run the below commands.

    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/ansible_id_rsa

The first command starts the ssh-agent, and the second command adds the ssh key.
You will be prompted for the passphrase after running the second command.
This will allow you to run ansible-playbooks as <b>USER</b>, with ansible's
SSH key, without being prompted for the ssh-key-passphrase.


<h2>Running bootstrap playbooks as root:</h2>

The bootstrap playbooks, and these instructions, are designed to be ran as
the root user. The 'ansible' project directory, and everything in it,
will be created and ran under the root user. After initial configuration,
the 'ansible' project will be transferred to both users.

<h3>Hosts require 'PermitRootLogin yes' for bootstrapping:</h3>

Configuring hosts as root requires 'PermitRootLogin yes' setting in /etc/ssh/sshd_config:

- Verify this 'PermitRootLogin' setting is configured on new hosts:
      
    `grep 'PermitRootLogin yes' /etc/ssh/sshd_config`
    
- If nothing is returned, add the setting to the file, and restart sshd:

    `echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config`
    `systemctl restart sshd`

<b>IMPORTANT</b>: All hosts must have the same root password.
Only one password can be supplied when running the playbook using --ask-pass,
and it is used to SSH into all the hosts.


<h1>Initial Set-up of Ansible Project</h1>
<h2>Configuring controller</h2>

1. Install ansible:

    `yum install ansible-core`

2. Create the 'ansible' directory under root account:
    
    `mkdir /root/ansible`

3. Create an ansible vault:

    `ansible-vault create /root/ansible/vault`

4. Input a password for the vault

5. Put in two variables::

    `ansible_ssh_passphrase: <i>passphrase for ansible ssh key</i>`
    `username_ssh_passphrase: <i>passphrase for USER ssh key</i>`

6. Create a 'vars' directory under /root/ansible:

    `mkdir /root/ansible/vars`

7. Put "username" variable in a bootstrap_vars file:

    `echo "username: <b>USER</b>>" > /root/ansible/vars/bootstrap_vars`

8. Create Playbooks directory:

    `mkdir /root/ansible/playbooks`

9. Add bootstrap_controller.yml and bootstrap_hosts.yml files under playbooks.

10. Create the inventory.yml file, add groups for 'all' and 'bastions'.

<b>NOTE</b>
There many ways to configure an inventory file.
I provided an example inventory file: example_files/inventory.yml.
Only the 'all' and 'bastions' groups are required,
as they are used in the playbooks.

11. Verify the correct collections are installed

    `ansible-doc -l`

<b>NOTE</b> - Installing collections is only necessary if 'ansible-core'
was installed, instead of the entire ansible package.

12. If ansible.posix is missing run:

    `ansible-galaxy collection install ansible.posix`

13. Other important collections that are used in this repo's playbooks:

    `ansible-galaxy collection install community.general`

13. Run the playbook 'bootstrap_controller.yml':

    `ansible-playbook bootstrap_controller.yml --ask-vault-pass`

15. Run the bootstrap_hosts.yml playbook, provided in this repository

    `ansible-playbook bootstrap_host.yml -i /root/ansible/inventory.yml --ask-pass --ask-vault-pass`

<b>NOTE:</b> '-i /root/ansible/inventory.yml' is required because
root's ansible.cfg points to <b>USER</b>'s inventory.
This is so, if root is used in the future, it's using an updated inventory.

<h4>Copying the ansible vault</h4>

The vault file copied with the entire ansible project won't work.
It needs deleted and recreated with the below commands.

1. Delete the vault that was copied:

    `rm /home/<b>USER</b>/ansible/vault`

2. Decrypt the vault, and place the output in <b>USER</b> home directory:

    `ansible-vault decrypt /root/ansible/vault --output /home/<b>USER</b>/ansible/vault_dc`

3. Re-encrypt the vault:

    `ansible-vault encrypt /home/<b>USER</b>/ansible/vault_dc --output /home/<b>USER</b>/ansible/vault`

4. Change permissions:

    `chown <b>USER</b>:<b>USER</b> /home/<b>USER</b>/ansible/vault`

</h2>Configuring Hosts</h2>

For these steps you will run ansible as the configured <b>USER<b>, and not as root. The playbook will change 'PermitRootLogin' to 'no', so it will no longer work.

1. Log in as <b>USER<b>.

2. Install ansible.posix as <b>USER<b>:

    `ansible-galaxy collection install ansible.posix`

3. Run the playbook, as <b>USER<b>

    `ansible-playbook configure_hosts.yml`

<b>IMPORTANT<b> - If prompted for ssh passphrase, you just need to run the below commands:

        eval "$(ssh-agent -s)"
        ssh-add ~/.ssh/ansible_id_rsa

<b>Example Commands for running playbooks as USER</b>

Running 'change_hostname_and_resubscribe.yml':

    ```
    ansible-playbook -i 192.168.0.123, change_hostname_and_resubscribe.yml --ask-vault-pass -e 'hostname=new-hostname.example.com'
    ```

Running 'bootstrap_hosts.yml':

    ```
    ansible-playbook bootstrap_hosts.yml --ask-pass --ask-vault-pass
    ```

<h1>After Initial Bootstrapping of Controller</h1>

After everything is set up, below is the method for how you will add/configure new servers.

No matter what, a new server must be configured by running the "bootstrap_hosts.yml" playbook as root. This must be ran as root because a new server doesn't have the ansible user configured yet. The "bootstrap_hosts.yml" playbook will set all that up.

<h2>Bootstrapping a new server</h2>

1. Server must allow root to ssh. Add below to /etc/ssh/sshd_config

    `PermitRootLogin yes`

2. Set server root password

3. Add server to ansible-controller "/etc/hosts"

4. Add server to ansible-controller ansible inventory file

5. Test ssh as root to the new server

    - This allows you to not only test but trust the server's host key

6. As root, run "bootstrap_hosts.yml" playbook with command shown below:

    `ansible-playbook bootstrap_hosts.yml --ask-pass --ask-vault-pass`

7. Now this server can be configured by "ansible" user like all other servers

<h2>Configuring a new server</h2>

1. Go back to being the normal user (exit root)
2. Run these two commands to save the ansible ssh key in the ssh-agent

    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/ansible_id_rsa

3. Must enter the passphrase for "ansible_id_rsa" after running the 2nd command.
4. Run "configure_hosts.yml" playbook as shown below

    `ansible-playbook configure_hosts.yml`

5. The server is now configured.

Now run any other playbook/roles to configure the server with applications.
