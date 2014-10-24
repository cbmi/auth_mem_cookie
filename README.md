# Auth MemCookie Apache Module

Forked from a SourceForge repository.

- [About](http://authmemcookie.sourceforge.net/)
- [SourceForge Page](https://sourceforge.net/projects/authmemcookie/)

## What is `"Auth MemCookie`"?

"Auth MemCookie" is an Apache v2.0 Authentication and authorization modules are based on "cookie" Authentication mechanism.

The module doesn’t make Authentication by it self, but verify if Authentication "the cookie" is valid for each url protected by the module. The module validate also if the "authenticated user" have authorization to access url.

Authentication is made externally by an Authentication html form page and all Authentication information necessary to the module a stored in memcached identified by the cookie value "Authentication session id" by this login page.

## How it Works

### Phase 1 : The login Form

Authentication is made by a login form page.

This login page must authenticate the user with any authenticate source (ldap, /etc/password, file, database....) accessible to language of the page (php, perl, java... an ldap login page sample in php are in samples directory).

Then this page must set cookie that contains only a key the "Authentication unique id" of the "Authentication session".

The login page must store authorization and user information of the authenticated user in memcached identified by the cookie key "Authentication unique id".

The login page can be developed in any language you want, but must be capable to use memcached (they must have memcache client api for us)



### Phase 2 : The Apache v2 Module

After the user is logged, the apache 2 module check on each protected page by apache ACL the presence of the "cookie".

If the "cookie" exists, try to get session in memcached with the "cookie" value if not found return "HTTP_UNAUTHORIZED" page.

If session exists in memcached verify if ACL match user session information if not match return "HTTP_FORBIDDEN" page.&nbsp;

## Session format stored in memcached

The session store in memcached are composed with multiple line in form of "name" equal "value" ended by "rn". some are mandatory, other are optional&nbsp;and the rest are information only (all this field are transmitted&nbsp;to the script&nbsp;language&nbsp;protect the module).

Session format:

```
UserName=<user name>
Groups=<group name1>:<group name2>:...
RemoteIP=<remote ip>
Password=<password>
Expiration=<expiration time>
Email=<email>
Name=<name>
GivenName=<given name>
```

- Username: are mandatory.
- Groups: are mandatory, are used to check group in apache acl. if no group are know for the user, must be blank (Groups=rn)
- RemoteIP: are mandatory, used by remote ip check function in apache module.
- Password: are not mandatory, and is not recommended to store in memcached for security reson, but if stored,&nbsp;is sent to the script language protected by the module.
- The other fields are information only, but they are sent to the langage that are behind the module (via environement variable or http header).

The session field size is for the moment limited to 10 fields by default.

## Build dependency

You must have compiled and installed:

- libevent use by memcached.
- memcached the cache daemon it self.
- libmemcache the C client API needed&nbsp;to compile the Apache Module.

## Compilation

You must modify Makefile:

- Set correctly the MY_APXS variable to point to the apache "apxs" scripts.
- Add the memcache library path in MY_LDFLAGS variable if necessary (-L<my memcache lib path>)

How to compile:

```
make
make install
```

After that the "mod_auth_memcookie.so" is generated in apache "modules" directory.

## How to configure Apache Module

### Module configuration option:

This option can be used in `location` or `directory` apache context.

- `Auth_memCookie_Memcached_AddrPort` - List of IP or host addresses and port ':' separated of memcache(s) daemon to be used, coma separed, e.g. `host1:12000,host2:12000`
- `Auth_memCookie_Memcached_SessionObject_ExpireTime` - Session object stored in memcached expiry time, in secondes.
Used only if "Auth_memCookie_Memcached_SessionObject_ExpiryReset" is set to on.
Set to 3600 seconds by default.
- Auth_memCookie_Memcached_SessionObject_ExpiryReset` - Set to 'no' to not reset object expiry time in memcache on each url... set to yes by default
- `Auth_memCookie_SessionTableSize` - Max number of element in session information table. set to 10 by default.
- `Auth_memCookie_SetSessionHTTPHeader` - Set to 'yes' to set session information to http header of the authenticated users, set to no by default.
- Auth_memCookie_SetSessionHTTPHeaderEncode` - Set to 'yes' to mime64 encode session information to http header, set to no by default.
- `Auth_memCookie_CookieName` - Name of the cookie to used for check authentification, set to "AuthMemCookie" by default.
- `Auth_memCookie_MatchIP` - Set to 'no' to not check IP address set in cookie with the remote browser ip, set to 'yes' by default.
- `Auth_memCookie_GroupAuthoritative` - Set to 'no' to allow access control to be passed along to lower modules, for group acl check. set to 'yes' by default.
- `Auth_memCookie_Authoritative` - Set to 'yes' to allow access control to be passed along to lower modules.Set to 'no' by default.
- `Auth_memCookie_SilmulateAuthBasic` - Set to 'no' to not fix http header and auth_type for simulating auth basic for scripting language like php auth framework work., set to 'yes' by default

### Sample to configure Apache v2 Module:

Configuration sample for using Auth_memcookie apache V2 module:

```apache
LoadModule mod_auth_memcookie_module modules/mod_auth_memcookie.so

<IfModule mod_auth_memcookie.c>
 <Location />
 Auth_memCookie_CookieName myauthcookie
 Auth_memCookie_Memcached_AddrPort 127.0.0.1:11000

 # to redirect unauthorized user to the login page
 ErrorDocument 401 "/gestionuser/login.php"

 # to specify if the module are autoritative in this directory
 Auth_memCookie_Authoritative on
 # must be set without that the refuse authentification
 AuthType Cookie
 # must be set (apache mandatory) but not used by the module
 AuthName "My Login"
 </Location>

</IfModule>

# to protect juste user authentification
<Location "/myprotectedurl">
 require valid-user
</Location>

# to protect acces to user in group1
<Location "/myprotectedurlgroup1">
 require group group1
</Location>
```
