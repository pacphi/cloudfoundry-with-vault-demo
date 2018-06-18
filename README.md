# Using HashiCorp Vault with Cloud Foundry  

This is a simple example of an integration between a Spring MVC application powered [Spring Boot 2](http://projects.spring.io/spring-boot/) and [Spring Cloud Finchley RC2](https://spring.io/blog/2018/05/29/spring-cloud-finchley-rc2-has-been-released) and [Hashicorp Vault](https://www.vaultproject.io).  

Credit to [Zoltan Altfatter](https://blog.mimacom.com/hashicorp-vault-in-cloud-foundry-environment/).

## Prerequisites

* [CF CLI](https://github.com/cloudfoundry/cli#downloads) 6.37.0 or better if you want to push the application to a Cloud Foundry (CF) instance
* [httpie](https://httpie.org/#installation) 0.9.9 or better to simplify interaction with API endpoints
* Hashicorp [Vault](https://www.vaultproject.io/downloads.html) 0.12.0 or better
* Java [JDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 1.8u172 or better
* [jq](https://stedolan.github.io/jq/) 1.5 or better to 
* [Maven](https://maven.apache.org/download.cgi) 3.5.3 or better
* [ngrok](https://ngrok.com) 2.28 or better to expose local servers behind NATs and firewalls to the public internet over secure tunnels


## Clone

```
git clone https://github.com/pacphi/cloudfoundry-with-vault-demo.git
```

## How to build

```
cd cloudfoundry-with-vault-demo
mvn clean package
```

## How to start and prepare Vault

1. Start vault

```bash
vault server -config inmemory.conf
```

2. In another terminal set the VAULT_ADDR before initializing Vault

```bash
export VAULT_ADDR=http://127.0.0.1:8200
```   

2. Initialize vault

```bash
vault init -key-shares=5 -key-threshold=2
```

3. Copy the `Initial Root Token` 

> We will need it later.

```bash
export VAULT_TOKEN=<token>
```

Vault requires an authenticated access to proceed from here on. 
Vault uses tokens as generic authentication on its transport level.

4. Vault is in `sealed` mode, let's unseal it

```
vault operator unseal <key>
vault operator unseal <key>
```

5. Verify that Vault is in `unsealed` mode

```bash
vault status | grep Sealed

Sealed: false
```

6. Write a secret into the `secret` backend

```bash
vault write secret/vault-demo message='I find your lack of faith disturbing.'
```

## How to run Locally

### Token based authentication

We'll authenticate using the root [token](https://www.vaultproject.io/docs/auth/token.html) we exported as an environment variable.

> Don't do this in production!

7. Start the application in another terminal 

```bash
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=<token>
java -jar target/cloudfoundry-with-vault-demo-0.0.1-SNAPSHOT.jar --spring.cloud.vault.token=`echo $VAULT_TOKEN`
```

8. Make a GET request to http://localhost:8080

```bash
http :8080

message: I find your lack of faith disturbing.
```

9. Update the secret inside Vault

```bash
vault write secret/vault-demo message='Now, young Skywalker, you will die.'
```

10. Verify that the application still has the old secret

```bash
http :8080

message: I find your lack of faith disturbing.
```

11. Send refresh command to the application

```bash
http post :8080/actuator/refresh
```

12. Verify that the application knows about the latest secret

```bash
http :8080

message:'Now, young Skywalker, you will die.'
```

### AppRole based authentication

The [AppRole](https://www.vaultproject.io/docs/auth/approle.html#configuration) auth method allows machines or apps to authenticate with Vault-defined roles.

13. Enable the AppRole auth method

```bash
vault auth enable approle
```

14. Set policy to allow AppRole to access key-value pairs

```bash
vault policy write vault-demo vault-demo.hcl
```

15. Setup a new role

```bash
vault write auth/approle/role/vault-demo \
    secret_id_ttl=120m \
    token_num_uses=10 \
    token_ttl=60m \
    token_max_ttl=120m \
    secret_id_num_uses=10 \
    policies="vault-demo"
```

16. Get role-id and secret-id

```bash
vault read auth/approle/role/vault-demo/role-id
vault write -f auth/approle/role/vault-demo/secret-id
```

17. Capture role-id and secret-id as environment variables

```bash
export VAULT_ROLE=bde2076b-cccb-3cf0-d57e-bca7b1e83a52
export VAULT_SECRET=66eec22c-50c7-04e3-f86a-10623aebf163
```

> Replace role-id and secret-id above with your own

18. Restart application

```bash
java -jar target/cloudfoundry-with-vault-demo-0.0.1-SNAPSHOT.jar \
--spring.cloud.vault.authentication=APPROLE \
--spring.cloud.vault.app-role.role-id=`echo $VAULT_ROLE` \
--spring.cloud.vault.app-role.secret-id=`echo $VAULT_SECRET`
```

19. Verify that the application knows about the latest secret

```bash
http :8080

message:'Now, young Skywalker, you will die.'
```


## How to run on Cloud Foundry

### Token based authentication w/ Service Broker

> The service broker implementation is currently limited to token-based authentication scheme

1. Using [Pivotal Web Services](https://run.pivotal.io/)

> This is Pivotal's Cloud Foundry as a Service offering

```bash
cf login -a api.run.pivotal.io
```

2. Get the [Open Service Broker API](https://www.openservicebrokerapi.org/) implementation from HashiCorp

```bash
git clone https://github.com/hashicorp/vault-service-broker
```

3. You will  to change the `DefaultServiceID` and `DefaultServiceName` in the `main.go` file

4. Deploy the broker
   
```bash
cf push my-vault-broker-service -m 256M --random-route --no-start 
```
   
The `--no-start` makes sure it is not started after it is deployed.

5. Expose the locally running vault via [ngrok](https://ngrok.com/)

```
ngrok http 8200

Forwarding http://3db1eef2.ngrok.io -> localhost:8200
Forwarding https://3db1eef2.ngrok.io -> localhost:8200
```

6. Open a browser and verify ngrok's web interface is available at `http://localhost:4040`

7. Set the following environment variables

```bash
VAULT_ADDR=<ngrok_url>
VAULT_TOKEN=<token>
```

The broker is configured to use basic authentication

```bash
VAULT_USERNAME=vault
VAULT_PASSWORD=secret
```

> You'll want to replace the username and password values above with your own

8. Configure the environment variables

```bash
cf set-env my-vault-broker-service VAULT_ADDR ${VAULT_ADDR}
cf set-env my-vault-broker-service VAULT_TOKEN ${VAULT_TOKEN}
cf set-env my-vault-broker-service SECURITY_USER_NAME ${VAULT_USERNAME}
cf set-env my-vault-broker-service SECURITY_USER_PASSWORD ${VAULT_PASSWORD}
```

9. Verify the configured environment variables 

```bash
cf env my-vault-broker-service
```

10. Start the broker:

```bash
cf start my-vault-broker-service
```

11. Check the logs to verify a successful start

```bash
cf logs --recent my-vault-broker-service
```

12. Verify in the Ngrok Inspect UI the activity requests sent to the exposed Vault broker

```bash
GET /v1/sys/mounts
PUT /v1/auth/token/renew-self
POST /v1/sys/mounts/cf/broker
GET /v1/cf/broker
```
 
13. The service broker created a new mount
 
```
vault mounts

...
cf/broker/  generic    generic_4c6ea7ec    n/a     system       system   false           replicated
...
``` 

14. View the running broker:

```
cf apps

name                      requested state   instances   memory   disk   urls
my-vault-broker-service   started           1/1         256M     1G     vault-demo-twiggiest-sennit.cfapps.io
```  

15. Get the broker url

```bash
VAULT_BROKER_URL=$(cf app my-vault-broker-service | grep routes: | awk '{print $2}')
```

16. Get the catalog information:

```bash
curl ${VAULT_USERNAME}:${VAULT_PASSWORD}@${VAULT_BROKER_URL}/v2/catalog | jq
```

```json
{
  "services": [
    {
      "id": "42ff1ff1-244d-413a-87ab-b2334b801134",
      "name": "my-hashicorp-vault",
      "description": "HashiCorp Vault Service Broker",
      "bindable": true,
      "tags": [
        ""
      ],
      "plan_updateable": false,
      "plans": [
        {
          "id": "42ff1ff1-244d-413a-87ab-b2334b801134.shared",
          "name": "shared",
          "description": "Secure access to Vault's storage and transit backends",
          "free": true
        }
      ]
    }
  ]
}
```

17. Create a service broker:

```bash
cf service-brokers
cf create-service-broker my-vault-service-broker "${VAULT_USERNAME}" "${VAULT_PASSWORD}" "https://${VAULT_BROKER_URL}" --space-scoped
```

> You need to specify the `--space-scoped` and the `service ids` and `service name` must be unique. See `https://docs.cloudfoundry.org/services/managing-service-brokers.html`

18. Create a service instance

```bash
cf create-service my-hashicorp-vault shared my-vault-service
``` 

> Note that the first parameter to `cf create-service` must match the value of `DefaultServiceName` that you set in Step 3 above

19. Verify the result

```bash
cf services

name               service              plan     bound apps   last operation
my-vault-service   my-hashicorp-vault   shared                create succeeded
```

20. Verify the HTTP requests sent the exposed Vault service using the Ngrok Inspect UI:

```bash
POST /v1/sys/mounts/cf/0b24f466-9a54-4215-852e-2bcfab428a82/secret
PUT /v1/cf/broker/0b24f466-9a54-4215-852e-2bcfab428a82
GET /v1/sys/mounts
POST /v1/sys/mounts/cf/0b24f466-9a54-4215-852e-2bcfab428a82/transit
POST /v1/sys/mounts/cf/be7eedf8-c813-49e1-98f8-2fc19370ee4d/secret
POST /v1/sys/mounts/cf/5f7b0811-d90a-47f2-a194-951eb324f867/secret
PUT /v1/sys/policy/cf-0b24f466-9a54-4215-852e-2bcfab428a82
PUT /v1/auth/token/roles/cf-0b24f466-9a54-4215-852e-2bcfab428a82
```

When  a new service instance is provisioned using the broker, the following paths will be mounted:

* Mount the generic backend at `/cf/<organization_id>/secret/`
* Mount the generic backend at `/cf/<space_id>/secret/`
* Mount the generic backend at `/cf/<instance_id>/secret/`
* Mount the transit backend at `/cf/<instance_id>/transit/`

A policy named `cf-<instance_id>` is also created for this service instance which grants read-only access to `cf/<organization_id>/*`, read-write access to `cf/<space_id>/*` and full access to `cf/<instance_id>/*`

21. Create a service key

```bash
cf create-service-key my-vault-service my-vault-service-key
cf service-keys my-vault-service
```

22. Verify the received requests for Vault using the Ngrok Inspect UI

```bash
PUT /v1/auth/token/renew-self
PUT /v1/auth/token/renew-self
PUT /v1/cf/broker/0b24f466-9a54-4215-852e-2bcfab428a82/5cf104c9-4515-40f3-94de-a63ab77cb84b
POST /v1/auth/token/create/cf-0b24f466-9a54-4215-852e-2bcfab428a82
```

23. Retrieve credentials for this instance

```bash
cf service-key my-vault-service my-vault-service-key
```

```json
{
 "address": "https://1f81e0d3.ngrok.io/",
 "auth": {
  "accessor": "3705e5b2-c0bb-6398-ecff-e05a9e6a7b28",
  "token": "d5971c27-cf77-6ff0-f5c9-430fdfe07066"
 },
 "backends": {
  "generic": "cf/0b24f466-9a54-4215-852e-2bcfab428a82/secret",
  "transit": "cf/0b24f466-9a54-4215-852e-2bcfab428a82/transit"
 },
 "backends_shared": {
  "organization": "cf/be7eedf8-c813-49e1-98f8-2fc19370ee4d/secret",
  "space": "cf/5f7b0811-d90a-47f2-a194-951eb324f867/secret"
 }
}
```

24. Deploy the Vault client application

```bash
cf push --random-route --no-start 
```

In the application, we can leverage these services using the following configuration in the `bootstrap.yml` file. 
Note that we are only able to access the exposed backends.

```bash
spring:
  application:
    name: vault-demo
  cloud:
    vault:
      token: ${vcap.services.my-vault-service.credentials.auth.token}
      uri: ${vcap.services.my-vault-service.credentials.address:http://localhost:8200}
      generic:
        backend: ${vcap.services.my-vault-service.credentials.backends.generic:secret}
```

25. Bind the `my-vault-service` to the `vault-demo` application

```bash
cf bind-service vault-demo my-vault-service
```

26. Start the Vault client application

```bash
cf start vault-demo
```

27. Verify that the application has started successfully

```bash
cf logs --recent vault-demo
```

Make a note of the organization id in the log output.  You will need this value for the next step.  Consult the `LeaseAwareVaultPropertySource` in sample log output below.

```
2018-06-12T06:55:36.26-0700 [APP/PROC/WEB/0] OUT 2018-06-12 13:55:36.259  INFO 14 --- [           main] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource {name='vault', propertySources=[LeaseAwareVaultPropertySource {name='cf/0b24f466-9a54-4215-852e-2bcfab428a82/secret/vault-demo/cloud'}, LeaseAwareVaultPropertySource {name='cf/0b24f466-9a54-4215-852e-2bcfab428a82/secret/vault-demo'}, LeaseAwareVaultPropertySource {name='cf/0b24f466-9a54-4215-852e-2bcfab428a82/secret/application/cloud'}, LeaseAwareVaultPropertySource {name='cf/0b24f466-9a54-4215-852e-2bcfab428a82/secret/application'}]}
```

28. Let's write a secret into the Vault to the given generic backend and send a refresh command.

```bash
vault write cf/0b24f466-9a54-4215-852e-2bcfab428a82/secret/vault-demo message='Vault Rocks'
http post https://vault-demo-twiggiest-sennit.cfapps.io/actuator/refresh
```

> Replace application URL with your own

29. We can verify that the secret is retrieved via

```bash
http get http://vault-demo-twiggiest-sennit.cfapps.io

message: Vault Rocks
```

> Replace application URL with your own

## Resources

* [Spring Cloud project for creating Cloud Foundry service brokers](https://github.com/spring-cloud/spring-cloud-cloudfoundry-service-broker)

* [Managing Secrets with Vault](https://spring.io/blog/2016/06/24/managing-secrets-with-vault)

* [Binding to Data Services with Spring Boot in Cloud Foundry](https://spring.io/blog/2015/04/27/binding-to-data-services-with-spring-boot-in-cloud-foundry)