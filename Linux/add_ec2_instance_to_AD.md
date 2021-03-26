# How to add a Linux instance to Active Directory?

## Issue

- use `id` search AD user returns nothing.
- how to join a Windows Active Directory using `samba` and `sssd`?

## Environment

- Red Hat Enterprise Linux 7
- CentOS 7
- Amazon Linux 2
- Windows Active Directory 2016 with domain name example.com

## Solution

1. Install packages.

    ```bash
    sudo yum install samba samba-common sssd adcli krb5-workstation oddjob oddjob-mkhomedir authconfig
    ```

2. Setup samba /etc/samba/smb.conf.

    ```bash
    [global]
        workgroup = EXAMPLE
        client signing = yes
        client use spnego = yes
        kerberos method = secrets and keytab
        log file = /var/log/samba/%m.log
        realm = EXAMPLE.COM
        security = ads
    ```

3. Setup sssd /etc/sssd/sssd.conf.

    ```bash
    [domain/example.com]
    debug_level = 9
    id_provider = ad
    access_provider = ad
    auth_provider = ad
    ad_domain = example.com
    krb5_realm = EXAMPLE.COM
    krb5_store_password_if_offline = True
    cache_credentials = True
    default_shell = /bin/bash
    ldap_id_mapping = True
    use_fully_qualified_names = True
    override_gid = 513
    fallback_homedir = /home/%u@%d
    default_shell = /bin/bash
    dyndns_update = false
    ldap_idmap_range_min = 100000
    ldap_use_tokengroups = False

    [sssd]
    services = nss, pam
    config_file_version = 2
    domains = example.com

    [nss]

    [pam]
    ```

4. Setup krb5 /etc/krb5.conf.

    ```/bash
    [logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log

    [libdefaults]
    default_realm = EXAMPLE.COM
    dns_lookup_realm = true
    dns_lookup_kdc = true
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns=false

    [realms]
    EXAMPLE.COM = {
    }

    [domain_realm]
    .example.com = EXAMPLE.COM
    example.com = EXAMPLE.COM
    ```

5. Update `/etc/resolv.conf` with AD DNS server.

6. Join instance to domain.

    ```bash
    adcli join example.com
    ```

7. Update system configuration to use AD login.

    ```bash
    chkconfig oddjobd on
    service oddjobd start
    authconfig --enablesssd --enablesssdauth --update
    authconfig --enablemkhomedir --update
    ```

## Testing

1. Check domain user information.

    ```bash
    [root@ip-172-31-12-62 sssd]# id EXAMPLE\\terry
    uid=1653501004(terry@example.com) gid=513 groups=513,1653500513(domain users@example.com)
    [root@ip-172-31-12-62 sssd]# id terry@example.com
    uid=1653501004(terry@example.com) gid=513 groups=513,1653500513(domain users@example.com)
    ```
