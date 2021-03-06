---
title: "Howto: SSSD"
---

SSSD Is the system security services daemon. It is the bridge between a unix system and resolving users through LDAP.

# Configure SSSD
----------------

**NOTE: We strongly advise you have (configured TLS)[howto-ssl.html] on your LDAP server first**

SSSD has a concept of domains and provides. Here is an example configuration that can be altered and should work with 389-ds-base.

    [sssd]
    services = nss, pam, ssh, sudo
    config_file_version = 2
    domains = default

    [nss]
    homedir_substring = /home

    [domain/default]
    # If you have large groups (IE 50+ members), you should set this to True
    ignore_group_members = False
    debug_level=3
    cache_credentials = True
    id_provider = ldap
    auth_provider = ldap
    access_provider = ldap
    chpass_provider = ldap
    ldap_schema = rfc2307bis
    ldap_search_base = dc=example,dc=com
    # We strongly recommend ldaps here.
    ldap_uri = ldaps://ldap.example.com
    ldap_tls_reqcert = demand
    ldap_tls_cacert = /etc/openldap/ldap.crt
    ldap_access_filter = (|(memberof=cn=<login group>,ou=Groups,dc=example,dc=com))
    enumerate = false
    access_provider = ldap
    ldap_user_member_of = memberof
    ldap_user_gecos = cn
    ldap_user_uuid = nsUniqueId
    ldap_group_uuid = nsUniqueId
    # This is really important as it allows SSSD to respect nsAccountLock
    ldap_account_expire_policy = rhds
    ldap_access_order = filter, expire

    # Setup for ssh keys
    # Inside /etc/ssh/sshd_config add the lines:
    #   AuthorizedKeysCommand /usr/bin/sss_ssh_authorizedkeys
    #   AuthorizedKeysCommandUser nobody
    # You can test with the command: sss_ssh_authorizedkeys <username>
    # The objectClass: nsAccount holds this attribute.
    ldap_user_ssh_public_key = nsSshPublicKey

Enable SSSD to start with systemctl

    systemctl enable sssd

# Configure NSS (SUSE)
----------------------

Ensure the following lines are present in /etc/nsswitch.conf

    passwd: compat sss
    group:  compat sss
    shadow: compat sss

You should now be able to resolve a user with "getent password <name>" from ldap.

# Configure NSS (Fedora/RH)
---------------------------

Ensure the following lines are present in /etc/nsswitch.conf

    passwd:     files sss
    shadow:     files sss
    group:      files sss
    netgroup:   files sss
    sudoers: files sss

You should now be able to resolve a user with "getent password <name>" from ldap.

# Configure PAM (SUSE)
----------------------

Suse uses 4 "common" files that are included by all other services for authentication and authorisation.
These files are:

    common-account-pc
    common-auth-pc
    common-password-pc
    common-session-pc

Their content should appear as:

    # common-account-pc
    account    requisite   pam_unix.so
    account    sufficient  pam_localuser.so
    account    required    pam_sss.so use_first_pass

    # common-auth-pc
    auth        required      pam_env.so
    auth        [default=1 ignore=ignore success=ok] pam_localuser.so
    auth        sufficient    pam_unix.so nullok try_first_pass
    auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success
    auth        sufficient    pam_sss.so prompt_always ignore_unknown_user
    auth        optional      pam_gnome_keyring.so
    auth        required      pam_deny.so

    # common-password-pc
    password    requisite   pam_cracklib.so
    password    [default=1 ignore=ignore success=ok] pam_localuser.so
    password    required    pam_unix.so use_authtok nullok shadow try_first_pass
    password    sufficient  pam_sss.so use_authtok
    password    optional    pam_gnome_keyring.so    use_authtok

    # common-session-pc
    session optional    pam_systemd.so
    session required    pam_limits.so
    session required    pam_mkhomedir.so skel=/etc/skel/ umask=0022
    session required    pam_unix.so try_first_pass
    session optional    pam_sss.so
    session optional    pam_umask.so
    session optional    pam_gnome_keyring.so    auto_start only_if=gdm,gdm-password,lxdm,lightdm,mdm
    session optional    pam_env.so

You should be able to login as local *and* sssd users who match your ldap_access_filter now. It's a good
idea to test that invalid passwords, empty passwords, and users who do not match the access filter
are all correctly rejected.

# Configure PAM (Fedora/RH)
---------------------------

**WARNING: Altering pam may lock you out of your system. Always maintain a second sudo/root shell while altering these files, and keep backups**

Consult your distriubtion documenation for this. Generally you want to add:

    auth        sufficient    pam_sss.so use_first_pass
    account     [default=bad success=ok user_unknown=ignore] pam_sss.so
    password    sufficient    pam_sss.so use_authtok
    session     optional      pam_sss.so

For Fedora/Centos, you should alter /etc/pam.d/system-auth-ac and /etc/pam.d/password-auth-ac to match the following:

    auth        required      pam_env.so
    auth        sufficient    pam_unix.so try_first_pass
    auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success
    auth        sufficient    pam_sss.so use_first_pass
    auth        required      pam_deny.so

    account     required      pam_unix.so
    account     sufficient    pam_localuser.so
    account     sufficient    pam_succeed_if.so uid < 1000 quiet
    account     [default=bad success=ok user_unknown=ignore] pam_sss.so
    account     required      pam_permit.so

    password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
    password    sufficient    pam_unix.so sha512 shadow try_first_pass use_authtok
    password    sufficient    pam_sss.so use_authtok
    password    required      pam_deny.so

    session     optional      pam_keyinit.so revoke
    session     required      pam_limits.so
    -session    optional      pam_systemd.so
    session     optional      pam_oddjob_mkhomedir.so umask=0077
    session     [success=1 default=ignore] pam_succeed_if.so service in crond quiet use_uid
    session     required      pam_unix.so
    session     optional      pam_sss.so




