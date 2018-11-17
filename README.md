# secrets-rsync
Scripts and instructions to support secret distribution with rsync
--

The goal: permit only authorized environments (e.g., specific hosts) to retrieve secrets.

* If one runs puppet or chef, one can leverage client authentication (certificates) to distribute secrets.

* If one has complex requirements, there is Hashicorp vault.

* If one pushes with ansible (the usual case), the target's authentication is the ssh host key presented by sshd.

This solution is for other environments.

It expects to use host keys, as does ansible, but could be modified if there is an alternate pre-distributed ssh key.  It relies on the property that a private host key is no different than any other private ssh key.  Then the "secrets distribution server" uses forced commands in $HOME/.ssh/authorized_keys to permit access only to the secrets intended for that host.

Why rsync?  To preserve ctime and mtime attributes of the secret when the secret doesn't change - otherwise one would use scp.

The client invocation is
```rsync -e 'ssh -i /etc/ssh/ssh_host_rsa_key -l secrets-rsync' keyserver:/path/to/secret /path/to/secret```
To have access to the host key, the client must run as root.

In the `$HOME/.ssh/authorized_keys` file for account secrets-rsync, the public key for the host specifies a forced command:
```
command="/usr/local/bin/rrsync -ro client-host-name",no-port-forwarding,no-agent-forwarding,no-pty,no-X11-forwarding ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCnMo+Nh15wBUcCWnzjXQXcex6CnpElu37opv/W3kwd51dYbbBvasMQsiS5CouUPg63djN5Z9DXGsIrzVkOYBvNdxOUf5LHYXGexaXaqQEOUYQGRNrEdmGwlDs/WLsMs2Z6Zf8Uh2cK78zibe8OPXba2U095pmoBm5YJq/4ONwnooIK2aPrtNOvn/QQOWl9ZL0X70q7UXazkBUi32R2mIEjxvQd97Yjx0v3vr8lzIzebGUTsWKcmAhizxlztj4wCxv0ZPHYtuDRi6gSMUBzwLdhZb5qtlw4ZmCQXP8vbEIJO0PLww8fXjP2b5PVQyge6YoYQkxTiy6Un+4AlmRgyDe5
```
There is one such line per client host.

The script ``ingest-ssh-known-hosts`` translates a `known_hosts` file into such an `authorized_keys` file.  It requires that the first host name on the line (the one before any comma or space) be the name given as the client-host-name (subdirectory) argument to `/usr/local/bin/rrsync`.

The perl script rrsync, included in most distributions of rsync, typically in the documentation section, examines `$SSH_ORIGINAL_COMMAND` to sanitize the rsync server invocation.  The `-ro` flag specifies the client may only read.  In the home directory for the secrets-rsync user, there is a subdirectory per client in which one puts the retrievable files.

The script `secrets-rsync-init` does initial setup: invoking `useradd` to create the account secrets-rsync with home directory /usr/local/lib/secrets with mode `drwx------`, and attempts to set SELinux permissions appropriately.

The script `ingest-ssh-known-hosts -d` populates `~secrets-rsync/.ssh/authorized_keys` from the known_hosts file (or a single file given as a further argument); creates subdirectories in ~secrets for each client.  Modify that script to change the account name or home directory location.  Invoked without '-d', it has no effect but to print the translated known_hosts file on standard out.

To provide a secret file, e.g., `/root/.ssh/extra_root_key`,

* have forced command line as described above in `/usr/local/lib/secrets/.ssh/authorized_keys` for the cient
* place the secret file in the parallel location within `/usr/local/lib/secrets/client-host-name`, e.g., `/usr/local/lib/secrets/client-host-name/root/.ssh/extra_root_key`

Run the shell script `undo-secrets-rsync-init` to remove both the account, the home directory, **and thus all the secrets**.

The following scripts work in CentOS 7:

* `secrets-rsync-init`
* `ingest-ssh-known-hosts`
* `undo-secrets-rsync-init`



