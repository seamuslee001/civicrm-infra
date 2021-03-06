# Setup worker node (3rd generation)

See also: https://wiki.jenkins-ci.org/display/JENKINS/Distributed+builds

## Set timezone to UTC

```
dpkg-reconfigure tzdata
```

## Create `jenkins` user

```bash
root@test-XXX:~# adduser --home /home/jenkins --shell /bin/bash jenkins
Adding system user `jenkins' (UID 1007) ...
Adding new user `jenkins' (UID 1007) with group `jenkins' ...
Creating home directory `/home/jenkins' ...
root@test-XXX:~# makepasswd
ranDOmlYg3N3rAteD
root@test-XXX:~# passwd jenkins
Enter new UNIX password: ranDOmlYg3N3rAteD
Retype new UNIX password: ranDOmlYg3N3rAteD
passwd: password updated successfully
root@test-XXX:~# adduser jenkins ssh-user
root@test-XXX:~# sudo -u jenkins -H vi ~jenkins/.gitconfig
```

And setup an identity:

```
[color]
        ui = true
[user]
        name = Jenkins
        email = jenkins@test-XXX.civicrm.org
```

Use the new username and password to grant Jenkins access to SSH
into the new system. Test that it works.

```bash
me@localhost:~$ ssh test-master.civicrm.osuosl.org
me@test-master:~$ sudo -i ssh-copy-id -i /var/lib/jenkins/id_rsa.pub jenkins@HOSTNAME.civicrm.org
[sudo] password for me:
The authenticity of host 'HOSTNAME.civicrm.org (111.111.111.111)' can't be established.
RSA key fingerprint is 11:11:11:11:11:11:11:11:11:11:11:11:11:11:11:11.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'HOSTNAME.civicrm.org' (RSA) to the list of known hosts.
jenkins@HOSTNAME.civicrm.org's password:
Now try logging into the machine, with "ssh 'jenkins@HOSTNAME.civicrm.org'", and check in:

  ~/.ssh/authorized_keys

to make sure we haven't added extra keys that you weren't expecting.
me@test-master:~$ sudo ssh -i /var/lib/jenkins/id_rsa jenkins@HOSTNAME.civicrm.org
jenkins@HOSTNAME:$ git config -l
```

Finally, restrict modifications to `~/.ssh/authorized_keys`:

```
root@test-XXX:~# chown root ~jenkins/.ssh/authorized_keys
```

## Install nix and bknix

```bash
# Install the multiuser `nix` package manager per https://nixos.org/nix/manual/#sect-multi-user-installation
sudo apt-get install rsync
sh <(curl https://nixos.org/nix/install) --daemon

# Install bknix profiles and services per https://github.com/totten/bknix/blob/master/doc/install-other.md
sudo -i bash
git clone https://github.com/totten/bknix /root/bknix
cd /root/bknix
./bin/install-ci.sh
```

NOTE: There are a handful of profiles (`min`/`max`/`dfl`) with different
combination of services (PHP 5.6 + MySQL 5.5; PHP 7.0 + MySQL 5.7; etc). 
When using `install-ci.sh`, a profile (eg `min`) will have some corresponding
artifacts:

* Binaries folder (ex: `/nix/var/nix/profiles/bknix-min`)
* Data folder (ex; `/home/jenkins/bknix-min`)
* Systemd service (ex: `/etc/systemd/system/bknix-min.service` and `bknix-min-mysql.service`)

> TIP: To upgrade the bknix binaries and systemd services, just `git pull`,
> re-run `install-ci.sh` (with the same options as before), and restart
> the systemd services.

## Setup nginx redirect

Each profile runs a set of services on alternate ports (e.g. HTTP on 8001,
8002, 8003). But some agents (like the Github bot) post hyperlinks without
specifying a port. We can use nginx to redirect.

```
apt-get install nginx
rm /etc/nginx/sites-enabled/default
vi /etc/nginx/sites-enabled/redirect-8001.conf
```

with content

```
server {
  listen 80 default_server;
  listen [::]:80 default_server;
  return 302 http://$host:8001$request_uri;
}
```

## Register worker on master

The worker needs to have Java JRE 8+ available

```
sudo apt-get install openjdk-8-jre-headless
```

In Jenkins Web UI, navigate to "Manage Jenkins => Manage Nodes" and register the new node. Some example settings:

* __Number of executors__: 3
* __Labels__: Some combination of these: 
    * `bknix-durable` (the system is suitable for hosting HTTP test sites for a few days; e.g. used by `CiviCRM-Core-PR` or `CiviCRM-Drupal-PR`)
    * `bknix-tmp` (the system is suitable for hosting one-off test runs; e.g. used by `CiviCRM-Core-Matrix` or `CiviCRM-Ext-Matrix`)
    * `bknix-publisher` (the system is suitable for handling official builds; it should only execute approved/official code -- and *not* PR code; it may have more privileges for publication)
* __Remote FS Root__: /home/jenkins
    * Launch method: ...Unix machine via SSH
    * Credentials: jenkins
    * Host Key Verification Strategy: Manually provided key
        * (Copy from a file like `/etc/ssh/ssh_host_rsa_key.pub` or `/etc/ssh/ssh_host_ecdsa_key.pub`)

## Open HTTP ports in firewall

See, e.g., https://github.com/civicrm/civicrm-infra/commit/f3d58a2f20a0240c4aa4a1ba90af91d20937a33e

## Setup a site-list

```
su - jenkins
eval $(use-bknix dfl)
civibuild create site-list
```

Then edit the `site-list.settings.php` and have it read all the build dirs, e.g.

```php
$sitelist['bldDirs'] = ['/home/jenkins/bknix-dfl/build', '/home/jenkins/bknix-max/build', '/home/jenkins/bknix-min/build'];
```

Note the `SITE_KEY` from the `site-list.settings.php`. Then, register the new server as part of the general index by connecting to `test-1.civicrm.org` and adding an entry to `/home/jenkins/bknix-dfl/build/site-list/more-sites.php`.
