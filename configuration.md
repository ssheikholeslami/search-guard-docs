<!---
Copryight 2016 floragunn UG (haftungsbeschränkt)
-->

# Configuration

## Basic concepts

All configuration settings for Search Guard, such as users, roles and permissions, are stored as documents in a special Elasticsearch index. This index is specially secured so that only an admin user with a special SSL certificate may write or read this index. You can define one or more of these certificates, which we'll call **admin certificates**.

Keeping the configuration settings in an Elasticsearch index enables hot config reloading. This means that you can **change any of the user-, role- and permission settings at runtime , without restarting your nodes**. Configuration changes will **take effect nearly immediately**. You can load and **change the settings from any machine** which has access to your Elasticsearch cluster. 

**This also means that you do not need to keep any configuration files on the nodes themselves!** No more dealing with configuration files on different servers!

The configuration consists of the following files. These are shipped with Search Guard, and you can use them as templates for your own, specific configuration:

* sg\_config.yml
 * Configure authenticators and authorisation backends
* sg\_roles.yml
 * define the roles and the associated permissions
* sg\_roles\_mapping.yml
 * map backend roles, hosts and users to roles
* sg\_internal\_users.yml
 * user and hashed passwords (hash with hasher.sh)
* sg\_action\_groups.yml
 * group permissions together

## Hot config reloading: Using sgadmin

Configuration settings are loaded into the Search Guard index using the `sgadmin` tool.
sgadmin identifies itself against the Elasticsearch cluster (which is secured by SG 2) by a _special_ client SSL certificate. Ok, not the certificate is special, it's just that the CN (common name) of the certificate is known by SG 2 by adding it to the ``searchguard.authcz.admin_dn:`` property in elasticsearch.yml. This is described [here](https://github.com/floragunncom/search-guard/wiki/Installation). If you use the [example SSL scripts](https://github.com/floragunncom/search-guard-ssl/tree/master/example-pki-scripts) to generate the certificates it's for example the _kirk_ or _spock_ client certificate.
 
```
chmod +x plugins/search-guard-2/tools/sgadmin.sh

plugins/search-guard-2/tools/sgadmin.sh \
   -h <elasticsearch hostname, default: localhost> \
   -p <elasticsearch transport port, default: 9300> \
   -ks <keystore file> \
   -ts <truststore file> \
   -kspass <password for keystore> \
   -tspass <password for truststore> \
   -cd <configuration directory> \
   -cn <clustername, default: elasticsearch> \
   -sniff \
   -icl \
   -f <file> \
   -t <type> \
   -nhnv \
   -nrhn \
```

* -h:  elasticsearch hostname, default: localhost
* -p:  elasticsearch port, default: localhost (NOT the http port!)
* -ks: The above mentioned client certificate (as .jks store)
* -ts: truststore which contains the root certificate (as .jks store)
* -cd: configuration directory containing a bunch of .yml files, details see below
* -cn: clustername, default: elasticsearch
* -sniff: [Sniff cluster nodes](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/transport-client.html)
* -icl: [Ignore clustername](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/transport-client.html)
* -f: single config file (cannot be used together with -cd) 
* -t: type, if -f is given then this must be one of
  * config
  * roles
  * rolesmapping
  * internalusers
  * actiongroups
* -nhnv: disable-host-name-verification, do not validate hostname
* -nrhn: disable-resolve-hostname, do not resolve hostname (only relevant if -nhnv is not set)

After one or more files are updated Search Guard will automatically reconfigure and the changes will take effect almost immediately. No need to restart ES nodes and deal with config files on the servers. The sgadmin tool can also be used fom a desktop machine as long as ES servers are reachable through 9300 port (transport protocol).

### Examples

#### Apply default configuration
```
plugins/search-guard-2/tools/sgadmin.sh \
   -cd plugins/search-guard-2/sgconfig/ \
   -ks plugins/search-guard-2/sgconfig/keystore.jks \
   -ts plugins/search-guard-2/sgconfig/truststore.jks  \
   -nhnv
```

#### Apply single file to cluster named "myclustername"
```
plugins/search-guard-2/tools/sgadmin.sh \
   -f /path/to/file/updated_internal_users.yml \
   -t internalusers \
   -ks plugins/search-guard-2/sgconfig/keystore.jks \
   -ts plugins/search-guard-2/sgconfig/truststore.jks  \
   -cn myclustername \
   -nhnv
```

#### Apply single file ignoring clustername
```
plugins/search-guard-2/tools/sgadmin.sh \
   -h esnode1.mycompany.com \
   -p 9301 \
   -f /path/to/file/updated_internal_users.yml \
   -t internalusers \
   -ks plugins/search-guard-2/sgconfig/keystore.jks \
   -ts plugins/search-guard-2/sgconfig/truststore.jks  \
   -icl
```

