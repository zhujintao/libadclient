# libadclient
Active Directory client for c++, Python and Golang

Requirements: openldap or SunLDAP

DESCRIPTION:
  This simple C++/Python/Golang classes can be used to manipulate Active Directory from c++, Python and Golang programs.
  
  This module reuses some code from adtool by Mike Dawson.

INSTALL (c++ and Python):

Note: you must have [scons](http://www.scons.org/) installed

  1. $ git clone https://github.com/paleg/libadclient.git
  2. $ cd libadclient
  3. $ scons install (to build/install c++ library)
  4. $ python setup.py install (to build/install python library)

Note: step 4 depends on step 3. So if your want to upgrade python module, you should upgrade c++ library first. 

OS X 10.11 INSTALL NOTE:

If you are getting errors in python while importing module:
```
ImportError: dlopen(/Library/Python/2.7/site-packages/_adclient.so, 2): Library not loaded: libadclient.dylib
  Referenced from: /Library/Python/2.7/site-packages/_adclient.so
  Reason: unsafe use of relative rpath libadclient.dylib in /Library/Python/2.7/site-packages/_adclient.so with restricted binary
```
that is because [System Integrity Protection](https://support.apple.com/en-us/HT204899). You can fix it with:
```
$ sudo install_name_tool -change libadclient.dylib /usr/local/lib/libadclient.dylib /Library/Python/2.7/site-packages/_adclient.so
```

INSTALL (Golang):
```
 $ go get github.com/paleg/libadclient
```
You must have swig >= 3.0.6 and Golang >= 1.4 to compile library. However, Golang >= 1.5.1 is recommended.
When building Golang library only openldap >= 2.2 are supported.

Note: this library is not safe for concurrent use. If you need to use library from concurrently executing goroutines, the calls must be mediated by some kind of synchronization mechanism (channels or mutexes).


Full list of supported methods can be found in [adclient.h](https://github.com/paleg/libadclient/blob/master/adclient.h) (for c++) and [adclient.py](https://github.com/paleg/libadclient/blob/master/adclient.py) (for python)

USAGE NOTES:

login: 
  - Login can be performed with (boolean `adConnParams.secured` chooses login mode):
    * SASL DIGEST-MD5 auth (default). It requires properly configured DNS (both direct and reverse) and SPN records (see [issue 1](https://github.com/paleg/libadclient/issues/1#issuecomment-131693081) for details). 
    * SASL GSSAPI auth (`adConnParams.use_gssapi` must be set to `true`). It requires properly configured kerberos library (`krb5.conf`) with `default_keytab_name` set to keytab with computer account. `msktutil` can be used for this purpose, see [Squid Active Directory Integration](http://wiki.squid-cache.org/ConfigExamples/Authenticate/WindowsActiveDirectory) `Kerberos` part for example. 
    * simple auth (clear text username and password). It does not require proper DNS and SPN setup, but with simple auth AD will refuse to do some actions (e.g. change passwords). **For some configurations (2008 domain?)**, to get simple auth to work, binddn must contain domain suffix i.e. `adConnParams.binddn = "user@domain.local"`).
  - Login can be performed with:
    * domain DNS name (`adConnParams.domain`) with optional [AD site name](https://technet.microsoft.com/en-us/library/cc754697\(v=ws.11\).aspx) (`adConnParams.site`). ldap uries for `domain` will be detected via DNS SRV query (_ldap._tcp.SITE._sites.DOMAIN.LOCAL / _ldap._tcp.DOMAIN.LOCAL). If `adConnParams.search_base` is empty - it will be constructed automatically from domain name (DC=DOMAIN,DC=LOCAL).
    * vector of ldap uries (`adConnParams.uries`). Ldap uries must be prefixed with `adclient::ldap_prefix`. `adConnParams.search_base` must be set explicitly.
  - LDAP_OPT_NETWORK_TIMEOUT and LDAP_OPT_TIMEOUT can be set with `adConnParams.nettimeout`.
  - LDAP_OPT_TIMELIMIT can be set with `adConnParams.timelimit`.

USAGE SAMPLE (c++):
```cpp
#include "adclient.h"
#include <iostream>
#include <vector>
#include <map>
#include <string>

using namespace std;

int main() {
    adConnParams params;
    // login with a domain name
    params.domain = "DOMAIN.LOCAL";
    // choose DC from SITE (optional, could be ommited)
    params.site = "SITE";
    // or login with a list of ldap uries
    // params.uries.push_back(adclient::ldap_prefix + "Server1");
    // params.uries.push_back(adclient::ldap_prefix + "Server2");
    // params.search_base = "dc=DOMAIN,dc=LOCAL";
    params.binddn = "user";
    params.bindpw = "password";
    // simple auth mode
    // params.secured = false;
    
    // GSSAPI auth, only domain and use_gssapi required
    // params.domain = "DOMAIN.LOCAL";
    // params.site = "SITE";
    // params.use_gssapi = true;
    
    adclient ad;
    try {
        ad.login(params);
    }
    catch(ADBindException& ex) {
         cout << "ADBindLogin: " << ex.msg << endl;
         return 1;
    }
    try {
        ad.groupRemoveUser("Group", "User");
        vector <string> user_groups = ad.getUserGroups("User");
        vector <string> users_in_group = ad.getUsersInGroup("Groups");
        vector <string> user_lastlogon = ad.getObjectAttribute("User", "lastLogon");
        map <string, vector<string> > user_attrs = ad.getObjectAttributes("User");
        vector <string> users = ad.getUsers();
        string dn = ad.getObjectDN("User");
        bool result = ad.checkUserPassword("User", "Password");
        bool disabled = ad.ifUserDisabled("User");
        bool locked = ad.ifUserLocked("User");
    }
    catch (const ADOperationalException& ex) {
      cout << "ADOperationalException: " << ex.msg << endl;
    }
    catch (const ADSearchException& ex) {
      cout << "ADSearchException: " << ex.msg << endl;
    }

   return 0;
}
```

USAGE SAMPLE (python):
```python
import adclient

params = adclient.ADConnParams()
# login with a domain name
params.domain = "DOMAIN.LOCAL"
# choose DC from SITE (optional, could be ommited)
params.site = "SITE"
# or login with a list of ldap uries
# params.uries = [adclient.LdapPrefix+"Server1", adclient.LdapPrefix+"Server2"]
# params.search_base = "dc=DOMAIN,dc=LOCAL";
params.binddn = "user";
params.bindpw = "password";
    
# simple auth mode
# params.secured = False;

# GSSAPI auth, only domain and use_gssapi required
# params.domain = "DOMAIN.LOCAL";
# params.site = "SITE";
# params.use_gssapi = True;

ad = adclient.ADClient()
try:
  ad.login(params)
except ADBindError as ex:
  print("failed to connect to Active Directory: {}".format(ex))
  exit(1)
  
try:
  dn = ad.getObjectDN("User");
except adclient.ADSearchError as ex:
  code = ad.get_error_num()
  if code == adclient.AD_OBJECT_NOT_FOUND:
    print("no such user")
  else:
    print("unknown search error")
except ADOperationalError:
  print("unknown operational error")
```

USAGE SAMPLE (Golang):
```go
package main

import (
  "github.com/paleg/libadclient"
  "fmt"
)

func main() {
  adclient.New()
  defer adclient.Delete()
  
  params := adclient.DefaultADConnParams()
  // login with a domain name
  params.Domain = "DOMAIN.LOCAL"
  // choose DC from SITE (optional, could be ommited)
  params.Site = "SITE"
  // or login with a list of ldap uries
  // params.Uries = append(params.Uries, adclient.LdapPrefix()+"Server1", adclient.LdapPrefix()+"Server2")
  // params.Search_base = "dc=DOMAIN,dc=LOCAL";
  params.Binddn = "user";
  params.Bindpw = "password";
    
  // simple auth mode
  // params.Secured = false;

  // enable GSSAPI auth
  // params.UseGSSAPI = true
  
  params.Timelimit = 60
  params.Nettimeout = 60
  
  if err := adclient.Login(params); err != nil {
    fmt.Printf("Failed to AD login: %v\n", err)
    return
  }
  
  group := "Domain Admins"
  if users, err := adclient.GetUsersInGroup(group, true); err != nil {
    fmt.Printf("Failed to get users in '%v': %v\n", group, err)
  } else {
    fmt.Printf("Users in '%v':\n", group)
    for _, user := range users {
      fmt.Printf("\t%v\n", user)
    }
  }
}
```
