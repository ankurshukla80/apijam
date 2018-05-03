# Apigee Edge Service Broker Microgateway Coresident Plan: Secure a CF App

*Duration : 45 mins*

*Persona : API Team*

# Use case

TODO

# How can Apigee Edge help?

TODO

# Pre-requisites

* You have [installed and configured](http://docs.pivotal.io/partners/apigee/installing.html) the *Apigee Edge Service Broker for PCF tile*. Or you got a set of credentials from your instructor that has access to a PCF environment with *Apigee Edge Service Broker for PCF* tile. 

* You have installed [cf CLI](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html).

* You have an Apigee account and have access to an Apigee Org.

TODO * EdgeMicro

# Instructions

Before you begin, you will need to get the following from your PCF instance or receive them from your instructor.

YOUR-SYSTEM-DOMAIN: This the the domian/hostname where the PCF is deployed. If you are using self signed certs for this endpoint, you will have to use `--skip-ssl-validation` for some of the commands

PCF-USER-NAME: PCF username

PCF-PASSWORD: PCF Password

PCF_ORG: The instance of your PCF deployment. If you are familiar with PCF, you may just refer to this as ORG. Since Apigee also as a concept of ORG, we will call this PCF_ORG for this lab

PCF_SPACE: An org can contain multiple spaces. This is the space you will pick for this lab

PCF_API: PCF API Endpoint

PCF_DOMAIN: PCF Domain for your apps. 

APIGEE_ORG: Apigee Edge Org Name.

APIGEE_ENV: Apigee Edge Environment.

EDGEMICRO_KEY: Edge Micro Key.

EDGEMICRO_SECRET: Edge Micro Secret.

## Step 1: Create a Service Instance

### 1. Set your API endpoint to the Cloud Controller of your deployment.

```bash
cf api $PCF_API --skip-ssl-validation
Setting api endpoint to api.YOUR-SYSTEM-DOMAIN...
OK
API endpoint:  https://api.YOUR-SYSTEM-DOMAIN (API version: 2.59.0)
Not logged in. Use 'cf login' to log in.
```

### 2. Log in to your deployment and select an org and a space.

```bash
$ cf login
API endpoint: https://api.YOUR-SYSTEM-DOMAIN
Email> user@example.com
Password>
```

### 2. List the Marketplace services and locate the Apigee Edge service:

```bash
$ cf marketplace
Getting services from marketplace in org example / space development as user@example.com...
OK

service          plans                     description
apigee-edge      org, microgateway, microgateway-coresident         Apigee Edge API Platform
```

### 2. Create an instance of the Apigee Edge service. Select the microgateway-coresident service plan to have Apigee Edge Microgateway run in the same container as your Cloud Foundry app.

e.g. Replace INITIALS-YOUR-SERVICE-INSTANCE to something similar to dz-apigee-mg-coresident-service-instance.

```bash
$ cf create-service apigee-edge microgateway-coresident INITIALS-YOUR-SERVICE-INSTANCE -c \
'{"org":"YOUR-ORG", "env":"YOUR-ENV"}'
Creating service instance INITIALS-YOUR-SERVICE-INSTANCE in org apigee / space ...
OK
```

### 3. Use the cf service command to display information about the service instance:

```bash
$ cf service INITIALS-YOUR-SERVICE-INSTANCE

name:            dz-apigee-mg-coresident-service-instance
service:         apigee-edge
bound apps:
tags:
plan:            microgateway-coresident
description:     Apigee Edge API Platform
documentation:
dashboard:       https://enterprise.apigee.com/platform/#/

Showing status of last operation from service dz-apigee-mg-coresident-service-instance...

status:    create succeeded
```

## Step 2: Install the Plugin

### 1. Install the Apigee Broker Plugin as follows.

```bash
cf install-plugin -r CF-Community "apigee-broker-plugin"
Searching CF-Community for plugin apigee-broker-plugin...
Plugin apigee-broker-plugin 0.1.1 found in: CF-Community
Attention: Plugins are binaries written by potentially untrusted authors.
Install and use plugins at your own risk.
OK
Plugin Apigee-Broker-Plugin successfully uninstalled.
Installing plugin Apigee-Broker-Plugin...
OK
Plugin Apigee-Broker-Plugin 0.1.1 successfully installed.
```

### 2. Make sure the plugin is available by running the following command:

```bash
$ cf -h
…
Commands offered by installed plugins:
  apigee-bind-mg,abm      apigee-unbind-mgc,auc    enable-diego
  apigee-bind-mgc,abc     apigee-unbind-org,auo    has-diego-enabled
  apigee-bind-org,abo     dea-apps                 migrate-apps
  apigee-push,ap          diego-apps               dev,pcfdev
  apigee-unbind-mg,aum    disable-diego
```
## Step 3: Clone Sample App with Microgateway Coresident

### 1. Clone the Apigee Edge GitHub repo:
```bash
$ git clone https://github.com/apigee/cloud-foundry-apigee.git
```

### 2. Change to the org-and-microgateway-sample directory of the cloned repo:

```bash
$ cd cloud-foundry-apigee/samples/coresident-sample
```

### 3. In the coresident-sample directory, open manifest.yml.

### 4. Edit manifest.yml to change the name and host properties to values specific to your deployment. See the following example:

```yaml
---
applications:
- name: dz-target-nodejs-app 
  memory: 1G
  instances: 1
  host: dz-target-nodejs-app 
  health-check-type: http
  health-check-http-endpoint: /healthcheck
  path: .
  env:
    #APIGEE_MICROGATEWAY_PROXY: edgemicro_cf-test.local.pcfdev.io
    #APIGEE_MICROGATEWAY_CUST_PLUGINS: plugins
    #APIGEE_MICROGATEWAY_PROCESSES: 2
    APIGEE_MICROGATEWAY_CONFIG_DIR: config
    APIGEE_MICROGATEWAY_CUSTOM: |
                                {"policies":
                                  {
                                  "spikearrest":
                                    {
                                      "allow": 10,
                                      "timeUnit": "minute"
                                    }
                                  },
                                "sequence": ["healthcheck","spikearrest"]
                                }
```

To support Cloud Foundry application health check, make sure your applications block includes the health-check-type and health-check-http-endpoint properties:

```yaml
health-check-type: http
health-check-http-endpoint: /healthcheck
```

Also, the sequence property must include a reference to the healthcheck plugin, as shown here.

```yaml
"sequence": ["healthcheck", "oauth", "spikearrest"]
```

For more on health check, see Using [Application Health Checks](https://docs.cloudfoundry.org/devguide/deploy-apps/healthchecks.html).

The following describes the manifest properties:



|Variable	| Description |
|---------|-------------|
|APIGEE_MICROGATEWAY_CONFIG_DIR	| Location of your Apigee Microgateway configuration directory. |
|APIGEE_MICROGATEWAY_CUST_PLUGINS	| Location of your Apigee Microgateway plugins directory. |
|APIGEE_MICROGATEWAY_PROCESSES |	The number of child processes that Apigee Microgateway should start. If your Microgateway performance is poor, setting this value higher might improve it. |
|APIGEE_MICROGATEWAY_CUSTOM |	“sequence” corresponds to the sequence order in the microgateway yaml file (this will be added on to the end of any current sequence in the microgateway yaml file with duplicates removed). </br>“policies” correspond to any specific configuration needed by a plugin; for instance, “oauth” has the ‘"allowNoAuthorization": true configuration. These policies will overwrite any existing policies in the microgateway yaml file and add any that do not yet exist.|


### 5. Save the edited file.

## Step 3 (Optional): Install Apigee Edge Microgateway and Cloud Foundry App

Here, you install Apigee Edge Microgateway and your Cloud Foundry app to the same Cloud Foundry container.

### 1. Step not required during API Jam as these files are provided under resources folder. Execute for generating key and secrets. [Install and configure Apigee Edge Microgateway.](https://docs.apigee.com/api-platform/microgateway/2.5.x/installing-edge-microgateway.html)

## Step 4: Create config directory and copy Apigee Microgateway config file from resources

In this step we will copy microgateway config file under under Microgateway-Coresident resources folder to config folder`amer-api-partner19-test-config.yaml`. Note APIGEE_MICROGATEWAY_CONFIG_DIR reference in Manifest.yaml file to config folder. 

```bash
$ mkdir config
$ cp ~/tools/git/apijam/Labs/PivotalJam/Lab\ 3\ -\ Proxy\ a\ CF\ app\ using\ Edge\ Microgateway\ \(Microgateway-Coresident\ plan\)/resources/amer-api-partner19-test-config.yaml ./config
```

## Step 5: Check Edge Microgateway and Cloud Foundry App ports

### 1. Check Edge Microgateway listening on port 8080
As per [PCF requirements Applications should listen on port `8080`](https://docs.run.pivotal.io/devguide/deploy-apps/routes-domains.html#http-vs.-tcp-routes). Therefore, Edge Microgateway will frontend our Cloud Foundry app, which listens on port `8081`.

To confirm this is the case, run the following commands on the MG config file:

```bash
$ cat config

cat config/amer-api-partner19-test-config.yaml
edge_config:
edgemicro:
  port: 8080
```

**edgemicro/port is effectively listening on 8080. IMPORTANT: by default MG config file uses port 8000. So, make sure to make changes accordingly to PCF requirements.**

### 2. Check Cloud Foundry App Port running on port 8081
Ensure that your Cloud Foundry app isn’t running on port 8080, nor on the port specified by the PORT environment variable.

```
$ cat server.js

var port = 8081

var webServer = app.listen(port, function () {
    console.log('Listening on port %d', webServer.address().port)
```

This looks good. Cloud Foundry app listens on port 8081.

## Step 6: Push the Cloud Foundry App to your Cloud Foundry Container

In this step Node.js will be pushed to PCF under 
```bash
cf apigee-push -u -v

Do you plan on using this application with the "microgateway-coresident" plan? [y/n]
Specific name of application to push [optional]:

0 of 1 instances running, 1 starting
0 of 1 instances running, 1 crashed
Error restarting application: Start unsuccessful
```

```
**If deployment fails try using switch `cf apigee-push -u -v` to ignore healthcheck temporarily.**
```

You’ll be prompted for the following:

|Argument	|Description|
|---------|-----------|
|Plan type confirmation	|Specify whether you’re using the microgateway-coresident plan.|
|Path to your Java archive |	If your app is a Java app, specify the path to its .jar file.|
|Path to the configuration directory with Microgateway .yaml file |	Location of your Apigee Microgateway configuration directory.|
|Path to the configuration directory with custom plugins |Location of your Apigee Microgateway plugins directory.|

It's okay to see that CF application didn't start successfully. This is happening because the App listens on port 8081, not port 8080 and we haven't installed Apigee Edge Microgateway yet, which actually listens on port 8080.

## Step 7: Bind the Cloud Foundry App to the Service Instance

In this step, you bind a Cloud Foundry app to the Apigee service instance you created. The apigee-bind-mgc command creates the proxy for you and binds the app to the service.

Each bind attempt requires authorization with Apigee Edge, with credentials passed as additional parameters to the apigee-bind-mgc command. You can pass these credentials as arguments of the apigee-bind-mgc command or by using a bearer token.

### 1. If you’re using a bearer token to authenticate with Apigee Edge, get or update the token using the Apigee SSO CLI script. 

(If you’re instead using command-line arguments to authenticate with username and password, specify the credentials in the next step.)

Another option for Cloud, although it should be your last resource if `Edge Script (approach below)`, and `acurl` fail is to [use the Apigee Management API to get a token](https://apidocs.apigee.com/api-reference/content/using-oauth2-security-apigee-edge-management-api#usingtheapitogettokens).

#### Download the Apigee Edge scripts:

```bash
$ curl https://login.apigee.com/resources/scripts/sso-cli/ssocli-bundle.zip -o "ssocli-bundle.zip"
```
#### Unzip the ssocli-bundle.zip file. 
This includes get_token, a script that gets or updates a token that you use to authenticate with your Apigee Edge organization. You need this token to bind the Apigee Edge route service to your app.

```bash
$ tar xvf ssocli-bundle.zip
```

#### Create a .sso-cli directory in your user directory:

$ mkdir ~/.sso-cli
Use the get_token script to create a token. When prompted, enter the Apigee Edge username and password you use to log in to your organization.

$ ./get\_token
The get_token script writes the token file into ~/.sso-cli. For more about get_token, see the Apigee documentation.

#### Bind the app to the Apigee service instance with the apigee-bind-mgc command.

Use the command without arguments to be prompted for argument values. To use the command with arguments, see the command reference at the end of this topic. For help on the command, type cf apigee-bind-mgc -h.

```bash
$ cf apigee-bind-mgc --app CF_APP_NAME --service CF_SERVICE_INSTANCE_NAME --apigee_org APIGEE_ORG --apigee_env APIGEE_ENV --edgemicro_key EDGEMICRO_KEY --edgemicro_secret EDGEMICRO_SECRET --target_app_route TARGET_APP_ROUTE --target_app_port 8081 --action 'proxy bind' --user APIGEE_USER_NAME --pass APIGEE_PASSWORD
```

This command should return `OK`. If it returns and error, try again until you get it.


**Tip: to troubleshot `cf` commands use `-v` for verbose output.**

You’ll be prompted for the following:

|Argument |	Description |
|---------|-------------|
|Apigee username	| Apigee user name. **Provided by instructor above.** Not used if you pass a bearer token with the –bearer argument.|
|Apigee password |	Apigee password. **Provided by instructor above.** Not used if you pass a bearer token with the –bearer argument.|
|Action to take	| Required. **Creates and binds (Go router registration) proxy 'proxy bind'. Don't forget single quotes.** proxy to generate an API proxy; bind to bind the service with the proxy; proxy bind to generate the proxy and bind with a single command.|
|Apigee environment (apigee_env)	|Required. **Provided by instructor above.** The Apigee environment where your proxy should be deployed.|
|Apigee organization (apigee_org)	| Required. **Provided by instructor above.**  The Apigee organization where your proxy should be created.|
|Application to bind to (app)	|Required. **Retrieve using `cf apps` use value from name column** Name of the Cloud Foundry application to bind to.|
|Microgateway key	(edgemicro_key) | Required. **Provided by instructor.** Your Apigee Edge Microgateway key.|
|Microgateway secret (edgemicro_secret)	| Required. **Provided by instructor** Your Apigee Edge Microgateway secret.|
|Service instance name to bind to (service)	| Required. **Retrieve using `cf services`** Name of the Apigee service to bind to.|
|Target application port	| Required. **Typically 8081 as per step above after `$ cat server.js` to inspect port on Node.js app.** Port for your Cloud Foundry app. This may not be 8080 nor the PORT environment variable.|
|Target application route (target_app_route)	| Required. **Retrieve using `cf apps` and copy url. Do not include protocol (http or https)** The URL for your Cloud Foundry app. This will be the suffix of the proxy created for you through the bind command.|

Example:
```bash
cf apigee-bind-mgc --app dz-target-nodejs-app2 --service dz-apigee-mg-coresident-service-instance --apigee_org amer-api-partner19 --apigee_env test --edgemicro_key 2d99ace69294c8f791404a7dd735a83b4ce08545f9f2da2cb736a7b9019989c5 --edgemicro_secret b4eb6dea5cb19ca6c784d3345c1630d3698ab2ef8041cf7843d191f3a5d5de66 --target_app_route dz-target-nodejs-app2.apps.apigee-demo.net --target_app_port 8081 --action 'proxy bind' --user sandeepmuru+pivotal+labuser5@google.com --pass Apigee123

cf apigee-push -app dz-target-nodejs-app2 -service dz-apigee-mg-coresident-service-instance --apigee_org amer-api-partner19 --apigee_env test --edgemicro_key 2d99ace69294c8f791404a7dd735a83b4ce08545f9f2da2cb736a7b9019989c5 --edgemicro_secret b4eb6dea5cb19ca6c784d3345c1630d3698ab2ef8041cf7843d191f3a5d5de66 --target_app_route dz-target-nodejs-app2.apps.apigee-demo.net --target_app_port 8081 --action 'proxy bind' --user sandeepmuru+pivotal+labuser5@google.com --pass Apigee123
```


### Step 8: Deploy Manually the API Proxy to the environment

Verify that proxy with proxy name edgemicro_{APP_NAME.PCF_DOMAIN} create in Edge has been deployed successfuly on the corresponding environment through [https://apigee.com/edge](https://apigee.com/edge). Remember to use Apigee username and password provided by instructor.

![alt text](./images/apigee_edge_test_deployment.png "Api Proxy deployment successful")

### Step 9: Test the App and the Bindin

Once you’ve bound your app’s path to the Apigee service (creating an Apigee proxy in the process), you can try it out with the sample app.

From a command line run the curl command you ran earlier to make a request to your Cloud Foundry app you pushed, such as:

```bash
$ cf apps

name                    requested state   instances   memory   disk   urls
dz-target-nodejs-app2   started           1/1         1G       1G     dz-target-nodejs-app2.apps.apigee-demo.net
```

```bash
$ curl http://dz-target-nodejs-app2.apps.apigee-demo.net/hello/Diego
Response:
200 OK
{"hello":"hello Diego"}
```

The new proxy is just a pass-through, but it is now ready for you or someone on your team to add policies to define security, traffic management, and more.