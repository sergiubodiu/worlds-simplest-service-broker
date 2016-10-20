World's Simplest Service Broker
===============================

If you have a shared service such as Hadoop where all applications will ultimately bind to the same service with the same credentials then you have found the service broker for you - the World's Simplest Service Broker for Cloud Foundry.

You configure it with a simple environment variable `CREDENTIALS` (the same JSON object that will be returned for all service bindings); and then register it as a service broker.

All services created and all service bindings will be given the same set of credentials. Definitely the simplest thing that could work.

As the admin of a service sharing it via a service broker - see section [Deploy to Cloud Foundry](#deploy-to-cloud-foundry) for setup.

As a user of the broker:

```
cf cs myservice some-service-name
cf bs my-app some-service-name
cf restage my-app
```

Why not "user-provided services"?
---------------------------------

Cloud Foundry includes "user-provided services" (see `cf cups` in the CLI) for easy registration of existing external service credentials.

The restriction for "cups" is that it is limited to the Space into which it was registered. For each organization/space, the `cf cups` command needs to be run. That is, when you create a new space, it does not immediately have access to the credentials for the service.

Instead, with the World's Simplest Service Broker you can make the credentials easily and instantly available to all organizations' spaces.

Build locally
-------------

```
godep get
export BASE_GUID=$(uuid)
export CREDENTIALS='{"port": "4000", "host": "1.2.3.4"}'
export SERVICE_NAME=myservice
export SERVICE_PLAN_NAME=shared
export TAGS=simple,shared
worlds-simplest-service-broker
```

Deploy to Cloud Foundry
-----------------------
latested PCFdev does not support go 1.7. if you see this message

cf create-buildpack go_buildpack /Users/sergiu/Downloads/go_buildpack-cached-v1.7.13.zip

```
It looks like you're trying to use go 1.7.
Unfortunately, that version of go is not supported by this buildpack.
The versions of go supported in this buildpack are:
- 1.6.3
- 1.6.2
- 1.5.4
- 1.5.3
If you need further help, start by reading: http://github.com/cloudfoundry/go-buildpack/releases.
```
you need to download the latest go buildpack from https://github.com/cloudfoundry/go-buildpack/releases/tag/v1.7.13 and update the buildpack on your environment

```
cf create-buildpack go_buildpack /Users/sergiu/Downloads/go_buildpack-cached-v1.7.13.zip 1
```

After troubleshooting the buildpack start deploying the application

```
export SERVICE=myservice
export APPNAME=$SERVICE-broker
cf push $APPNAME --no-start -m 128M -k 256M
cf set-env $APPNAME BASE_GUID $(uuid)
cf set-env $APPNAME CREDENTIALS '{"port": "4000", "host": "1.2.3.4"}'
cf set-env $APPNAME SERVICE_NAME $SERVICE
cf set-env $APPNAME SERVICE_PLAN_NAME shared
cf set-env $APPNAME TAGS simple,shared
cf env $APPNAME
cf start $APPNAME
```

To register the service broker (as an admin user):

```
export SERVICE_URL=$(cf app $APPNAME | grep urls: | awk '{print $2}')
cf create-service-broker $SERVICE admin admin https://$SERVICE_URL
cf enable-service-access $SERVICE
cf logs $APPNAME
```

To clean up the service

```
cf disable-service-access $SERVICE
cf delete-service-broker $SERVICE -f
cf delete $APPNAME -f
```

To change the credentials being offered for bindings:

```
export SERVICE=myservice
export APPNAME=$SERVICE-broker
cf set-env $APPNAME CREDENTIALS '{"port": "4000", "host": "1.2.3.4"}'
cf restart $APPNAME
```

Each application will need rebind and restart/restage to get the new credentials.

### Basic Authentication

Set the environment variables `AUTH_USER` and `AUTH_PASSWORD` to enable basic authentication for the broker.

This is useful to prevent unauthorized access to the credentials exposed by the broker (e.g. by somebody doing a `curl -X PUT http://$SERVICE_URL/v2/service_instances/a/service_bindings/b`).

To do so (of course, change `secret_user` and `secret_password` to something more secret):

```
cf set-env $APPNAME AUTH_USER secret_user
cf set-env $APPNAME AUTH_PASSWORD secret_password
cf restart $APPNAME
cf update-service-broker $SERVICE secret_user secret_password https://$SERVICE_URL
```

### syslog_drain_url

The broker can advertise a `syslog_drain_url` endpoint with the `$SYSLOG_DRAIN_URL` variable:

```
cf set-env $APPNAME SYSLOG_DRAIN_URL 'syslog://1.2.3.4:514'
```

### Dashboard

Each service instance is assigned the same dashboard URL - `/dashboard`.

### Image URL

Adding image url to service broker

```
cf set-env $APPNAME IMAGE_URL '<image url>'
```
