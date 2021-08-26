# SUTDiscourse

The platform for SUTD's community discussion. Free, open, simple.

<a href="https://observatory.mozilla.org/analyze/sutdiscourse.org"><img alt="Mozilla HTTP Observatory Grade" src="https://img.shields.io/mozilla-observatory/grade-score/sutdiscourse.org?publish&style=for-the-badge"></a>

## Introduction

This README will document the instructions necessary to setup the SUTD Discourse Forum from a sysadmin perspective or point of view, as well as list off the plugins that we are using in the forum. Meanwhile, the repository will host the files used for the custom SUTD theme style used for the forum.

## Table of Contents <a name="top"></a>

- [Discourse Setup](#setup)
  * [Step 1: Initial Resource Establishment](#step-1)
  * [Step 2: Service Inter-Connection](#step-2)
  * [Step 3: Initial Server Configuration](#step-3)
  * [Step 4: Install Discourse](#step-4)
  * [Step 5: Maintenance](#step-5)
  * [Step 6: Add Extra Features & Plugins](#step-6)
- [Forum Theme Installation](#theme)
- [Acknowledgements](#acknowledgements)


## Discourse Setup <a name="setup"></a>

[go to top](#top)

For now, we will be using Namecheap, DigitalOcean and Mailgun. Once we have gone official, we will be migrating to the Microsoft Outlook MX Mail Server officially utilized by SUTD so as to smoothly support and integrate into the SUTD emailing system and its current Active Directory. We will also consider migrating to `discourse.sutd.edu.sg` as the official subdomain and utilizing an internal SUTD server to provide hosting services, thereby slowly migrating away from external service providers such as Namecheap and DigitalOcean over time. The usage of Let's Encrypt would be reconsidered in the future, but it is likely to stay (unless we migrate from DV to OV or even EV certificates). The system environment setup (such as using Ubuntu 18.04.3 LTS x64 and Docker) should be similar between our current setup and the eventual official setup. Any instructions for migration and system update/upgrade would be documented here as well.

> NOTE: We would need to switch to-and-fro between Namecheap, DigitalOcean and Mailgun as we need relevant information from one another to be inputted to each other's settings panels for a complete setup.

### Step 1: Initial Resource Establishment <a name="step-1"></a>

[go to top](#top)

> Ensure that you have access to your primary email that you will be using throughout this setup process, your registered mobile phone for any required authentication and verification processes, as well as your respective credentials of your selected payment card option.

The resources that we would need are:

- Domain Name (Namecheap)
- Hosting (DigitalOcean)
- Mail Server (Mailgun)

#### Namecheap

Firstly, obtain a domain name from Namecheap. Enable `WhoisGuard` (it's free) with a 3-days refresh rate and you could optionally enable Namecheap `PremiumDNS` protection (for a fee). Verify your email credentials so that the domain is activated. Auto-renewal is optional. Enable `DNSSEC` under the `Advanced DNS` section (we use Namecheap's nameservers since DigitalOcean does not support `DNSSEC` at the moment). For subsequent steps, you could either use the root domain that you have obtained or a `discourse` subdomain (for the latter, you would then need to manage it in the Advanced DNS Host Records section).

#### DigitalOcean

> While DigitalOcean does not provide a robust protection against DDoS attacks at the Droplet level (read more about it [here](https://www.digitalocean.com/community/questions/does-digitalocean-have-an-anti-ddos-protection?answer=32485)), Cloudflare CDN and Discourse have not historically played well together (read more about it [here](https://meta.discourse.org/t/enable-a-cdn-for-your-discourse/14857) and [here](https://meta.discourse.org/t/setting-up-https-support-with-lets-encrypt/40709)). Currently, there are no Caddy module implementations [here](https://github.com/caddy-dns/) for Fastly support. We are also not using Caddy since the default `discourse-setup` now already provides auto-renewed SSL (through a daily cron job) + HTTP2 + QUIC out of the box. If we were to use a CDN in the future for mitigation against DDoS attacks, we would elaborate more here (might need to use the corresponding API keys or scoped tokens for HTTPS support with CDN). For future reference, refer to [here](https://meta.discourse.org/t/running-discourse-with-caddy-server/54716) or [here](https://github.com/caddyserver/examples/tree/master/discourse) to setup Caddy for Discourse. The latter requires manual installation of [Go](https://www.digitalocean.com/community/tutorials/how-to-install-go-and-set-up-a-local-programming-environment-on-ubuntu-18-04) (latest version) and [Caddy](https://www.digitalocean.com/community/tutorials/how-to-host-a-website-with-caddy-on-ubuntu-18-04) on the Droplet to facilitate the execution of Caddy without a container.

Create a DigitalOcean `Droplet` (Cloud Virtual Machine). You could park it under a project named `SUTDiscourse`. These are the settings for the `Droplet`:

- Image: Ubuntu 18.04.3 (LTS) x64
- Sufficient CPU Plan (see recommended minimum hardware requirements [here](https://github.com/discourse/discourse/blob/master/docs/INSTALL.md#hardware-requirements))
- Datacenter Region: Singapore
- VPC Network: `default-sgp1`
- IPv6 Enabled
- Monitoring Enabled
- SSH Key Authentication Method
- 1 Droplet (Hostname: `sutdiscourse` or your respective domain/subdomain name)
- OPTIONAL: Enable backups (recommended)

After that, create a DigitalOcean Cloud Firewall (`Manage` > `Networking` > `Firewalls` > `Create Firewall`) with these settings:

- An appropriate name of your choice (`sutdiscourse-rules`, `sutdiscourse-firewall`, etc.)

- Inbound Rules:
  | Type | Protocol | Port Range | Sources |
  | --- | --- | --- | --- |
  | SSH | TCP | 22 | `All IPv4` `All IPv6` |
  | HTTP | TCP | 80 | `All IPv4` `All IPv6` |
  | HTTPS | TCP | 443 | `All IPv4` `All IPv6` |
  | Custom (for Git) | TCP | 9418 | `All IPv4` `All IPv6` |
  
- Outbound Rules:
  | Type | Protocol | Port Range | Destinations |
  | --- | --- | --- | --- |
  | ICMP | ICMP |  | `All IPv4` `All IPv6` |
  | All TCP | TCP | All ports | `All IPv4` `All IPv6` |
  | All UDP | UDP | All ports | `All IPv4` `All IPv6` |
  
- Apply to the `sutdiscourse` Droplet (or your corresponding hostname) **ONLY after you are done with Step 4 (after rebuilding the container).**

> The specified inbound firewall rules would lead to no response for any `ping` commands to the DigitalOcean Droplet (no ICMP traffic allowed). No need to panic if there are no responses! As long as you can SSH into the Droplet, it is working properly.

#### Mailgun

Register for a free trial account (Flex Tier Plan) and activate the account by verifying through your email and mobile phone. After that, go to `Sending` > `Domains` and click `Add New Domain`. Use a `mg` subdomain for the domain name (`mg.sutdiscourse.org`). Either `US` or `EU` is fine (we chose `EU` since Europe is closer to Singapore). Enable the `Create DKIM Authority` checkbox and select `2048`. Turn on `Click tracking`, `Open tracking` and `Unsubscribes` under `Sending` > `Domain settings` > `Domain settings` > `Tracking`.

### Step 2: Service Inter-Connection <a name="step-2"></a>

[go to top](#top)

> Take note that we prefer to have our SSL Certificates from Let's Encrypt instead of from Comodo, GoDaddy or Symantec, in the light of certain security breaches that shall not be named. We also prefer SSL Certificates from Let's Encrypt instead of Cloudflare due to the implementations of DNS hijacking, shared SSL Certificates and no end-to-end encryption by Cloudflare (Free Plan).

#### Namecheap-DigitalOcean

We do not need to modify the nameservers, so leave those settings to their default options/values.

In Namecheap > `Domain List`, select your domain and click `Manage`. Under the `Advanced DNS` > `Host Records` section, add 2 new records with the following details:

| Type | Host | Value | TTL |
| --- | --- | --- | --- |
| A Record | `@` | `<DigitalOcean Droplet's IPv4 Address>` | 60 min |
| A Record | `www` | `<DigitalOcean Droplet's IPv4 Address>` | 60 min |

> If you are pointing the `discourse` subdomain to the Droplet instead of the main root domain, use `discourse` and `www.discourse` as the values under `Host` respectively.

To improve the security of the issued SSL certificates, we can restrict the certificate issuance. Since we are using Let's Encrypt, add another record with the following details:

| Type       | Host | Flag | Tag     | Value (CAA Identifying Domain) | TTL    |
| ---------- | ---- | ---- | ------- | ------------------------------ | ------ |
| CAA Record | `@`  | `0`  | `issue` | `letsencrypt.org`              | 60 min |

#### Namecheap-Mailgun

Check your specific records in Mailgun under `Sending` > `Domain settings`.

In Namecheap > `Domain List`, select your domain and click `Manage`. Under the `Advanced DNS` > `Host Records` section, add 3 new records with the following details:

| Type | Host | Value | TTL |
| --- | --- | --- | --- |
| TXT Record | `mg` | `v=spf1 include:eu.mailgun.org ~all` | 60 min |
| TXT Record | `pic._domainkey.mg` | `k=rsa; p=...` | 60 min |
| CNAME Record | `email.mg` | `eu.mailgun.org` | Automatic |

The first entry is for SPF, while the second entry is for DKIM. Both are required for our Discourse forum. The `CNAME Record` is optional for tracking purposes.

In Namecheap > `Domain List`, select your domain and click `Manage`. Under the `Advanced DNS` > `Mail Settings` section, choose `Custom MX` and add 2 new records with the following details:

| Type | Host | Mail Server | Priority | TTL |
| --- | --- | --- | --- | --- |
| MX Record | `mg` | `mxa.eu.mailgun.org` | `10` | Automatic |
| MX Record | `mg` | `mxb.eu.mailgun.org` | `10` | Automatic |

> If you chose `US`, replace any `eu.mailgun.org` with `mailgun.org`.

After entering the records for the first time, you should verify the DNS records in Mailgun to ensure that your entries are correct by going to `Sending` > `Domain settings` > `DNS records` and clicking `Verify DNS Settings` (if first-time) or `Check DNS Records Now` (subsequent attempts).

### Step 3: Initial Server Configuration <a name="step-3"></a>

[go to top](#top)

Access your DigitalOcean's cloud server via SSH. Follow the instructions [here](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04) first, without setting up `UFW` (since we are already using DigitalOcean Cloud Firewall).

> Reasoning behind our choice of DigitalOcean Cloud Firewall instead of `UFW` is explained [here](https://www.digitalocean.com/community/questions/disabling-ufw-in-favour-of-do-cloud-firewall?comment=70283). Please only choose one to avoid redundancy or even conflicting firewall rules.

Remember to add a 2 GB swapfile by following the instructions [here](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-18-04) (in Step 3, change `1G` to `2G`). Do not worry too much about the SSD warning since this swap space will only serve as an [_"insurance policy"_](https://meta.discourse.org/t/create-a-swapfile-for-your-linux-server/13880) during heavy work load times.

After that, follow the instructions [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04) (only Step 1) to install Docker.

### Step 4: Install Discourse <a name="step-4"></a>

[go to top](#top)

Follow the instructions [here](https://github.com/discourse/discourse/blob/master/docs/INSTALL-cloud.md) to clone the official Discourse Docker image, launch the `discourse-setup` tool and initiate bootstrapping to build the Discourse forum. Use your SMTP credentials that you have obtained from Mailgun.

Do follow some of the post-install maintenance instructions such as running these commands:

```console
# Enable automatic updates
$ sudo dpkg-reconfigure -plow unattended-upgrades

# Enforce strong VM user password (modify /etc/pam.d/common-password accordingly)
$ sudo apt install libpam-cracklib

# Install and enable fail2ban (no conflict with iptables)
$ sudo apt install fail2ban
$ sudo service fail2ban start

# Tell Linux to relax and perform the Redis fork in a more optimistic allocation fashion
$ echo vm.overcommit_memory=1 | tee -a /etc/sysctl.conf
```

If you would like to redirect the `www` subdomain to the root domain directory, set up Let's Encrypt for the `www` subdomain by following the instructions [here](https://meta.discourse.org/t/setting-up-let-s-encrypt-with-multiple-domains/56685) and add the redirection rules by following the instructions [here](https://meta.discourse.org/t/redirect-single-multiple-domain-s-to-your-discourse-instance/18492).

Reboot the Droplet after running the aforementioned commands and rebuild the Discourse forum to restart the Docker container.

Remember to activate and apply the DigitalOcean Cloud Firewall to the Droplet after rebuilding the forum.

### Step 5: Maintenance <a name="step-5"></a>

[go to top](#top)

For regular maintenance, whereby Discourse would need to be regularly updated via the `/admin/upgrade` subcategory directory on the website, we would also need to routinely run these commands to reclaim enough space so as to ensure that there would be minimal lag and prevent any `502 Bad Gateway` timeout errors:

```console
# Elevate your permissions as super user (root@sutdiscourse)
$ sudo -s

# Navigate to the SUTDiscourse installation folder
$ cd /var/discourse

# Fetch and download the latest version of the Discourse launcher
$ git pull

# Rebuild the app
$ ./launcher rebuild app

# Start the pruning process only AFTER the app is rebuilt and relaunched (with its current status being turned on and running)
$ yes | ./launcher cleanup
```

This is to remove any stopped Docker containers, delete any build cache, expunge any unused networks (if any) and prune any old dangling Docker images which might slowly pile up in the server for some reason.

A user-level cron(tab) job (on the `root` user) could also be set up to routinely conduct and execute this maintenance process at a specific time interval:

```console
$ sudo apt update
$ sudo apt install cron
$ sudo systemctl enable cron
$ sudo systemctl start cron
$ crontab -e
```

Add this line to the `/var/spool/cron/crontabs/root` file using your preferred selected file/text editor (with the syntax `@monthly` being a replacement shorthand for `0 0 1 * *`):

```nano
@monthly cd /var/discourse && ./launcher rebuild app && (yes | ./launcher cleanup)
```

> Note that not all `cron` daemons can parse this relatively new shortcut syntax of `@monthly` (particularly older versions) so double-check that it works and that `crontab` does not throw any errors before you rely on it and saving the job file. Alternatively, you could also specify a different custom time interval, instead of `@monthly`. Also, a reminder that you do not need to specify the user account to be used to execute the commands if we are using the `crontab -e` method (instead of directly editing the system-wide `/etc/crontab` file) since all commands will be run as the owner of the file.

### Step 6: Add Extra Features & Plugins <a name="step-6"></a>

[go to top](#top)

We will list down the features that we enabled and plugins that we are using here!

> Every time before rebuilding the Discourse forum after adding a `git clone` plugin entry in the `app.yml` file, ensure that you temporarily disable the DigitalOcean Cloud Firewall rules. Else, it will fail to properly rebuild and restart the app.

#### Features

More content coming soon!

#### Plugins

- https://github.com/discourse/docker_manager.git (default built-in)
- https://github.com/discourse/discourse-spoiler-alert.git
- https://github.com/discourse/discourse-solved.git
- https://github.com/discourse/discourse-data-explorer.git
- https://github.com/discourse/discourse-calendar.git
- https://github.com/discourse/discourse-cakeday.git
- https://github.com/discourse/discourse-assign.git
- https://github.com/discourse/discourse-math.git

To add a plugin, simply add the plugin's repo URL to your container's `app.yml` file (`exec` section under the `after_code` hook), and then re-build the container.

## Forum Theme Installation <a name="theme"></a>

[go to top](#top)

More instructions coming soon!

## Acknowledgements <a name="acknowledgements"></a>

[go to top](#top)

Credits to the OpenSUTD team, the DiscoverSUTD 2020 team and the relevant respective SUTD Offices in supporting this special project.
