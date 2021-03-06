## ForgeRock Identity Edge Controller Installation Guide

### Install AM

Install AM 6.5 as described in the [Quick Start Guide](https://backstage.forgerock.com/docs/am/6.5/quick-start-guide/index.html). In production deployments, HTTPS should be used to protect network traffic. See [securing communications](https://backstage.forgerock.com/docs/am/6.5/install-guide/#secure-communications) for more information about securing AM. If HTTPS is enabled, then the IEC needs access to the certificate. This can be done by placing the certificate in the Operating System's certificate store.

### Install IEC AM Plugin

Copy the IEC AM Plugin to the AM web server. For example when using Tomcat with its home directory stored in the variable TOMCAT_HOME and AM installed to `${TOMCAT_HOME}/webapps/openam`, then:

    tar -xzf iec-am-plugin-*.tgz
    cp am-iec-plugin-*.jar ${TOMCAT_HOME}/webapps/openam/WEB-INF/lib
    cp config/* ${TOMCAT_HOME}/webapps/openam/config/auth/default

The Tomcat server must be restarted for this change to take effect.

### DS Configuration for IoT Identities

IoT identities are similar to user identities in AM, but has additional attributes and are stored alongside OAuth2 Clients. 
Prepare DS by configuring it to accept multiple structural object classes:

    ~/openam/opends/bin/dsconfig set-global-configuration-prop \
        --set single-structural-objectclass-behavior:accept \
        -p 4444 -X -D "cn=Directory Manager" -w "password" -n

Modify DS with the the IoT object class and add new attributes:

    ~/openam/opends/bin/ldapmodify --port 50389 --hostname localhost \
        --bindDN "cn=Directory Manager" --bindPassword password \
        ~/forgerock/ds/iot-device.ldif

### AM Configuration for IoT Realm

The IEC communicates with AM via an IoT enabled realm. Open the AM admin console and create a new realm called `edge`. Unless otherwise instructed, always use the default values provided. When configuring a production environment, consult the AM documentation on the best values for your use case.

The first step is to reconfigure the Identity Store to store the identities in the Application Store (AgentService), alongside OAuth2 clients. In the `edge` realm go to `Identity Stores > embedded` and change the following:

* Server Settings Tab
  * LDAP Organization DN: change to `ou=OrganizationConfig,ou=1.0,ou=AgentService,ou=services,o=edge,ou=services,dc=openam,dc=forgerock,dc=org`
  * Click `Save Changes`
* User Configuration Tab
  * LDAP Users Search Attribute: change to `ou`
  * LDAP Users Search Filter: change to `(objectclass=sunservicecomponent)`
  * LDAP User Object Class: add `sunservicecomponent`, `forgerockIotDevice`
  * LDAP User Attributes: add `sunkeyvalue`, `sunserviceID`, `edgeControllerIdentifier`, `edgeControllerVersion`, `edgeControllerPlatform`, `edgeNodeRegistrationStatus`, `edgeNodeRegistrationTime`, `edgeNodeRegistrationJwk`, `edgeNodeEnvironmentData`, `edgeClientIdentifier`, `edgeNodeConfig`, `edgeNodeUserConfig`, `edgeNodeType`, `edgeNodeDeviceCode`, `edgeNodePairedUser`
  * LDAP People Container Value: change to `default`
  * Select `Load Schema` and click `Save Changes`
* Authentication Configuration Tab
  * Authentication Naming Attribute: change to `ou`
  * Click `Save Changes`

The next step is to create an OAuth2 Provider. Go to `Dashboard > Configure OAuth Provider > Configure OpenID Connect` and click `Create`.

Edge node identity profiles can be created dynamically. Alternatively profiles must be created manually before they are registered. The default setup uses dynamic profile creation. To enable this go to `Authentication > Settings > User Profile` and set User Profile to `Dynamic`.

Edge node tasks like registration and token retrieval are done with the help of scripts in AM. To install the default registration and command scripts we must add the IEC Service to the realm. Adding the service will also configure the default authentication modules and OAuth2 group. Go to `Services`, click `Add a Service` and choose `IEC Service` under service type. For example use the following values and click `Create`:

* ID Token Issuer: `edge-device`
* ID Token Audience: `am.iec.com`
* ID Token Client Secret: `letmein`
* Challenge Signing Key: `es256test`

The signing key in the above configuration is one of the default test keys available in AM. For production systems, a new challenge signing key should be generated. Refer to [setting up keys and keystores](https://backstage.forgerock.com/docs/am/6.5/maintenance-guide/#chap-maint-keystores) for how to create and add a key to AM's keystore, remembering to note the alias of the new key. The challenge signing key must be an ECDSA P-256 asymmetric key.

### Edge Identity Manager

Copy the identity manager war file to the same server that is running the AM web server:

    cp edge-identity-manager-*.war ${TOMCAT_HOME}/webapps/identitymanager.war

The identity manager can be accessed via the path `/identitymanager`, for example [http://am.iec.com:8080/identitymanager](http://am.iec.com:8080/identitymanager).
In order for the identity manager to have the sufficient privileges to access AM, an AM admin user must be logged into the AM Admin Console in the same browser.

Currently, the identity manager has the following restrictions:

* AM's install directory has to be `openam`.
* The IoT realm has to be called `edge`.

### IEC Service

Firstly, it should be ensured that the device can communicate with the AM instance. Test the connection to AM by using curl to call the `serverinfo` endpoint of the AM instance:

    curl --request GET  http://am.iec.com:8080/openam/json/serverinfo/*

Unpack the tarball:

    tar -xzf iec-service-*.tgz
 
The following files are contained in the tarball:

|File |Directory After Installation |Description |
|--- |--- |--- |
|iecservice |/opt/forgerock/iec/bin |Main IEC Service executable. |
|iec.service |/lib/systemd/system |Systemd unit file. |
|lib.* |/opt/forgerock/iec/lib |3rd party libraries for IEC Service. |
|*.ta |/lib/optee_armtz |IEC trusted application in default OP-TEE TA directory. (OP-TEE installation only) |
|iecutil |- |IEC Utility executable used at install time and removed after system setup. |
|install.sh |- |Bash script used to perform installation and removed after system setup. |
|iec-config.json |- |IEC Service configuration used at install time and removed after system setup. |

Open the IEC Service configuration file:

    vim iec-config.json

Change the following values:

* iec_configuration.id_token_config.audience: `am.iec.com`
* am_configuration.url: `http://am.iec.com:8080/openam`

Run the install script:

    ./install.sh

The following files are created during the installation process or on first run:

|File                        |Directory             |Description                                                                        |
|---            |---                    |---                                                                                |
|iec-service.db     |/var/opt/forgerock/iec |IEC Service database, created at install time. (RichOS installation only)|
|TA secure storage  |/data/tee              |TA secure storage files, created at install time. (OP-TEE installation only)           |
|iecservice.log     |/var/opt/forgerock/iec |IEC Service log file.                                                                  |

The IEC is now installed and running as a daemon. If the target system is a Docker image or does not support systemctl, you will get a "Failed to connect to bus: No such file or directory" message. You can start the IEC Service manually with:

    /opt/forgerock/iec/bin/iecservice &

The IEC will repeatedly attempt to register itself with AM. A registered IEC will appear as an identity in the defined `edge` realm in the AM instance. This can be confirmed via the AM admin console or the Edge Identity Manager.

In production, once the IEC Service has been successfully installed the files extracted from `iec-service-*.tgz` should be deleted.

### Installing a client that uses the IEC SDK

Unpack the tarball:

    tar -xzf iec-sdk-*.tgz

Add the lib directory to the PATH e.g.

    export LD_LIBRARY_PATH=~/forgerock/lib
    export DYLD_LIBRARY_PATH=~/forgerock/lib

To test the IEC SDK, run the simpleclient example application:

    cd ~/forgerock/examples/simpleclient
    cp ~/forgerock/sdk-config.json .

Open the IEC SDK configuration file:

    vim sdk-config.json

Change the following values, save and exit:

* zmq_client.endpoint: `tcp://172.16.0.11:5556`

Use the IEC utility to initialise the IEC SDK and run the example:

    ~/forgerock/iecutil --file sdk-config.json --initialise sdk
    ./simpleclient

The client should register successfully and retrieve OAuth2 tokens. Confirm that the simpleclient identities were created via the AM admin console or the Edge Identity Manager. Once the client application has been successfully initialised the configuration file should be deleted.

When writing a C client application, use the following as reference:

* IEC SDK API - ${SDK_DIR}/include/libiecclient.h
* Client example programs - ${SDK_DIR}/examples/

The IEC SDK requires the Sodium and ZMQ libraries (supplied with the IEC SDK distribution) for compilation e.g.

    libs="-liecclient -lsodium -lzmq"
    gcc ${ex_dir}/${ex}.c -I${SDK_DIR}/include -L${SDK_DIR}/lib ${libs}

Run and enjoy!
