Want to use SSH Certificates

Going to use Hetzner for VPN, despite latency (only ~125ms)

Taking notes for SSH certificate setup from: https://smallstep.com/blog/use-ssh-certificates/

I want to have the router also handle all of the DNS requests, using unbound for recursive DNS cacheing and validating, and NSD for internal queries.

Would DHCP be necessary? Probably.

First goals, though, are to get SSH setup, without a CA and everything for now.

Then a firewall.

Then fail2ban.

Then nsd+unbound, resolving only local queries from a particular subnet.

After that, we can add Samba in AD mode and a VPN with routing, whichever sounds more fun.


First off, repository: https://github.com/mawillcockson/router-configs

Following the directory layout from: https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html#directory-layout

Ansible installed under cygwin by

```sh
python -m pip install --user -U pipx
python -m pipx ensurepath
pipx install ansible
```

