# Ansible Playbooks

> **Note**
> 
> This repo is not particularly well-organised. I plan to clean up the general structure/handling of variables in the future. However, for my purposes it is functional. Use at your own risk.

This repo contains a collection of my ansible playbooks for various purposes.

# Prerequisites

## Libraries

You will require `sshpass` to be installed if you are intending to use SSH passwords to connect to any remote hosts.

```bash
sudo apt install sshpass -y
```

You'll also need to ensure you have `passlib` installed for handling passwords.

```bash
pip install passlib
```