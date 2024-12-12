
<h1>Ansible Bootstrapping</h1>

This provides an explanation on initially configuring the controller and hosts.

The following playbooks are used:
- bootstrap_controller.yml
- bootstrap_hosts.yml
- configure_hosts.yml

<h3>IMPORTANT NOTE:</h3>

This does NOT cover creating the VM or Networking any VMs.
It assumes all of the VMs are created and connected to the network.

<h3>IMPORTANT SECURITY NOTICE:</h3>

The bootstrap playbooks adds two users to the controller and hosts.

- The <b>USER</b> passed as 'username' - the interactive user account.
- A user called 'ansible' - the account used to run ansible jobs on hosts.

<mark>The **USER** is given the ansible user's private key 'ansible_id_rsa'.</mark>

I did this so it is not necessary to log in as 'ansible' to work with ansible.
The <b>USER</b> logs into the hosts as 'ansible' (remote_user) to perform all the actions.

So, the <b>USER</b> can use ansible's private key, to run ansible, and not share an account.
Sharing an account is worse than sharing a private key amongst user's on the same host, in my opinion.
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

    `eval "$(ssh-agent -s)"`

    `ssh-add ~/.ssh/ansible_id_rsa`

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


<h1>Initial Set-up of Ansible Project, and configuring controller:</h1>

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

<h1>Configuring Hosts</h1>

For these steps you will run ansible as the configured <b>USER<b>, and not as root. The playbook will change 'PermitRootLogin' to 'no', so it will no longer work.

1. Log in as <b>USER<b>.

2. Install ansible.posix as <b>USER<b>:

    `ansible-galaxy collection install ansible.posix`

3. Run the playbook, as <b>USER<b>

    `ansible-playbook configure_hosts.yml`

<b>IMPORTANT<b> - If prompted for ssh passphrase, you just need to run the below commands:

    `eval "$(ssh-agent -s)"`

    `ssh-add ~/.ssh/ansible_id_rsa`

<h1>TODO:</h1>

- Fix 'get_url' module not working on debian hosts

- Use ansible to install ansible-galaxy collections for <b>USER<b>

- test configuring controller from scratch
