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

Realized Asnible needs symlink support on the development machine.

So, we'll use VSCode remote development, and develop on the remote machine!

Easiest way seems to be to add the repository to the host, and work entirely remotely.

Since I like code signing and the remote host will need access to my SSH keys on my card, I'll use:

 - SSH agent forwarding: https://developer.github.com/v3/guides/using-ssh-agent-forwarding/
 - GnuPG Agent Forwarding: https://wiki.gnupg.org/AgentForwarding

This does not work completely, as Windows SSH can't do RemoteForward, and PuTTY can't seem to be able to do it with Unix sockets. There were some suggestions about using netcat or socat, but I just installed Debian-WSL instead.

Okay, now a simple firewall. Try nftables?

Well, let's first add the basic configuration for getting a just-built Hetzner server running.

We need to first login as root on the first run, and copy over the ssh_config file,
and then populate the /etc/ssh/authorized_keys file, and then restart sshd.

Taking lots of notes from https://github.com/cullum/dank-selfhosted

Inside the WSL distro, you need a /etc/wsl.conf that has at least:

```ini
[automount]
enabled = tue
root = /mnt/
options = "metadata"
```

Otherwise, it can't read all the NTFS permissions, so Ansible will complain about a world-writable directory.

Also, `umask 0022` will ensure any files created aren't writable by group or other.

Okay, NOW nftables firewall: https://www.youtube.com/playlist?list=PLgC62UIBo6cJ-wrlZ5lbqY2Ts6IkPWNtY

The instructions in https://wiki.archlinux.org/index.php/Nftables#Usage advise to load kernal modules.

The command, though, looks for already loaded kernel modules. We need the ones that aren't loaded, because a lot aren't loaded automatically. This pointed to where they are: https://unix.stackexchange.com/a/184880/235043

So we need to find the modules, make sure they're loaded on boot, and then load them now. This finds the names:

```bash
find /lib/modules/$(uname -r) -type f -regextype posix-egrep \
    -regex '.*/netfilter/nft[[:alnum:]_.]+$' \
    -printf %f\\n \
| grep -Eo "^[[:alnum:]_]+"
```

They need to go in /etc/modules-load.d/nftables.conf

And then `modprobe` each module into the kernel, so we don't have to reboot.

This video is decent: https://youtu.be/FXTRRwXi3b4?list=PLgC62UIBo6cJ-wrlZ5lbqY2Ts6IkPWNtY

Bssic firewall ruleset should:

 - Drop packets going in or out in private networks: https://en.wikipedia.org/wiki/Private_network . We'll deal with these when we get to the VPN.
 - Drop all packets from https://en.wikipedia.org/wiki/Martian_packet keeping in mind https://tools.ietf.org/html/rfc6441 from https://en.wikipedia.org/wiki/Bogon_filtering
 - Allow inet tcp/22
 - Allow all on iface lo
 - policy accept any outgoing packets not dropped and established,related packets
 - Include rules to be added for each additional role needing them (e.g. unbound)

I've only set up the bare minimum for a firewall, less than described above, but I'll leave that for now, and move to fail2ban for SSH.

Following: https://wiki.archlinux.org/index.php/Fail2ban

Switched to: https://www.fail2ban.org/wiki/index.php/MANUAL_0_8

Clear and concise instructions difficult to come by. From the jail.conf manual:

```
ignoreip
    list of IPs not to ban. They can include a DNS resp. CIDR mask too. The option affects additionally to ignoreself (if true) and don't need to contain own DNS resp. IPs of the running host.
```

The advice here seems to work: https://github.com/fail2ban/fail2ban/issues/1814#issuecomment-312016985