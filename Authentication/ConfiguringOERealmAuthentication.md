# Configuring OERealm Authentication
## Setting Up The Server
The following steps only have to be carried out once per server instance - however, the settings used will be required whenever you are configuring a new REST project, so keep note of them.

* If you are using the Pacific Application Server, open `<Server Folder>/conf/jvm.properties` then add or update the following setting:

```ini
-Dorg.apache.catalina.STRICT_SERVLET_COMPLIANCE=false
```
* Open a Proenv command prompt as administrator and use the `cd` command to navigate to:
  * `<OpenEdge Work Folder>/<Server Name>/common/lib`, if using the Pacific Application Server.
  * `<Project Root>/RESTContent/WEB-INF/classes`, if using the classic AppServer.
* Generate a Client-Principal file using the `genspacp.bat` utility. This will be used to secure the HybridRealm class from being called by other PASOE clients.

```
genspacp -password <Password> -role <Role Name>

genspacp 1.0
Generated sealed Client Principal...
    User: BPSServer@OESPA
    Id: SmjnCQ1kTm2fY5r8pxQg5A
    Role: <Role Name>
    Encoded Password: <Encoded Password>
    File: oespaclient.cp
    State: SSO from external authentication system
    Seal is valid
```

* Create a new plain-text file called `spaservice.properties`, and enter the following options. Some important things to note:
	* The `Password` refers to the encoded password returned by `genspacp`, **not** the plain-text one entered on the command-line.
	* **Make sure a blank line is left at the end of the file!** Not doing this can lead to strange errors when the file is being parsed.

```ini
Password=<Encoded Password>
Role=<Role Name>
DebugMsg=true
```

* Save `spaservice.properties` to `<Server Folder>/openedge` if using Pacific Application Server, or your server's OpenEdge work folder if using a classic AppServer.

## Configuring Spring Security
These steps need to be carried out every time you create a new project, regardless of whether there are other projects running on the same server instance.

