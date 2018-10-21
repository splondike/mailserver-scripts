**I no longer use this setup.** After a few years of running it successfully I found that my mail had started to consistently end up in people's Junk folders despite me having valid SPF, DKIM, and DMARC records. After some investigation using http://dkimvalidator.com/ it looks like the problem was I didn't have any kind of SMTP server reputation (e.g. at http://www.dnswl.org/). If you want to run your own mailserver, in addition to the steps below, you should test with the major email providers, and probably try and get yourself a positive reputation (e.g. by registering at dnswl).

---


These instructions and scripts will set you up with a functioning mail server based on Debian 8.

## Preparation

First you will need somewhere to host the server. I used vultr.com.

How you install the server is up to you. You should set a reverse PTR address somehow though.

I also locked down the server a bit as described in this [article](http://plusbryan.com/my-first-5-minutes-on-a-server-or-essential-security-for-linux-servers). For your firewall, you need to allow connections to ports 465, 25, 993, and your SSH port.

The end result of this is you will end up with an ipv4 address for your server. Also ensure you have SSH set up, and full access sudo for the SSH user (behind the user password).

## Installation and configuration with Ansible

This repository contains some [Ansible](http://www.ansible.com/home) scripts which should set up your server if all goes well.

First edit ansible.yml and set the mail\_domain variable to what you want to use. If you wanted addresses like bob@example.com, you'd set it to example.com.

Second create an inventory file with content like this:

   default ansible\_ssh\_host=<your host ip> ansible\_ssh\_user=<your user> ansible\_sudo=True

Lastly run ansible using that inventory:

   ansible-playbook --ask-sudo-pass -i inventory ansible.yml

That should run through setting everything up. Any errors will be shown in your terminal. The ansible scripts should be safe to run multiple times if you need to do any adjustments.

## Adding mailboxes and aliases

Now we're going to add your user mailboxes and any aliases (forwarders from one address to another).

Shell in to the server and run the following to generate your password hash (which is the bit after the {SHA512-CRYPT} part):

   doveadm pw -s SHA512-CRYPT

Next, switch to root and run

   mysql mailserver

Then in the MySQL prompt run the following (the domain\_id foreign key row has been inserted already by Ansible):

   INSERT INTO `mailserver`.`virtual_users` (`domain_id`, `password` , `email`) VALUES ('1', '$6$YOURPASSWORDHASH', '<your name>@<your mail domain>');

For some reason, you also need a postmaster alias in your table:

   INSERT INTO `mailserver`.`virtual_aliases` (`domain_id`, `source` , `destination`) VALUES ('1', 'postmaster', '<your name>@<your mail domain>');

If you have any additional aliases you would like to add, do that like this:

   INSERT INTO `mailserver`.`virtual_aliases` (`domain_id`, `source` , `destination`) VALUES ('1', '<your alias>@<your mail domain>', '<some email address>');

## Setting up DNS

The final step is to set up your DNS records: MX, SPF, DKIM, and DMARC.

First your MX records. You'll need to add one pointing to the A record for your mailserver. I've set priority 5 here:

   <your mail domain> MX 5 <name for your mailserver, e.g. mail.example.com>

Second SPF. If you're only going to send mail from this mail server, then you just need a TXT record like this:

   <your mail domain> TXT v=spf1 mx -all

Third DKIM. Switch to root on the mailserver and run `cat /etc/opendkim/mail.txt`. Put the contents of this in a TXT record under mail._domainkey.<your mail domain>, and also _domainkey.<your mail domain> (2 identical records):

   mail._domainkey.<your mail domain> TXT  v=DKIM1; h=sha256; k=rsa; s=email; p=<some specific stuff>
   _domainkey.<your mail domain> TXT  v=DKIM1; h=sha256; k=rsa; s=email; p=<some specific stuff>

Lastly DMARC. These entries state how email claiming to be from your domain which fails SPF or DKIM validation should be handled. These rules say to send aggregate failure reports to the given email, and to reject mail which fails a check (also for subdomains):

   _dmarc.<your mail domain> TXT v=DMARC1; p=reject; rua=mailto:<something@your domain>; fo=1; sp=reject

## Connecting with a mail client

You can connect to your server using IMAP and SMTP over TLS/SSL. Your port for IMAP will be 993, and 465 for SMTP. The username you use is the fully qualified email address, e.g. bob@example.com.

## Validating your email

Finally, you'll want to verify that your email doesn't look like spam. This means checking that SPF, DKIM, and reverse PTR are valid. There are a few ways to do this:

1. http://dkimvalidator.com/
2. Send an email to a gmail.com address. It will say whether SPF and DKIM pass in the headers.
