This Server profile shows a complete install of PF \ PD with the Delegated Administator service and application configured.  

The [PF-Base Profile](https://github.com/cprice-ping/Profile-PF-Base) contains more complete documentation about how PingFed is configured.

This stack can be used as the basis of Delegated Admin Use Cases and includes the following structure \ rights:

![Delegated Admin](./DelegatedAdmin.png)

**Note:** `master` contains the latest version of Ping software. Prior versions can be found here:
* [Delegator 3.5](https://github.com/cprice-ping/Profile-DelAdmin/tree/delegator-v3)

## Deployment - Docker Compose
Environment variables in the `env_vars-sample` can be modified to inject the correct locations into this stack. Rename this to `env_vars` for it to be read by `docker-compose.yaml`  

The `postman_vars.json` file is used to inject values into the [PF-Base Profile](https://github.com/cprice-ping/Profile-PF-Base) API configuration.  

To implement this Use Case, download the `docker-compose.yaml` file and run `docker-compose up -d`

---
## Delegator Usage

The Delegator App is delivered behind an NGINX service. 

`http://{DELEGATOR_PUBLIC_URL}/delegator`

**Note:** If you are using self-signed certs in PD, you'll first need to create an exception for it in your browser. Without it, Delegator will fail with a CORS error (it's not CORS - it's SSL Validation that's failing).

Connect a browser to `https://${PD_HOST}:${PD_PORT}` to create the exception

---

Delegator can used to generate passwords, and leverages the Notification Email functionality in PingDir v8 to send a New User message to the email address of the created User.  
  
This email contains a link to a PF LIP Profile Management URL -- this can be used to switch from Delegated Admin to Self-Service Account Management.  

**Note:** A SMTP service is installed as part of the stack.  

PingFed is configured with 2 OAuth clients:
* PingLogon -- used to authenticate a user and issue tokens (AuthZ Code \ Implicit)
* PingIntrospect -- used to validate tokens (PD has a PF Access Token Validator pointing to this client)

## Delegated Admin Configuration 
This stack demonstrates several levels of delegated administration:
* Global (Users \ Groups \ OUs)
* Users (Users in `ou=People`)
* Groups (Groups in `ou=Groups`)
* Partners (OUs and Users in `o=Partners`)
* Partner Users (Partner Admin)

* User Profile (Self-Service Profile Management)
* Passwords (Self-Service Password Reset - PingID) 

**Note:** Using PingID as the SSPR mechanism means the user needs to have been enrolled into PID. You can first use the Dummy-SAML connection to set the Extended Properties -- `authNexp` to `MFA` and login as the user and enroll a device.

A set of PD users are also created and assigned Delegated Administrator roles:

These users are created in `ou=Administrators` to demonstrate separating the Admins from the objects they have rights to.

**Super Administrator**  
`SuperAdmin` \ `2FederateM0re`

**Partner Administrator**  
`PartnerAdmin` \ `2FederateM0re`

**User Administrator**  
`UserAdmin` \ `2FederateM0re`

**Group Administrator**  
`GroupAdmin` \ `2FederateM0re`  
This user is a member of the `DelAdmins` group that is used to delegate the Group resources

---
### Data Layer demonstration
If you want to show how DelAdmin is applied outside of Delegator, you can make calls via SCIM or DirAPI with the appropriate Delegated Admin credentials in the `Authorization: Basic` header. 

```
curl --location --request GET 'https://{{Your PingDirectory Host}}:1443/directory/v1/dc=coupademo.com/subtree?searchScope=wholeSubtree&limit=1000&filter=uid%20pr' \
--header 'Authorization: Basic ZjJhZG1pbjoyRmVkZXJhdGVNMHJl'
```

---
Delegated Objects are managed using the PingData console:  



### Onboarding new Partner Administrator
In order to show the onboarding of a new Partner, with Delegated Admin, do this:
* Logon to Delegator with `SuperAdmin` or `PartnerAdmin`
 * Create a PartnerOU
 * Create a PartnerAdmin User in the new OU (If more than 1 PartnerOU, you'll see a dropdown list)
* Logon to PD Console  
 `https://{{PingDataConsole}}:8443/console`
  * Server: `pingdirectory` 
  * User: `Administrator`
  * Pwd: `2FederateM0re`
 * Add new Delegated Admin rights to the DN of the User that was created
 * Assign **PartnerOU** rights with `read` in `resources-in-specific-subtrees` and the DN of the PartnerOU you put the Admin into
 * Assign **PartnerUsers** rights with `all` in `resources-in-specific-subtrees` and the DN of the PartnerOU you put the Admin into
 * Assign **Groups** with `all` in `resources-in-specific-subtrees` and the DN of the PartnerOU you put the Admin into

### Emailing new Users
An SMTP server is included in the stack with PD wired into it. There's a Notifier configured in [95-delegatedAdmin.dsconfig](pingdir/pd.profile/dsconfig/95-delegatedAdmin.dsconfig) that will attempt to send a message to the User Email that was entered for that User. The email will contain the `username`, the `password` and a link to PF Profile Management.

**Note:** On the PF Profile Management screen, the User can add \ modify their `phoneNumber` (not available in Delegator) and `Set Password`

---
PingFederate includes a couple of additional options:

* Virtual Host -- `pingfederate`  (Used for the backchannel ATV call from PD)
* OAuth AS --> Allowed Origins -- `http://${DELEGATOR_CORS_URL}`  (Used to allow Delegator to call OIDC endpoints)
* PingLogon client is configured for Implicit and a wildcard `redirect_uri`
* OIDC policy to map the User CN to the `name` claim -- this is displayed in the Sign Out option in Delegator

---
## Delegator Docker Image
The [delegator](/delegator) folder of the repo contains all the files needed to create your own Docker image of the Delegator app.

You may want your own image if you are doing custom branding or enabling custom UI fields (https://docs.pingidentity.com/bundle/pingdirectory-80/page/fpo1567772010793.html)

From the folder, you can run:  
`docker build . -t {image tag}`  

This will create a local image that can use used in the `docker-compose.yaml` file -- replace `pricecs\pingdelegator:4.0` with `{image tag}`