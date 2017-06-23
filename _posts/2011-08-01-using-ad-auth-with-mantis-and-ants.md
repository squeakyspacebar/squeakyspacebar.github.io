---
layout: post
title: "Using Active Directory Authentication with Mantis Bug Tracker and Ants"
date: 2011-08-01
---

I've recently been working on getting ANTS (an applicant tracking system based
on the [Mantis Bug Tracker](https://www.mantisbt.org/) code) configured to use
AD/LDAP for authentication. I'm going to leave this tutorial here in case
anybody else who has been having problems with configuring Mantis/ANTS for LDAP
authentication comes upon this post - both of these applications (Mantis and
Ants) use the same authentication code.

The most important items are to have an account that you can run LDAP queries
through and to make sure that the application is configured for LDAP
authentication.

In your `config_inc.php` file (this file will override settings in
`config_defaults_inc.php`), you'll want to include the following configurations
(I added a few comments of my own):

```
# --- login method ----------------
 # CRYPT or PLAIN or MD5 or LDAP or BASIC_AUTH
 # You can simply change this at will. Mantis will try to figure out how the passwords were encrypted.
 $g_login_method                = LDAP;
#############################
 # Mantis LDAP Settings
 #############################
# look in README.LDAP for details
# --- using openldap -------------
 $g_ldap_server           = 'LDAP://domain.com';
 $g_ldap_port             = '389'; # 636 for LDAP over SSL
 $g_ldap_root_dn          = 'ou=Users,dc=domain,dc=com';
 $g_ldap_organization     = ''; # e.g. '(organizationname=*Traffic)'
 $g_ldap_uid_field        = 'sAMAccountName'; # Use 'sAMAccountName' for Active Directory
 $g_ldap_bind_dn          = 'johndoe'; # Username
 $g_ldap_bind_passwd      = 'opensesame'; # Password
 $g_use_ldap_email        = ON; # Should we send to the LDAP email address or what MySql tells us
# The LDAP Protocol Version, if 0, then the protocol version is not set.
 $g_ldap_protocol_version = 3;
```

One caveat is that the base Distinguished Name `$g_ldap_root_dn` needs to be
very specific. I personally had a few issues trying to query with an
incomplete base DN; when I attempted to authenticate with credentials I knew
were valid, I was still unable to log into the system.

In order to hunt down the full DN where the user you're searching for may be
located, run a LDAP query using this command (make sure `ldapsearch` is
installed first):

```
ldapsearch -H LDAP://domain.com -b "dc=domain,dc=com" -D "Users\johndoe" -w opensesame "(sAMAccountName=johndoe)"
```

The `-H` option is used to specify the URI of the LDAP server, `-b` is the base
DN for your search, `-D` is for the DN (username) you'll use to bind, `-w` is
used for your password (use `-W` if you don't want to enter your password into
the command itself), and the final portion is the query itself.

If your query was successful, look for the `dn` or `distinguishedName`
attributes to figure out where the username you want to search for is located.

Another caveat in the way LDAP authentication works with Mantis/ANTS is that a
local account has to be made that matches the LDAP account you want to
authenticate with.

For example, if you wanted to log into Mantis/ANTS using an LDAP account with
the `username` / `password` credentials of `johndoe` / `opensesame`, then you
must have a pre-existing account with the username `johndoe` and the password
`opensesame` already in the Mantis/ANTS database. This is probably no surprise.

This can be a problem when you either can't - or don't want to - import all the
relevant users under AD to your Mantis/ANTS database. A workaround is to make
the following edits to the beginning of the `auth_attempt_login()` function in
`authentication_api.php`:

```
function auth_attempt_login( $p_username, $p_password, $p_perm_login=false ) {
    $t_user_id = user_get_id_by_name( $p_username );

    $t_login_method = config_get( 'login_method' );

    if ( false === $t_user_id ) {
        if ( BASIC_AUTH == $t_login_method || LDAP == $t_login_method ) {
            if ( BASIC_AUTH == $t_login_method ) {
                # attempt to create the user if using BASIC_AUTH
                $t_cookie_string = user_create( $p_username, $p_password );
            } else if ( LDAP == $t_login_method) {
                $t_cookie_string = user_create( $p_username, $p_password,
                ldap_email_from_username( $p_username ) );
            }

            if ( false === $t_cookie_string ) {
                # it didn't work
                return false;
            }

            # ok, we created the user, get the row again
            $t_user_id = user_get_id_by_name( $p_username );

            if ( false === $t_user_id ) {
                # uh oh, something must be really wrong
                # @@@ trigger an error here?
                return false;
            }
        } else {
            return false;
        }
    }
}
```

What this does is create a user in Mantis/ANTS upon login **attempt** if the
user doesn't exist yet.

Please note that this particular edit relies on a proper e-mail attribute
being returned from the LDAP query - if there is no valid e-mail being returned,
an error will be generated when attempting to log in with a currently
non-existent account (that is, non-existent on Mantis/ANTS).

There is also the potential problem of cluttering up your database with bogus
accounts.

It's not the most elegant solution, and there are improvements that can be made
in the code, but this should be enough to get you started.

And that's it! You may run into some configuration issues particular to your
scenario, but what I posted here should work.
