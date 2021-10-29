---
layout: post
title:  "IT Automation with Ansible Vault and Kerberos"
date:   2021-10-29 00:00:00 -0400
categories: 
---

Based on Fedora 34. This post also assumes you have an Active Directory environment.

```
sudo dnf install ansible
sudo pip3 install --upgrade pip
sudo dnf install gcc python3-devel krb5-devel krb5-libs krb5-workstation
Install pywinrm with kerberos with pip3
pip3 install pywinrm
pip3 install pywinrm[kerberos]
```

Now point Kerberos to your domain.

```
$ sudo nano /etc/krb5.conf

# Final working version of krb5.conf for reference
includedir /etc/krb5.conf.d/
[logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log
[libdefaults]
    dns_lookup_realm = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false
    pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
    spake_preauth_groups = edwards25519
    default_realm = AVENTIS.LOCAL
    default_ccache_name = KEYRING:persistent:%{uid}
[realms]
 INTERNAL.LOCAL = {
    kdc = dc.internal.local
    admin_server = dc.internal.local
 }

[domain_realm]
 .internal.local = INTERNAL.LOCAL
 internal.local = INTERNAL.LOCAL
```
Verify that Kerberos authentication is working.

```
# Get a Keberos Ticket - AD Domain is in CAPITAL LETTER
$ kinit administrator@AVENTIS.LOCAL
Password for administrator@AVENTIS.LOCAL:

# List Kerberos Session
$ klist
Ticket cache: KCM:1000
Default principal: administrator@INTERNAL.LOCAL

Valid starting       Expires              Service principal
mm/dd/yyyy 00:00:00  mm/dd/yyyy 00:00:00  krbtgt/INTERNAL.LOCAL@INTERNAL.LOCAL
        renew until mm/dd/yyyy 00:00:0

# Destroy all Keberos connection 
$ kdestroy
```

Add the Remote Windows Host to the default ansible inventory file in /etc/ansible/hosts.

```
[winhost]
dc.internal.local

[winhost:vars]
ansible_user = administrator@INTERNAL.LOCAL
ansible_password = {{ansbible_password}}
ansible_connection = winrm
ansible_winrm_transport = kerberos
ansible_winrm_server_cert_validation = ignore
```

Enable HTTPS for WinRM using PowerShell and open port 5986

```
New-SelfSignedCertificate -DnsName "<YOUR_DNS_NAME>" -CertStoreLocation Cert:\LocalMachine\My
winrm create winrm/config/Listener?Address=*+Transport=HTTPS '@{Hostname="<YOUR_DNS_NAME>"; CertificateThumbprint="<COPIED_CERTIFICATE_THUMBPRINT>"}'
port=5986
netsh advfirewall firewall add rule name="Windows Remote Management (HTTPS-In)" dir=in action=allow protocol=TCP localport=$port
```

Secure password with Ansible Vault. Here a text file `win-vault.yml` is created and vaulted.

```
$ cat win-vault.yml
ansible_password: Password1
ansible-vault encrypt win-vault.yml
```

Now you can run a playbook with `ansible-playbook -e @win-vault.yml --ask-vault playbook.yml`
