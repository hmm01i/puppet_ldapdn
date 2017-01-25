# puppet_ldapdn

puppet_ldapdn is a puppet type and provider that aims to simply manage ldap entries via ldapmodify and ldapadd commands. This is much more preferable than writing files directly to the database.

In essence the mechanism it uses is described as follows:

* Translate the puppet "ldapdn" resource into an in-memory ldif
* ldapsearch the existing dn to verify the current contents (if any)
* compare the results of the search with what should be the case
* work out which add/modify/delete commands are required to get to the desired state
* write out an appropriate ldif file
* execute it via an ldapmodify statement.


## Usage

Create a user

```puppet
ldapdn{'admin user':
  ensure => present,
  dn => 'uid=admin,cn=config',
  attributes => [
    'objectClass: top',
    'objectClass: person',
    'objectClass: inetorgperson',
    'uid: admin',
    'cn: admin',
    'userPassword: password'],
  unique_attributes => ['userPassword'],
}
```


Modify a single attribute

```puppet
ldapdn{'changelog':
  dn     => 'cn=changelog5,cn=config',
  attributes => [
    "nsslapd-changelogdir: /var/lib/dirsrv/slapd-${suffix}/changelogdb"
  ],
  unique_attributes => ['nsslapd-changelogdir']
}

```
## Reference

`ensure`
String. present or absent. Defaults to present

`dn`
String. The entry you want to ldapadd/ldapmodify. Defaults to resource name

`attributes`
Array. Attributes being set/changed for the resource. Each element should be formatted "attribute: value"
Sets the attributes that you wish (be sure to separate key and value with <semi-colon space>).

`unique_attributes`
Array. List of attributes that are unique.

Used to specify the behaviour of ldapmodify when there is an existing attribute with this name. If the attribute key is specified here, then the ldapmodify will issue a replace, replacing the existing value (if any), whereas if the attribute key is not specified here, then ldapmodify will simply ensure the attribute exists with the value required, alongside other values if also specified (e.g. for objectClass).

`indifferent_attributes`
Array. If attribute exists its value will not be changed (e.g. useful for passwords). If attribute does not exist and is secified in `attributes`, it will be created with specified value.

`auth_opts`
Array. Options passed to ldapadd/ldapmodify for authentication. Defaults to ['-QY', 'EXTERNAL',]

`remote_url`
String. Connect to a remote ldap server. Should be valid LDAP url. Port is optional
Example
```
remote_url => 'ldaps://ldap1.example.com:636'
```

`remote_ldap`
This is deprecated but still available for comptibility.

String. Just provide a hostname. The resource will construct a url using ldap://

## Examples

Examples of usage are as follows:

You might like to set a root password:

```puppet
ldapdn{"add manager password":
  dn => "olcDatabase={2}bdb,cn=config",
  attributes => ["olcRootPW: password"],
  unique_attributes => ["olcRootPW"],
  ensure => present,
}
```

A defined type to create OU's

In this example, multiple groups are created. Notice, that "objectClass" is not in unique_attributes, so that (in future) more objectClasses may be added to each ou, without others being replaced.

```puppet
$organizational_units = ["Groups", "People", "Programs"]
ldap::add_organizational_unit{ $organizational_units: }

define ldap::add_organizational_unit () {

  ldapdn{ "ou ${name}":
    dn => "ou=${name},dc=example,dc=com",
    attributes => [ "ou: ${name}",
                    "objectClass: organizationalUnit" ],
    unique_attributes => ["ou"],
    ensure => present,
  }

}
```


By default, all ldap commands are issued with the `-QY EXTERNAL` SASL auth mechanism. To deal with this, you might want to allow managing of the bdb database by this external mechanism, which then allows you to create a database without a "no write access to parent" error:

```puppet
ldapdn{"set general access":
  dn => "olcDatabase={2}bdb,cn=config",
  attributes => ["olcAccess: {1}to * by self write by anonymous auth by dn.base="cn=Manager,dc=example,dc=com" write by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage by * read"],
  ensure => present
}
```

Here is how you can create a database in the first place:

```puppet
ldapdn{"add database":
  dn => "dc=example,dc=com",
  attributes => ["dc: example",
                 "objectClass: top",
                 "objectClass: dcObject",
                 "objectClass: organization",
                 "o: example.com"],
  unique_attributes => ["dc", "o"],
  ensure => present
}
```

Additionally, you may need to specify alternative authentication options when managing resources:

```puppet
ldapdn{"add database":
  dn => "dc=example,dc=com",
  attributes => ["dc: example",
                 "objectClass: top",
                 "objectClass: dcObject",
                 "objectClass: organization",
                 "o: example.com"],
  unique_attributes => ["dc", "o"],
  ensure => present,
  auth_opts => ["-xD", "cn=admin,dc=example,dc=com", "-w", "somePassword"],
}
```

Optionally you can manage a remote ldap resources. Combine with option `auth_opts`:

```puppet
  ldapdn{"adding_new_user":
    dn => "cn=users,dc=mygroup",
    attributes => ["member: cn=myteam,cn=myunit,dc=mygroup"],
    ensure => present,
    remote_url => "ldaps://ldap1.example.com",
    auth_opts => ["-xD", "cn=admin,cn=example,dc=com", "-w", "somePassword"],
  }
```


To ensure an attribute exists, but ignore its subsequent value. An example of this is a password.

```puppet
ldapdn{"add password":
  dn => "cn=Geoff,ou=Staff,dc=example,dc=com",
  attributes => ["olcUserPassword: {SSHA}somehash..."],
  unique_attributes => ["olcUserPassword"],
  indifferent_attributes => ["olcUserPassword"],
  ensure => present
}
```

By specifying indifferent_attributes, ensure => present will ensure that if the key doesn't exist, it will create it with the desired passwordhash, but if the key does exist, it won't bother replacing it again. In this way you can keep passwords managed by something like phpldapadmin if you so wish.

Please report any bugs, and enjoy.

License
=======

This software is copyright free. Enjoy