* Create an implementation of IHybridRealm and a Properties class in `<Project Root>/AppServer/auth`. Examples can be found on [Progress KnowledgeBase](http://knowledgebase.progress.com/servlet/fileField?id=0BEa0000000LNZj).
* Inside your project directory, open the `WEB-INF` folder for your chosen server configuration - if you are using a Pacific Application Server instance, this will be `<Project Root>/PASOEContent/WEB-INF`, whilst on a classic AppServer this will be `<Project Root>/RESTContent/WEB-INF`.
* In `WEB-INF/web.xml`, set `contextConfigLocation` to either `/WEB-INF/oeablSecurity-form-oerealm.xml` or `/WEB-INF/oeablSecurity-basic-oerealm.xml`, depending on which authentication method you want to utilize. Basic authentication is stateless, requiring a username and password to be sent in the header of each HTTP request, whereas Form authentication uses cookie-based sessions.

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
<!-- USER EDIT: Select which application security model to employ
        /WEB-INF/oeablSecurity-basic-local.xml
        /WEB-INF/oeablSecurity-anonymous.xml
        /WEB-INF/oeablSecurity-form-local.xml
        /WEB-INF/oeablSecurity-container.xml
        /WEB-INF/oeablSecurity-basic-ldap.xml
        /WEB-INF/oeablSecurity-form-ldap.xml
        /WEB-INF/oeablSecurity-basic-oerealm.xml
        /WEB-INF/oeablSecurity-form-oerealm.xml
        /WEB-INF/oeablSecurity-form-saml.xml
        /WEB-INF/oeablSecurity-basic-saml.xml
-->
        /WEB-INF/oeablSecurity-basic-oerealm.xml
    </param-value>
</context-param>
```

* Open the configuration file that you specified in the previous step.
* Configure `OERealmAuthProvider` with the following settings.
	* `key` should be set to the encoded password returned by `genspacp`.
	* If `sealClientPrincipal` is set to false, the client principal object will be passed through the `OEClientPrincipalFilter`, found in the same configuration file. This allows for a little more control over the object, but at the expense of added complexity - unless you have a specific reason to utilize the settings in `OEClientPrincipalFilter`, it is probably a good idea to set this to `true`.
	  * *This option was added in OpenEdge 11.6, and will not be present if configuring earlier versions.*
	* If you want to use the built-in OpenEdge user domains/database table, set `userDomain` to the name of the domain, and set `authz` to `true`. However, if you have a custom user table/authentication system, `userDomain` must be blank, and `authz` must be `false`. If you get this wrong, things will not work!


```xml
<b:bean id="OERealmAuthProvider"
    class="com.progress.appserv.services.security.OERealmAuthProvider" >
    <b:property name="userDetailsService">
        <b:ref bean="OERealmUserDetails"/>
    </b:property>
    
    <b:property name="createCPAuthn" value="true" />
    <b:property name="sealClientPrincipal" value="true" />
    <b:property name="multiTenant" value="false" />
    <b:property name="userDomain" value="" />
    <b:property name="key" value="<Encoded Password>" />
    <b:property name="authz" value="false" />
</b:bean>
```

* Configure `OERealmUserSettings` with the following settings.
	* `realmURL`'s value will vary depending on what type of application server is being used:
		* For a remote classic AppServer using NameServer, use `AppServer[s]://<Name Server Host>:<Name Server Port>/<Broker Name>`.
		* For a remote classic AppServer using DirectConnect, use `AppServerDC[s]://<AppServer Host>:<AppServer Port>/<Broker Name>`.
		* For a remote classic AppServer using AIA, use `http[s]://<Host>[:<Port>]/<AIA App Name>/aia`.
		* For a remote Pacific Application Server, use `http[s]://<Host>[:<Port>]/<OEABL App Name>/apsv`. This requires the APSV transport to be enabled on your ABL application - follow the steps outlined in Progress' [documentation](http://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/ompas/managing-apsv-transports.html) to ensure this is set up correctly.
		* For a local Pacific Application Server, use `internal://localhost/nxgas`.
	* `realmClass` should be set to the namespaced path of your HybridRealm class. If you are using the example implementation, this will probably be `auth.HybridRealm`.
	* `realmPwdAlg` sets the algorithm that the server should use to validate the user's credentials. If the value is `0`, the data will be sent to the server using HTTP Basic authentication. Unless you are using SSL to connect, **this is insecure**, as the credentials will be sent as clear-text, making it possible for them to be intercepted. If the value is `3`, the data will be sent using HTTP Digest authentication, which encodes the password on the client side before sending it to the server. Use SSL where possible, and HTTP Digest anywhere else - **never use basic auth on its own**, as it will leave your application open to [man-in-the-middle attacks](https://en.wikipedia.org/wiki/Man-in-the-middle_attack).
	* `realmTokenFile` should be set to the filename of the client-principal file generated by `genspacp`. Unless you went out of your way to rename it, this should be `oespaclient.cp`.

```xml
<b:bean id="OERealmUserDetails"
        class="com.progress.appserv.services.security.OERealmUserDetailsImpl" >
        <b:property name="realmURL" value="http://<Host Name>:<Port>/<Application Name>/apsv" />
        <b:property name="realmClass" value="<HybridRealm Class>" />
        <b:property name="grantedAuthorities" value="ROLE_PSCUser" />
        <b:property name="rolePrefix" value="ROLE_" />
        <b:property name="roleAttrName" value="ATTR_ROLES" />
        <b:property name="enabledAttrName" value="ATTR_ENABLED" />
        <b:property name="lockedAttrName" value="ATTR_LOCKED" />
        <b:property name="expiredAttrName" value="ATTR_EXPIRED" />
        <b:property name="realmPwdAlg" value="3" />
        <b:property name="realmTokenFile" value="oespaclient.cp" />
        <!-- For SSL connection to the oeRealm appserver provide the complete
             path of psccerts.jar as the value of 'certLocation' property
        -->
        <b:property name="certLocation" value="" />
        <!-- set appendRealmError = true in order to append the Realm 
        class thrown error in the error details send to the REST Client -->
        <b:property name="appendRealmError" value="false" /> 
    </b:bean>
```