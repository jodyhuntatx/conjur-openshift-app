# conjur-openshift-app

You are encouraged to clone this repo to your development machine.

## Methods of Accessing Secrets
A major design goal of Conjur is to minimize the amount of code an application requires in order to securely retrieve secrets. This OpenShift application deployment example shows two simple ways an application in OpenShift can access secrets stored in Conjur:
 - with Conjur's REST API
   - https://github.com/jodyhuntatx/conjur-openshift-app/blob/master/build/webapp/webapp_v4.sh#L12
 - with Summon, an open source tool that injects secrets as environment variables
   - https://github.com/jodyhuntatx/conjur-openshift-app/blob/master/build/webapp/webapp_summon.sh

Both methods of secrets retrieval are demonstrated by this script:
 - https://github.com/jodyhuntatx/conjur-openshift-app/blob/master/4_retrieve_secret.sh

## Application Image
Both examples are provided as webapp.sh and webapp_summon.sh respectively. The application container is built on a base Ubuntu 16.04 image that has Summon, jq and other utilities already installed. The "webapps" are actually just bash shell scripts that are copied into the base image and run only when called.

## Pod Deployment
The application is deployed in a pod that automatically authenticates to the Conjur service. The pod has two containers: the application container and the Conjur authenticator client container. The authenticator client automates a SPIFFE-compliant authentication process  (https://spiffe.io) to establish the identity of the pod. On startup, once the Conjur authn-k8s service running in the Follower validates the pod is running in a whitelisted Project, the authenticator client writes a Conjur access token to a shared memory volume accessible to the application. The authenticator greatly simplifies bootstrapping the application's identity securely, without resorting to cached tokens or credentials.

The Conjur access token is "proof of authentication" and is only valid for 8 minutes (the default, configurable) and is not persistent. The application can use the access token in two ways:
 - Directly: in REST API calls to Conjur. 
 - Indirectly: the Summon client uses the access token to retrieve secret values and injects them into environment variables that exist only in the application's sub-shell.
By default, the authenticator client container continuously runs as a sidecar and refreshes the access token every 6 minutes. It can also be deployed as an init container.

The pod deployment configuration template is here: 
 - https://github.com/jodyhuntatx/conjur-openshift-app/blob/master/manifests/deploy-template.yaml

The pod is deployed with this script which instantiates the values between double braces in the dc template:
 - https://github.com/jodyhuntatx/conjur-openshift-app/blob/master/3_deploy_test_app.sh

## Experimentation Encouraged
The app container entrypoint is just a sleep infinity loop. You can exec into the app with ./exec-into-app.sh. To encourage real-time experimentation, the apps are editable in the container with vim. You can also run Summon from the container's bash prompt. 

Conjur documentation: https://developer.conjur.net
Summon documentation: https://cyberark.github.io/summon/

## Administration
Whitelisting new projects and creating new secrets are accomplished with Conjur Policies. Conjur Policy authoring is an advanced topic and in this environment, policies initially will be edited and loaded by an administrator. However in general, policy loading is only restricted outside of Dev.

## Directory contents:

 - common.env - configuration env vars & utility functions
 - 1_init_app_cert.sh - initializes TLS cert for Conjur service
 - 2_build_and_push_images.sh - builds and pushes app image
 - 3_deploy_test_app.sh - deploys app pod w/ authenticator
 - 4_retrieve_secret.sh - runs secrets retrieval examples
 - exec-into-app.sh - for shell prompt in app container
 - manifests - deployment yaml (template and instantiated)
 - start - runs all scripts
 - stop - deletes deployed app pods
 - build/webapp - build directory for the webapp example application
 - build/baseimage - build directory for the application base image.
