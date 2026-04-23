# Dovecot Configuration Examples

The following examples are provided for MySQL + Dovecot **2.4.1+**

You may need to adapt them if you are using different database engine (postgres/sqlite).

## Dovecot supported features

- Has been tested on Dovecot 2.4.3
- Check dovecot [config compatibility](https://doc.dovecot.org/latest/installation/upgrade/2.4-to-2.4.x.html) and `dovecot_config_version` config option when upgrading.

## Changes for Dovecot 2.4

- **IMPORTANT! Breaking changes upgrading to Dovecot 2.4+**
   - **Short variable names** have been deprecated. For example, `%u` no longer works and must be replaced with the long name `%{user}`
   - **Modifiers** have been made more consistent. For example, `%Lu` must be replaced with the new filter format: `%{user | lower }`
   - **Database queries** used to be configured in separate files. Queries are now configured inline with the auth config.
   - See the [Dovecot Upgrading notes](https://doc.dovecot.org/latest/installation/upgrade/2.3-to-2.4.html) (Also to refer to other upgrading notes if you are upgrading from older (< 2.3) versions of Dovecot.)
   - This is good: It makes the configuration more consistent and easier to understand. But you need to carefully review and update your existing config files to have a working Dovecot 2.4 installation.
   - See: [This page](https://doc.dovecot.org/2.4.3/installation/upgrade/2.3-to-2.4.html) for a list of most of the variable changes between 2.3 and 2.4. Dovecot provides an [Upgrader](https://dovecot.org/upgrader/) tool to help update configs. (It's not perfect, but can help the most common changes. You will still need to make a few changes yourself (e.g. SQL queries))
   - My approach to upgrading was to start with a fresh 2.4 install and compare the config files side-by-side. Many of the old config files were no longer needed or are the defaults anyway.

## Dovecot configuration

Dovecot's config files are usually somewhere like `/etc/dovecot`, and includes several other files from `dovecot.conf` under `/etc/dovecot/conf.d`

Authentication configs are usually included from `./conf.d/10-auth.conf`.

1. Make a copy of `auth-sql.conf.ext` to a new file, e.g. `auth-mysite.conf.ext` enable this in `./conf.d/10-auth.conf`:

```
!include auth-mysite.conf.ext

#!include auth-deny.conf.ext
#!include auth-master.conf.ext
#!include auth-oauth2.conf.ext
```

**Alternatively, add the relevant `passdb` / `userdb` sections (examples below) to your existing `auth-` config.**

2. Dovecot auth configuration

After changing the configuration, run `doveadm reload`

## WARNING!

> **ALWAYS TEST** the configuration does what you expect!

Dovecot is amazing, but its authentication config can get awkward and somewhat **hard to understand** when using multiple passdbs.

It is possible to accidentally configure, for example, a passdb that allows ANY password, or for dovecot to
accept mail for non-existent users. It's important to test wrong passwords, valid/invalid usernames, blank passwords etc.

Dovecot's authentication documentation is scattered over many pages, lacks explanation in places, and there are few _good_ 
example configurations. Some of the examples (in the Dovecot docs or elsewhere) are incomplete or wrong.

> This document provides useful examples, but we cannot provide for every possible configuration and environment. You need to **ADAPT THESE EXAMPLES** for your particular requirements. Please DO NOT just copy+paste these examples without thought.

If unsure, refer to the Dovecot documentation (Dovecot 2.4.3):

- [Password databases (passdb)](https://doc.dovecot.org/2.4.3/core/config/auth/passdb.html#password-databases-passdb) / [Password database extra fields](https://doc.dovecot.org/2.4.4/core/config/auth/passdb.html#user-extra-fields)
- [User databases (userdb)](https://doc.dovecot.org/2.4.3/core/config/auth/userdb.html) / [User database extra fields](https://doc.dovecot.org/2.4.3/core/config/auth/userdb.html#extra-fields)
- [Multiple authentication databases](https://doc.dovecot.org/2.4.3/core/config/auth/mutltiple.html)

> **The order in which passdb / userdb entries appear in the configuration is significant!** 


## Username formats

Set `ap4rc_username_format` in ap4rc's config.inc.php to use one of the username formats. 

Example dovecot configurations for each username format:

## Format 1 (Default)

Format: `"username@application"` (or: `"username@example.com@application`")

TODO

(I don't use this format, as I found it quite cumbersome. I recommend using format 2)

## Format 2: Username

This goes in your auth config e.g. `auth-mysite.conf.ext`.

Format: `"username"` / `"user@example.com"` (use same username everywhere)

This format allows users to use the same username everywhere (If the username is an email address,
users can benefit from any auto configuration e.g. [RFC 6186](https://www.rfc-editor.org/rfc/rfc6186),
or [various other auto configuration methods](https://pipo.blog/articles/20210826-email-autoconf), 
and not having to change the username every time the password expires.)

The authentication works by trying one passdb first (for roundcube login with two 
factor authentication, for example) and if this fails, try to find the same username
in application passwords, looking up by password.

The following example looks up user@example.com first in `/etc/dovecot/auth/passwd`,
but ONLY allows from our trusted IPs (roundcube). Users are required to have two-factor
authentication in roundcube. 

Users will no longer be able to use the same password for roundcube (without 2fa) on 
a mobile device/IMAP client.

We also do not want users logging in to roundcube using an application-specific password.
(Some 2fa plugins can enforce 2fa. Others do not.)

To make it easy to determine if a login is from roundcube, roundcube is configured to
connect to ports 5993 for IMAP and 5465 for submission. Make sure these ports are 
NOT accessible externally (firewall etc.) (Unfortunately dovecot does not currently 
let us do: `deny_real_nets=` or `allow_real_nets=!192.168.10.1`)

### Dovecot ports

Add some secure ports for roundcube to use:

`10-master.conf` example:

```
service imap-login {
  inet_listener imap {
    # disable port 143
    port = 0
    # port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
  inet_listener roundcube-imaps {
    port = 5993
    ssl = yes
  }

[...]

service submission-login {
  inet_listener submission {
    # disable port 587
    port = 0
    #port = 587
  }

  inet_listener submissions {
    port = 465
    ssl = yes
  }
  inet_listener roundcube-submissions {
    port = 5465
    ssl = yes
  }
}

```
NOTE: As per [RFC 8314](https://www.rfc-editor.org/rfc/rfc8314) (2018), it is now recommended to use implicit TLS, and
as of [RFC 8997](https://www.rfc-editor.org/rfc/rfc8997) at least TLS v1.2.

For many years, the recommendation was to use STARTTLS on ports 143/587 and deprecate the use of ports 993/465.
This is still the default configuration for many mail servers / clients. (STARTTLS on ports 143/587 is preferred).

RFC 8314 **reversed** this recommendation: Use implicit TLS everywhere and deprecate the use of STARTTLS on ports 143/587.
This is shown in the example above. **You may not wish to do this if you have many clients already using STARTTLS on 
ports 143/587 and no way to auto-configure them**. Provided client implementations use STARTTLS properly and the 
server NEVER accepts plain text passwords before STARTTLS, there is nothing wrong with using STARTTLS. (In the past, 
some clients were broken and sent the username/password without encryption anyway, even though the server asked them not to!)

There is some confusion with client implementations and the wording "SSL" "TLS" "STARTTLS" "TLS/SSL" etc.
SSL, "Secure Sockets Layer" [was deprecated in 2015](https://datatracker.ietf.org/doc/html/rfc7568) and should no longer be used. [It has been replaced by TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security). 

Today, when people refer to "SSL" they usually mean "TLS".

Clients use these terms interchangeably e.g. "SSL" meaning "Connect to the port using implicit TLS",
with "TLS" meaning "Connect to unencrypted port, use STARTTLS" 

Other times, "SSL" means "force use of deprecated SSLv3", and "TLS" means "use TLS" (with/without STARTTLS ?)

To remove this confusion, I just always use implicit TLS: Then clients will ALWAYS use encryption (or fail), instead of trying to use other unwanted/insecure methods.

As I never use ports 143/587, I disable them to reduce port scans/login attempts etc. (It also seems a waste
of effort having _every_ connection first connect unencrypted, then start encryption using STARTTLS, when
I _always_ want to use TLS anyway.) 

The goal is to make it simpler for users to configure the correct settings, ensuring they can ONLY 
use the most secure encryption method, and only use an up-to-date client which supports it. 


### Roundcube config.inc.php:

Configure roundcube to use new ports.


```php
// IMAP host chosen to perform the log-in.
// See defaults.inc.php for the option description.
$config['imap_host'] = 'ssl://mailserver.example.com:5993';

// SMTP server host (for sending mails).
// See defaults.inc.php for the option description.
$config['smtp_host'] = 'ssl://mailserver.example.com:5465';
```

- **NOTE FOR ROUNDCUBE VERSIONS BEFORE 1.6.x**: 
  - Use `default_host` and `default_port` instead of `imap_host`.
  - Use `smtp_server` and `smtp_port` instead of `smtp_host`.


### Auth Config Example

```

# Example lookup from static passwd file

passdb system_users {
  driver = passwd-file
  username_filter = *@*
  passwd_file_path = /etc/dovecot/auth/passwd
  result_failure = continue-fail
  auth_username_format = %{user | lower}
  # Uncomment to only allow if from roundcube/trusted nets:
  #fields {
  #  allow_real_nets = local,192.168.10.1,2001:DB8:10:1a4::1
  #}
  auth_verbose = no
}

# Example SQL Login via webmail/trusted (based on postfixadmin's table (See README_LAST_ACCESS.md))

sql_driver = mysql

mysql localhost {
  dbname = roundcube
  user = YOUR_MYSQL_USER
  password = YOUR_MYSQL_PASS
}

passdb sql_local {
  driver = sql
  username_filter = *@*
  skip = authenticated
  # TODO - Make default BLF-CRYPT. Migrate remaining SHA512 hashes.
  # default_password_scheme = BLF-CRYPT
  default_password_scheme = SHA512-CRYPT
  auth_verbose = no
  sql_query = SELECT username, password FROM p_mailbox WHERE username = '%{user}' AND active='1';

  fields {
    allow_real_nets = local,192.168.10.1,2001:DB8:10:1a4::1
  }
}

# Roundcube connects to ports 5993 (imap) and 5465 (submission)
# Webmail users must use 2FA. Reject application-specific password
# if being used via webmail (port > 5000). Otherwise accept.

# ap4rc format 2
# Use the same username everywhere, select by password:

passdb sql_ap { 
  driver = sql
  username_filter = *@*
  # username_filter = *@example.com  # Or limit to my domain
  skip = authenticated
  default_password_scheme = SHA512
  auth_verbose = no

  sql_query = \
   SELECT username, password, id as userdb_ap_id \
   FROM application_passwords \
   WHERE username='%{user}' \
         AND password = '%{password | sha512}' \
         AND created >= NOW() - INTERVAL 12 MONTH;

  fields {
    allow_real_nets = local,%{real_local_port | if(">", 5000, "127.0.0.2", real_remote_ip)}
  }

}

[...]

# Now the corresponding userdb config:

# Get info for static system users
userdb system_users {
  driver = passwd-file
  passwd_file_path = /etc/dovecot/auth/passwd
  auth_username_format = %{user | lower}

  fields {
    home = /var/mailboxes/%{user | domain | lower}/%{user | username | lower}
  }
}

# Example post-login (or if no login e.g. lmtp), get all other user fields from sql:

userdb sql {
  driver=sql
  skip=found
  sql_iterate_query = SELECT username FROM p_mailbox WHERE active = '1';
  sql_query = SELECT username, CONCAT('/var/mailboxes/', maildir) AS home, \
                CONCAT('*:bytes=', quota) AS quota_rule \
              FROM p_mailbox \
              WHERE username = '%{user}' \
              AND active='1';
}

#
# Example fallback - add wanted fields if nothing else provides:
# 
userdb static {
  driver = static
  skip = found

  fields {
    home = /var/mailboxes/%{user | domain | lower}/%{user | username | lower}
  }

}

```

It's also possible to use IP without using different ports, but it starts to look ugly if you have a few, or IPv6 and IPv4.

```
# ap4rc format 2 - single IP 2001:DB8:10:1a4::2
passdb sql_ap {
[...]
  fields {
    allow_real_nets = local,%{real_remote_ip | if("eq", 2001\:DB8\:10\:1a4\:\:2, "127.0.0.2", real_remote_ip)}
  }
}
```

### SQL Dict Config 

`dovecot-sql-ap4rc.conf.ext`

- Dict/query config files are no longer required in Dovecot 2.4.


### Notes

- If dovecot's debug logging is enabled, the SQL query (and hence the user's password) can be logged. Ensure `auth_debug_passwords` and / or `log_debug=category=auth` is turned off on production servers.

- Dovecot v2.2.27+ support hashing a variable instead of passing the plaintext password to the SQL in order for mySQL to hash the password. This is safer since plaintext passwords will not show up in SQL/Dovecot server debug logs.

- Searching by password might not give good performance if you have a LOT of users.


## Preventing users from using roundcube password with IMAP

When implementing, it is recommended users change their existing password for good measure.

Once you have verified the config is working correctly with both existing and application 
specific passwords and have rolled out application specific passwords to any existing users, 
you will want to restrict use of the original password to roundcube only. (You have 
presumably enabled roundcube 2fa, so we want to make sure this password cannot be 
used without 2fa.)

There are various methods to do this, depending on your existing dovecot authentication setup and username format.

If your usernames are different (username formats 1, 3 or 4) you might be able to add a `username_filter` to each `passdb`.

If your roundcube usernames do not contain "@", but the application specific passwords do:

```
passdb { 
  # ... existing passdb for roundcube...
  username_filter = !*@*
}

passdb { 
  # ... passdb for application specific passwords...
  username_filter = *@*
}
```

If your roundcube usernames contain a domain:

```
passdb { 
  # ... existing passdb for roundcube...
  username_filter = *@example.com
}

passdb { 
  # ... passdb for application specific passwords...
  username_filter = !*@example.com, *@*
}
```

You may also want to look at disabling roundcube's `auto_create_user` option, to prevent "incorrect" accounts from 
being able to access it. You will have to create roundcube users yourself by adding them to the `users` table, and
maybe invent a tool to do that. Also note: If a user is deleted from roundcube's `users` table, the corresponding
user's entries are also removed from the `application_passwords table`.

**If a user is deleted from IMAP but not `application_passwords`** access will still be possible via 
application passwords. If expiry is enabled, application passwords will eventually expire.
You may want to implement a periodic check, or even a dovecot post-login script to remove entries from
application_passwords where the primary user no longer exists. (Or ensure your script/process for deleting
users also includes removing application_passwords entries.)

**If your usernames are the same** one method is shown in the example 2 above. For your EXISTING `passdb` config, add something like:

```
  fields {
    allow_real_nets = local,192.168.10.1,2001:DB8:10:1a4::1
  }
```

Where the IP addresses are your roundcube hosts. You can also add `local` temporarily to enable testing with `doveadm auth` / `doveadm user`.

This means the `passdb` section will only succeed (even if the password is correct) if the login is from the specified IPs.

Next, we want to express the _opposite_ logic: Do not allow the application specific passwords to be used via roundcube.

Unfortunately dovecot (at least 2.4.x) does not provide an easy way to do this, e.g. "`deny_real_nets`"  or "`allow_real_nets = !...`"

If your 2fa plugin _always_ enforces the use of 2fa, you may not care about this case: roundcube will prompt for 2fa authentication regardless of which password is used.

See the Auth config example for method 2 above.


## fail2ban etc...

If you use **fail2ban**: In some logging configurations, dovecot will first log `unknown user` from the first passdb entry,
before the application specific passdb entry is looked up:

```
mail dovecot: auth: passwd-file(user@example.com,192.168.100.146,<RH3fDX/4uNrURTGS>): unknown user
```

If there are too many of these, `fail2ban` will ban the client's IP address for the configured interval.

The solution (once you've finished debugging) is either to set `auth_verbose = no` in at least the first passdb entry, or tweak the fail2ban regexp e.g. `/etc/fail2ban/filter.d/dovecot.conf` to remove the `unknown user` match.

If you used the suggested ports for roundcube (5993 and 5465), even if fail2ban is triggered by a user repeatedly failing to login,
this will only block access via the standard ports (110,995,143,993,587,465). fail2ban will not block 5993/5465. (This is probably a good
thing: If the IP of the roundcube host is blocked, nobody will be able to access via roundcube!)

You may also want to add roundcube's IP addresses to dovecot's `login_trusted_networks` (usually `10-master.conf`).

Roundcube has its own built-in brute-force rate limit: `login_rate_limit`. This rate limit only applies for failed login attempts for _existing_ users in its database.

Roundcube (at least 1.6.1) does not prevent someone trying lots of different/random usernames and passwords. The login process
seems slow enough to make large-scale brute-force attacks fairly unfeasible. You may want to make it more difficult for
potential attackers to exploit roundcube's differing failed login behaviour to determine if a user exists or not. (Or just
to stop many failed login attempts from locking out users / wasting resources and cluttering your logs.)

Dovecot has its own [Authentication penalty support](https://doc.dovecot.org/2.4.3/core/config/auth/penalty.html) and there is a [Dovecot Authentication penalty plugin](https://packagist.org/packages/takerukoushirou/roundcube-dovecot_client_ip) for roundcube.