## Configuring authentication and authorisation 

The main configuration file for Search Guard is `sg_config.yml`. This file is used to specify both authentication and authorisation and has three main parts:

```
searchguard:
  dynamic:
    http:
      ...
    authc:
      ...
    authz:
      ...
```

In short, these sections are used to specify how Search Guard retrieves the user credentials, how to verify these credentials, and where to get the (additional) users roles from. The latter is optional.

### Authentication

Let's first look at the `authc` section. This section has the following format:

```
<name>: 
  enabled: <true|false>
  order: <integer>
    http_authenticator:
      ...
    authentication_backend:
      ...
```

An entry in the `authc` section of the config file is called an `authentication domain`. It specifies where to get the user credentials from, and against which backend they should be authenticated.

The first thing you'll notice is that you can define more than one authentication domains in this section. Each authentication domain has a telling name (e.g. "basic_auth_internal"), an `enabled` flag and an `order`. This makes it possible to chain authentication domains together, and Search Guard will execute them in the order provided. Let's concentrate on a single entry first.

The two sections you need to fill are `http_authenticator` and the `authentication_backend`. The `http_authenticator` specifies which authentication method you want to use on the HTTP layer.

The syntax for defining an authenticator on the HTTP layer is:

```
http_authenticator:
  type: <type>
  challenge: <true|false>
  config:
    ...
```

Allowed values for `type` are:

* basic
 * HTTP basic authentication. This type needs `challenging` set to `true`. No additonal configuration is needed.
* kerberos
 * Kerberos authentication. This type needs `challenging` set to `false`. Additonal. Kerberos-specific configuration is needed.
* clientcert (not challenging, needs https)
 * Authentication via a client TLS certificate. This certificate must be trusted by one of the Root CAs in the truststore of your nodes. See also Search Guard SSL (todo: link) docs on certificates and Root CAs. This type needs `challenging` set to `false`. No additonal configuration is needed.
* proxy
 * Use an external, proxy based authentication. This type needs `challenging` set to `false`. Additional, proxy-specific configuration is needed, and the "X-forwarded-for" module has to be enabled as well. See "Proxy authentication" for further details.

The config section contains configuration settings for the selected type. At the moment, only the Kerberos and Proxy type need additional configuration. See these sections in the documentation for further information.

After the HTTP authenticator was executed, you need to specify against which backend system you want to authenticate the user. This is specified in the `authentication_backend` section and has the following format:

```
authentication_backend:
  type: <type>
  config:
    ...
```

Possible vales for `type` are:

* noop
 * This means that no authentication against a backend system is performed. This setting only makes sense if the HTTP authenticator already authenticated the user, or if the request carries some credentials that are implicitely trusted. The former is true if you choose Kerberos as HTTP authentication type, the latter is true if you choose Proxy as HTTP authentication type. Note that while it would theoretically be possible to validate an already Kerberos-authenticated user a second time against an LDAP server, this scenario is probably not a real-word use-case. 
* internal
 * Use the users and roles defined in `sg_internal_users` for authentication. This requires you to specify users and roles in the file `sg_internal_users.yml`, and load them into Search Guard by using the `sgadmin` command line tool.
* ldap
 *  Authenticate users against an LDAP server. This requires additional configuration settings, see section "LDAP and Active Directory" for further details.

### Authorization

After the user has been authenticated, Search Guard can optionally collect additional user roles from backend systems. Authorisation configuration has the following format:

```
authz:    
  <name>:
    enabled: <true|false>
    authorization_backend:
      type: <type>
      config:
        ...
```

You can also define multiple entries in this section, the same way as you can for authentication entries. The execution order is not relevant here, hence there is no `order` field.

Possible vales for `type` are:

* noop
 * Used for skipping this step altogether
* ldap 
 * Fetch additional roles from an LDAP server. This requires additional configuration settings, see section "LDAP and Active Directory" for further details.

### HTTP

```
anonymous_auth_enabled: <true|false>
xff:
  enabled: <true|false>
  internalProxies: '192\.168\.0\.10|192\.168\.0\.11' # regex pattern
  #internalProxies: '.*' # trust all internal proxies, regex pattern
  remoteIpHeader:  'x-forwarded-for'
  proxiesHeader:   'x-forwarded-by'
  #trustedProxies: '.*' # trust all external proxies, regex pattern
  ###### see https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html for regex help
  ###### more information about XFF https://en.wikipedia.org/wiki/X-Forwarded-For
  ###### and here https://tools.ietf.org/html/rfc7239
  ###### and https://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#Remote_IP_Valve
```

