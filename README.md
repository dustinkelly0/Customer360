The Ping **Customer360** Solution provides a CIAM package for PingDirectory \ PingFederate \ PingDataSync

![Solution - Customer360](Customer360.png)

---
## Pre-Requisites
* PingID SDK Tenant / SDK Application
  * Logon to PingOne Admin (https://admin.pingone.com)
  * Download PingID SDK Properties file (Setup --> PingID --> Client Integrations --> Integrate with PingID SDK)
  * Create SDK Application (Applications --> PingID SDK Applications)
    * Enable Email / SMS (Application --> Configuration)

## Deployment
This repo contains 2 configuration sets for deployment:
* [Docker Compose](deployment/Compose)
* [Kubernetes](deployment/Kubernetes)

## Deployment Configuration

The bulk of the configuration is performed by a Postman API Collection:  
https://documenter.getpostman.com/view/1239082/SzRw2Axv

[Environment Variables](deployment)

## Post-Deployment Considerations
This Solution leverages **unsecured** LDAP between PingFederate and PingDirectory as it launches.  

To enabled LDAPS, follow these steps:

**Command Line**
1. Export the PingDirectory certificate
* Compose -- 
  * `docker-compose exec pingdirectory /opt/out/instance/bin/manage-certificates export-certificate --keystore /opt/out/instance/config/keystore --keystore-password-file /opt/out/instance/config/keystore.pin --alias server-cert --output-file server-cert.crt --output-format PEM`
* Kubernetes -- 
  * Connect to the Service -- `kubectl exec -it service/pingdirectory /bin/sh`
  * Execute this command -- `/opt/out/instance/bin/manage-certificates export-certificate --keystore /opt/out/instance/config/keystore --keystore-password-file /opt/out/instance/config/keystore.pin --alias server-cert --output-file server-cert.crt --output-format PEM`
1. Copy the certificate from the PingDirectory container
* Compose --
  * Get the Container names -- `docker container ls` - retrieve the Container IDs for PingDirectory and PingFederate
  * `docker cp {{pingdirectoryId}}:/opt/server-cert.crt ./server-cert.crt`
* Kubernetes -- 
  * Get the Container names -- `kubectl get pods` - retrieve the Container IDs for PingDirectory and PingFederate
  * `kubectl cp {{pingdirectoryId}}:/opt/server-cert.crt ./server-cert.crt`

**PingFederate Admin Console**
1. Import the PingDirectory certificate into PingFederate
* Open the Trusted CA store
  * `Security` --> `Trusted CAs`
* Import the `server-cert.crt` file

1. Edit the PingDirectory DataStore
* `System` --> `External Systems -- DataStores` --> `PingDirectory` --> `LDAP Configuration`
* Check the `Use LDAPS` box
* Test the Connection

**Disable LDAP on PingDirectory**
* Compose -- 
  * `docker-compose exec pingdirectory /opt/out/instance/bin/dsconfig set-connection-handler-prop --handler-name "LDAP Connection Handler" --set enabled:false --no-prompt`
* Kubernetes -- 
  * `kubectl exec service/pingdirectory /opt/out/instance/bin/dsconfig set-connection-handler-prop --handler-name "LDAP Connection Handler" --set enabled:false --no-prompt`

## Solution Configuration

### Administrator Consoles
| Product | Console URL |
| ----- | ----- |
| PingCentral | https://{{PC_HOSTNAME}}:9022
| PingFederate | https://{{PF_HOSTNAME}}:9999/pingfederate |
| PingDirectory | https://{{PD_HOSTNAME}}:8443/console |
| PingID | https://admin.pingone.com |

### PingFederate
---
To access the Admin UI for PF go to:  
`https://{{PF_HOSTNAME}}:9999/pingfederate`

Credentials (LDAP):  
`Administrator` / `2FederateM0re`

This configuration includes:

### Adapters
* HTML Form
* HTML Form with LIP
* Identifier-First (Passwordless)
* PingID SDK

**Social Logon**
* Apple (not configured)
* Facebook (can be configured in Environment)
* Google (can be configured in Environment)
* LinkedIn (not configured)

**Risk \ ID Proofing**
* iOvation IK (not configured)
* ID Data Web IK (not configured)

### Connections
* PingID SDK Connector
  * Auto-enrollment of Email \ SMS -- `(mobile=*)`

### PingID SDK - Special Considerations
The PingID adapter uses the secrets from your PingID tenant to create the proper calls to the service. As such, storing those values in a public location, such as GitHub, should be considered **risky**.

For this Profile, you can place the text from a `pingidsdk.properties` file into `postman_vars.json`. The API calls will base64 encode and inject into the PingIDSDK Adapter and HTML Form (for Self-Service Password Reset)

### PingID SDK - User Enrollment
The PingID SDK Connector is also configured to automatically enroll any User with a valid SMS number in the `mobile` field. If that User also has an email address, it will be enrolled as a secondary method.

**Note:** For email OTP - you will also need an email template created called `auth_without_payload` for the PF SDK Adapter to use Email as an OTP method. 

### AuthN Policy - Default AuthN Experiences
Extended Property Selector
* `Basic` (HTML Form with LIP & Social [`Google` | `Facebook`] & QR Code [PID SDK Mobile App])
* `MFA` (HTML Form with LIP --> PingID SDK)
* `Passwordless` (ID-First --> PingID SDK)
* `Internal` (HTML Form)

### AuthN Policy - Default AuthN API
Extended Property Selector
* `API` (ID-First --> HTML Form with LIP)

### AuthN Policy - Failback
Used for anything without an Ext Prop -- i.e. LIP Profile Management
* HTML Form with LIP & Social [`Google` | `Facebook`] & QR Code [PID SDK Mobile App]

### AuthN Policy - Forgot Password
Used to allow PID SDK for SSPR
* PID SDK Adapter

The Authentication Experience is controlled by setting the `Extended Properties` on the Application.   

### Extended Properties
* `Basic` (HTML Form with LIP & Social [`Google` | `Facebook`] & QR Code [PID SDK Mobile App]) *default*
* `MFA` (HTML Form with LIP --> PingID SDK adapter)
* `Passwordless` (ID-First --> PingID SDK)
* `Internal` (HTML Form)

### Authentication API
The AuthN API is enabled -- a value in the Extended Property of `API` will trigger it.
* ID-First --> HTML Form with LIP --> AuthN API Explorer 

### Applications
Two applications are pre-wired:

**SAML:**  
https://`${PF_BASE_URL}`/idp/startSSO.ping?PartnerSpId=Dummy-SAML

**OAuth \ OIDC:**  
`Issuer` == `${PF_BASE_URL}`  

**OIDC Logon**  
`client_id` == PingLogon  
`client_secret` == 2FederateM0re 

**Introspect**  
`client_id` == PingIntrospect  
`client_secret` == 2FederateM0re

### CIBA Authenticators
* Email
* PingID SDK
  * **Note:** Configuration is done with  [Use Case: Add CIBA to CIAM](https://www.getpostman.com/collections/246ba03433c2ffe26de0)

### Users
`user.[0-4]` / `2FederateM0re`

---
### PingCentral
To access the Admin Console for PC go to:  
https://{{PC_HOSTNAME}}:9022

User Credentials:
* `Administrator` / `2FederateM0re`
* `appowner.0` / `2FederateM0re`
* `appowner.1` / `2FederateM0re`

Access to PingCentral is controlled on the `employeeType` of the PS User - it's pulled with the OIDC Policy used by the PingCentral OIDC client.

PingCentral is not wired to any environment to begin with - but it will pull in the apps that are wired up in Customer360:

PingFed Environment:
* `PingFederate Admin`: `pingfederate:9999`
* `PingFederate Admin Username`: `api-admin`
* `PingFederate Admin Password`: `2FederateM0re`

---
### PingDirectory

To access the Admin Console for PD go to:  
https://{{PD_HOSTNAME}}:8443/console

Server:  
`pingdirectory`

Credentials (LDAP):  
`Administrator` / `2FederateM0re`

---
### PingDataSync

To access the Admin Console for PDS go to:  
https://{{PDS_HOSTNAME}}:8443/console

Server:  
`pingdatasync`

Credentials (LDAP):  
`Administrator` / `2FederateM0re`