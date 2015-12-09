# self-hosted-mailserver
A set of ansible scripts, to set up fully functional, self-hosted mailserver.

This set of scripts was based on those excellent articles:
* [My First 5 Minutes On A Server; Or, Essential Security for Linux Servers](http://plusbryan.com/my-first-5-minutes-on-a-server-or-essential-security-for-linux-servers) ([copy](docs/my-first-5-minutes-on-a-server-or-essential-security-for-linux-servers.md))
* [NSA-proof your e-mail in 2 hours](http://sealedabstract.com/code/nsa-proof-your-e-mail-in-2-hours/) ([copy](docs/nsa-proof-your-e-mail-in-2-hours.md))
* [Strong Ciphers for Apache, nginx and Lighttpd](https://cipherli.st/)
* [How To Use Duplicity with GPG to Securely Automate Backups](https://www.digitalocean.com/community/tutorials/how-to-use-duplicity-with-gpg-to-securely-automate-backups-on-ubuntu) ([copy](docs/how-to-use-duplicity-with-gpg-to-securely-automate-backups-on-ubuntu.md))
* [Guide to Deploying Diffie-Hellman for TLS](https://weakdh.org/sysadmin.html) ([copy](docs/guide-to-deploying-diffie-hellman-for-tls.md))
* [Secure Secure Shell](https://stribika.github.io/2015/01/04/secure-secure-shell.html) ([copy](docs/secure-secure-shell.md))

with some exceptions:

* no EncFS - because it's pointless
* no full text search - beacuse it's JAVA
* added Roundcube - because you might need your email the less you expect it
* added ddclient - since mobile hardware needs mobile support

Those rules were written with Debian in mind, and were tested in 7.0 (Wheezy) and 8.0 (Jessie). They should also
work on Ubuntu, but I didn't try it and [you shouldn't too](https://gnu.org/philosophy/ubuntu-spyware.html).
Also, by default E-mail server disables SSLv2, SSLv3, TLSv1 and TLSv1.1. This can cause problems with Apple Mail,
but if you are concerned in privacy, this shouldn't be a big issue - since you should not use it anyway...

## Install

First of all you need [ansible](http://www.ansible.com/home)

    brew/apt-get/yum/whatever install ansible

Also, you need an inventory file, and configure all the hosts you wish to install your mailserver to:

    cat << EOF > ansible-inventory
    [local]
    123.234.345.456 ansible_ssh_user=root
    some.other.host.com ansible_ssh_user=root
    EOF

See [ansible documentation](http://docs.ansible.com/intro_inventory.html) for more details. If you wish to use
`ansible_ssh_user=root` like in example above, don't forget to put your public key to `/root/.ssh/authorized_keys` on
the remotes.

Next, you need to configure your new system(s) variables:

    for role in base duplicity maildb security; do vi roles/${role}/vars/main.yml; done

and add your SSL certificates to `roles/ssl/templates/certs-this-machine.pem` and `roles/ssl/templates/private-this-machine.pem`.
I strongly recommed to use trusted certificate (for example from [here](https://www.startssl.com/?app=1)), but you can
also create a self-signed one. Last thing you need to configure is `roles/ddclient/templates/ddclient.conf.j2` for
`ddclient` (or disable it in `main.yml` if you won't need it).

Also keep in mind, that those scripts would create many 4096bit primes, and would take a while (especially security and SSL roles). If you wish to sped them up (not recommended), set `security.strong_primes` flag in `roles/security/vars/main.yml` to `false`. Another option, is disabling it and generating those primes on stronger machine (this is very relevant, if you plan to use those scripts on ARM machine like Raspberry PI). To see which commands are invoked, type:

    cat roles/*/tasks/main.yml | grep -B1 'when: security.strong_primes' | grep shell

Adapt them as necessary, and then copy generated files to proper locations on remote machine (after ansible script finished its magic).

And you're ready to rock!

    ansible-playbook -s -i ansible-inventory main.yml

Have fun, and don't forget to set your TXT and PTR records for DKIM and SPF!