#### LDAP and Active Directory

LDAP and AD can be used both in the `authc` and `authz` section of the configuration. 

**Authentication:**

```
authentication_backend:
  type: ldap 
  config:
    ...
```

**Authorization:**

```    
authorization_backend:
  type: ldap 
  config:
    ...    
```
    
For both entris you need to specify additional configuration parameters. Most of them are identical between the `authc` and `authz` section. For `authz`, you need to specify some additional settings.

Common configuration parameters:

```
config:
  # enable ldaps
  enable_ssl: false
  # enable start tls, enable_ssl should be false
  enable_start_tls: false
  # send client certificate
  enable_ssl_client_auth: false
  # verify ldap hostname
  verify_hostnames: true
  hosts:
    - localhost:8389
  bind_dn: null
  password: null
  userbase: 'ou=people,dc=example,dc=com'
  # Filter to search for users (currently in the whole subtree beneath userbase)
  # {0} is substituted with the username 
  usersearch: '(uid={0})'
  # Use this attribute from the user as username (if not set then DN is used)
  username_attribute: null
```

If used in the authz section, the following additional `parameters` are available:

```
config:
  rolebase: 'ou=groups,dc=example,dc=com'
  # Filter to search for roles (currently in the whole subtree beneath rolebase)
  # {0} is substituted with the DN of the user
  # {1} is substituted with the username 
  # {2} is substituted with an attribute value from user's directory entry, of the   authenticated user. Use userroleattribute to specify the name of the attribute            
  rolesearch: '(uniqueMember={0})'
  # Specify the name of the attribute which value should be substituted with {2} above
  userroleattribute: null
  # Roles as an attribute of the user entry
  userrolename: memberOf
  # The attribute in a role entry containing the name of that role
  rolename: cn
  # Resolve nested roles transitive (roles which are members of other roles and so on ...)
  resolve_nested_roles: true
```

#### Kerberos

```
config:
  # If true a lot of kerberos/security related debugging output will be logged to standard out
  krb_debug: false
  # Acceptor (Server) Principal name, must be present in acceptor_keytab_path file
acceptor_principal: 'HTTP/localhost'
  # If true then the realm will be stripped from the user name
  strip_realm_from_principal: true
```

#### Proxy authentication

todo: write

#### Examples

Please refer to the Addendum A of this documentation for some common configuration examples.

## Configure internal users and roles

If you do not have any external authentication backend like LDAP or Kerberos, you can also define an internal list of users and their passwords and roles, and use that for authentication and authorization. This is done in the file `sg_internal_users.yml`.

```
admin:
  hash: $2a$12$xZOcnwYPYQ3zIadnlQIJ0eNhX1ngwMkTN.oMwkKxoGvDVPn4/6XtO
  roles:
    - readall
    - writeall

analyst:
  hash: $2a$12$ae4ycwzwvLtZxwZ82RmiEunBbIPiAmGZduBAjKN0TXdwQFtCwARz2
  roles:
    - readall

```
In `sg_config.yml`, authenticating a user against this intern user databse is referred to as `internal`.  

### Generate hashed passwords

The password hash is a salted bcrypt hash of your cleartext password. You can use the `hasher.sh` script that is shipped with Search Gueard to generate them:

``plugins/search-guard-2/tools/hasher.sh -p mycleartextpassword``

## Map backend roles, hosts and users to roles

Backend users, roles and hosts need to be mapped to Search Guard roles. This is done in the file `sg_roles_mapping.yml`. 

```
sg_read_write:
  backendroles:
    - management
    - operations
    - 'cn=ldaprole,ou=groups,dc=example,dc=com'
  hosts:
    - "*.devops.company.com"
  users:
    - janedoe
    - johndoe
```
As you see, you can map users directly to a Search Guard role, but also any backend role found during the authorisation phase. These backend roles can for example stem from an LDAP server, but you can also use internal roles defined in `sg_config.yml`.

When using TLS certificates for authentication, you can also use hostnames. Wildcards are allowed.

## Define roles and the associated permissions

Search Guard roles and their associated opemrissions are defined in the file `sg_roles.yml`. You can define as many roles as you like. The syntax to define a role, and associate permissions is as follows:

```
<sg_role_name>:
  cluster:
    - '<permission>'
  indices:
    '<indexname or alias>':
      '<type>':  
        - '<permission>'
      _dls_: '<querydsl query>'
      _fls_:
        - '<field>'
        - '<field>'
```

