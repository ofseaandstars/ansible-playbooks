# User Management

# Configuration

## Setting up an inventory file (Optional)

If you don't want to add (or haven't added) your hosts to `/etc/ansible/hosts`, then you should create an inventory file for ansible to use instead.

I have provided an example one (`inventory.yml.example`) which you can make a copy of and make adjustments to, to better match your environment. This will be used by some playbooks to identify which machines to target.

## Creating an encrypted password salt (Mandatory)

To ensure that all of your generated passwords are encrypted sufficiently, and to prevent accidental password exposure within your playbooks, it is **strongly** recommended that you follow this process.

> **Warning**
> 
> Failure to follow this step could lead to system compromise for all potential remote hosts you make changes to or configure.

### Step 1. Create a new Salt

The first step is to create a new salt for your passwords. This should (ideally) be 16 characters for a SHA-512 salt (which is what these playbooks expect).

To do this, run the following command:

```bash
read -sp 'Enter Salt: ' saltvar; echo -n $saltvar | ansible-vault encrypt_string --vault-id password_salt@prompt --stdin-name 'salt'
```

This will do the following things:

- Provide a user prompt for entering the salt. Entering a salt in plain-text via the terminal will make it appear within your shell history. This command resolves this issue by providing a protected input to enter your salt value.
- Passes the value of the salt into `ansible-vault` to generate a new variable called `salt` under the vault ID of `password_salt`.

Once you have run this command, and entered the appropriate details, you will be given a large chunk of text; this is your encrypted salt value. Copy this to your clipboard or store it somewhere safe for later.

### Step 2. Create a new variable file

Next, you should run the following commands to create a new variable file containing this generated salt:

```bash
cp variables/password_salt.yml.example variables/password_salt.yml
```
This will make a copy of the `password_salt.yml.example` file in your `variables/` directory for you to edit.

Replace the contents with your generated salt with the editor of your choice. Then, run the following command:

```bash
ansible-vault encrypt --vault-id password_salt@prompt variables/password_salt.yml
```

This will ensure that your file is encrypted with the same password you used to encrypt the salt in the first place.

Now that you have done this, your encrypted salt will be decrypted when running the `ansible-playbook` command with the additional argument `--vault-id password_salt@prompt`.

Doing this ensures that you do not need to include your password salt into the playbooks directly (for security purposes). The same logic should be applied anywhere where something should not be exposed (such as passwords).

### Important - Refreshing your salt

If, for any reason, you need to refresh your salt (such as it being compromised/leaked), you should do the following:

- Repeat `Step 1. Create a new salt`.
- Run the below command to replace the salt variable.
- Change **all** passwords that have been created with the previous salt. You should be able to do this with the `rotate-password.yml` playbook.

To amend the salt file that already exists, run:

```bash
ansible-vault edit variables/password_salt.yml --vault-id password_salt@prompt
```

This will open a new `vim` editor for you to edit your file. The quickest way to replace it would be to copy the new salt to your clipboard, and then type the following keystrokes:

```vim
dd
dG
Ctrl+V
:wq
ENTER
```

This will wipe the existing contents of the file, let you paste the new contents, and then 'Write and Quit' the file.

----

# Playbooks

## Create Ansible Account
**_Requires Password Salt to be configured_**

This playbook creates a new `ansible` user on the specified hosts.

> **Note**
> 
> This requires there to be a sudo user that you have access to, and SSH to be enabled on the remote machine.

> **Note**
> 
> If you are running this for the first time, you can also specify the `--extra-vars "target=localhost generateSSH=true"` to generate an SSH key-pair for your ansible control node to distribute to remote hosts later on. This is useful if you wish to disabled password-based authentication on remote hosts.

### Usage

To run with a single host (that you have listed in `/etc/ansible/hosts`):
```bash
ansible-playbook create-ansible-account.yml --extra-vars "target=<host>" --vault-id password_salt@prompt
```

To run with hosts from a file:
```bash
ansible-playbook create-ansible-account.yml -i <path/to/inventory.yml> --extra-vars "target=<host_group>" --ask-pass --ask-become-pass --vault-id password_salt@prompt
```

----

## Distribute SSH Public Key (WIP)

This playbook will distribute the public SSH key created during the account creation process for the control node to other remote hosts for future use.

### Usage

```bash
ansible-playbook -i <path/to/inventory.yml> distribute-ssh-public-key.yml --extra-vars "target=<hostgroup>" --ask-become-pass --ask-pass
```

----

## Change User Password
**_Requires Password Salt to be configured_**

This is a simple playbook that allows you to modify the password of a user on the remote host. You will be prompted for the username and the subsequent password.

### Usage

```bash
ansible-playbook change-user-password.yml --extra-vars "target=<host>" --vault-id password_salt@prompt
```

----

## Rotate Passwords
**_Requires Password Salt to be configured_**

This playbook allows you to specify a unique password for any host that you have listed in an inventory file or in `/etc/ansible/hosts`.

This playbook requires a bit of set up to function correctly.

### Configuration - Passwords File

In order to ensure that your passwords are kept safe, and not exposed on the control node, we need to do the following:

- Set up a new encrypted file using `ansible-vault`
- Register the file within our playbooks

To set up a new encrypted file, take a look at the file `variables/new_passwords.yml.example` and adjust it to your liking on your host machine. 

Run the following command to make a copy you can edit:

```bash
cp variables/new_passwords.yml.example variables/new_passwords.yml
```

Make sure that any hosts you include here are defined in an inventory file somewhere on your host. 

If you are unsure as to what that should look like, see the above section `Configuration -> Setting up an inventory file`.

Then, run the following command:

```bash
ansible-vault encrypt --vault-id rotating_passwords@prompt variables/new_passwords.yml
```
This will encrypt the file for you, and allow you to establish your new variables for this playbook.

> **Note**
> 
> If you have set up your `new_passwords.yml` file locally before running this command, you can just copy-paste the contents into the editor (making sure to delete the file from your machine afterwards).

Once you've configured the `new_passwords.yml` file, type `:wq` and then hit ENTER in order to save your changes.

The editor should now close and you should be good to go.

### Usage

```bash
ansible-playbook rotate-passwords.yml -i <path/to/inventory.yml> --vault-id rotating_passwords@prompt --vault-id password_salt@prompt --extra-vars "target=<host_group>" --ask-become-pass --ask-pass
```