The `cluster` entry is used to define permissions on cluster level. The `indices` entry is used to define permissions as well as DSL/FLS on index level.

When a user make a request to elasticsearch then all defined roles will be evaluated to see if the user has permissions for the request. A request is always associated with an action and is executed against and index (or alias) and a type. If a request is executed against all indices (or all types) then the asterix ('*') can be used. 

Every role a user has will be examined if it allows the action against an index (or type). At least one role must match for the request to be successful. If no role matches then the request will be denied. 

Currently a match must happen within one single role - that means that permissions can not span multiple roles. 

For <permission>, <indexname or alias> and <type> simple wildcards are possible:
 
* An asterix (\*) will match any character sequence (or an empty sequence)
* A question mark (?) will match any single character (but NOT empty character)

Example: `\*my\*index` will match `my_first_index` as well as `myindex` but not `myindex1`

Example: `?kibana` will match `.kibana` but not `kibana`

**Note: You cannot have a dot (.) in the <permission>, <indexname or alias> or <type> name/wildcard**

For <permission>, <indexname or alias> and <type> also regular expressions are possible. 
You have to pre- and apend a `/` to use regex instead of simple wildcards: `/<java regex>/`.

Example: `/\S*/` will match any non whitespace characters

Note: You cannot have a dot (.) in the <permission>, <indexname or alias> or <type> regex. Use `\S` instead.

See [https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html](https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html)

### Defining permissions

Permissions can be applied on cluster- and index level. Cluster-level permissions always start with `cluster:`, while index-level permissions start with `indices:`. After that, a REST-style path further defines the exact action the permission grants.

For example, this permission would grant the right to execute a search on an index:

```
indices:data/read/search
```

While this permission grants the right to delete write to the index:

```
indices:data/write/index
```

On cluster-level, this permission grants the right to display the cluster health:

```
cluster:monitor/health
```

There is a plethora of permissions you can set. Search Guard is compatible with the permission definition of Shield up to version 2.1, so you can see [here](hhttps://www.elastic.co/guide/en/shield/2.1/reference.html#ref-actions-list) for a complete list.

Since there are so many permissions you can use, we strongly recommend to use action grouping (see next chapter) and work with action group aliases in `sg_roles.yml`.

### Defining document- and field level security

As with regular permissions, settings for document- and field-level security can be applied on index-level, meaning that you can have different settings for each index.

Document-level security basically means that you want to display only certain documents to a particular role. These "certain documents" are simply defined by a **standard Elasticsearch query**. Only documents matching this query will be visible for the role that the DLS is defined for.

In a typical use-case, you probably want to display only certain types of documents to a particular role. You can define a DLS-permission like that:

```
_dls_: '{"term" : {"_type" : "payroll"}}'
```

This grants the bearer of this permission to view documents of type `payroll`. You can define multiple queries, if this is the case they are OR'ed.

Note that you can make the DSL query as complex as you want, but since it has to be executed for each request, this of course comes with a performance penalty.

Defining FLS is even simpler: You specify one or more fields that the bearer of the permissions is able to see:

```
_fls_:
        - 'firstname'
        - 'lastname'
        - 'salary'
```       

In this case the fields `firstname`, `lastname` and `salary` would be visible. You can add as many fields as you like.

**Note that DLS and FLS is applied on Lucene-level, not Elasticsearch-level. This means that the final documents handed from Lucene to Elasticsearch are already filtered according to the DLS and FLS settings for added security.**

### Examples

Please refer to the Addendum B of this documentation for some common permission settings examples. (todo: link)

## Define action groups

Since defining permissions based on fine-grained actions can be quite verbose, Search Guard comes with a feature called action groups. As the name implies, this is a named group of actions. After an action group has been defined, you can refer to the group of actions simply by its name.

Action groups are defined in the file `sg_action_groups.yml`. The file structure is very simple:

```
<action group name>:
    - '<permission>'
    - '<permission>'
    - '<permission>'
    - ...
```

The definition of actions is the same as outlined in the chapter "Defining permissions". Wildcards are also supported. You can use any telling name you want, and you can also refernce an action group from within another action group:

```
SEARCH:
  - "indices:data/read/search*"
  - "indices:data/read/msearch*"
  - SUGGEST
SUGGEST:
  - "indices:data/read/suggest*" 
```
 
In this case, the action group `SEARCH` includes the (wildcarded) `search*` and `msearch*` permissions, and also all permissions defined by the action group `SUGGEST`. 

You can then reference these action groups in the file `sg_roles.yml` simply by name:

```
sg_readall:
  indices:
    '*':
      '*':
        - SEARCH 
```