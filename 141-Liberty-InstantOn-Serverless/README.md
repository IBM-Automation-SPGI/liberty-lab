# Hands-on Open Liberty InstantOn Lab

[Return to main labs page](../README.md)

In this lab, you will get to build a simple Open Liberty application and run it in a container, which you will then get to deploy in multiple ways - locally and on Kubernetes.

The application will be run in 2 modes: standard and with InstantOn. You will get to verify the significant improvement in start time that InstantOn provides.

This lab will take you through the steps required to build InstantOn images, as well as how to deploy images onto the OpenShift Cloud Platform.

Note that in many of the commands listed below, we have supplied a file to perform the command. You can either choose to type the commands yourself or simply run the script.

This lab requires you start at least one terminal session, and start the Firefox browser.

As part of this lab, the Knative service needs to installed and configured. This will typically be performed by the instructor before the lab and be available for all to use. If you are the instructor, or if you would like to see the steps involved, please refer to the [Knative setup instruction](https://github.com/rhagarty/techxchange-knative-setup).

# Steps:

1. [Initial lab setup](#1-initial-lab-setup)
1. [Build and run the application locally](#2-build-and-run-the-application-locally)
1. [Push the images to the OpenShift registry](#3-push-the-images-to-the-openshift-registry)
1. [Enhance the OpenShift Cloud Platform (OCP) environment](#4-enhance-the-openshift-cloud-platform-ocp-environment)
1. [Deploy the applications to OCP](#5-deploy-the-applications-to-ocp)

## 1. Initial lab setup

### InstantOn Lab VM setup

Please reserve the InstantOn lab on TechZone in order to set up the VM.

### OCP Server VM setup

> **NOTE**: You will not need to perform this step if your event organizer has set up the OCP server for you.

Please follow the instructions mentioned in the InstantOn lab description to set up the OCP server.

### Login as root

After provisioning the InstantOn Lab VM and ensuring its readiness, utilize the provided NoVNC link to access the lab's desktop. Then, proceed to log in as root from the lab's terminal.

```bash
su
```

Use as root password: `IBMDem0s!`

### Upgrade OpenJDK

```bash
wget https://github.com/ibmruntimes/semeru21-binaries/releases/download/jdk-21.0.2%2B13_openj9-0.43.0/ibm-semeru-open-21-jdk-21.0.2.13_0.43.0-1.x86_64.rpm
yum localinstall ibm-semeru-open-21-jdk-21.0.2.13_0.43.0-1.x86_64.rpm 
alternatives --config java
vi ~/.bashrc
	export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.2.0.13-1.el9.x86_64
	PATH=$PATH:JAVA_HOME/bin
source ~/.bashrc
```

### Clone the application from GitHub

```bash
cd /home/ibmuser/Lab-InstantOn
git clone https://github.com/grvuolo/liberty-spgi-lab.git
cd liberty-spgi-lab/141-Liberty-InstantOn-Serverless
```

> **NOTE**: If you prefer to use an IDE to view the project, you can run VSCode with admin privileges using the following command: 
> ```bash
> sudo code --no-sandbox --user-data-dir /home/techzone
> ```

### Login to the OpenShift console, using the following URL:

A shared OpenShist cluster has been provisioned at the following URL: https://console-openshift-console.apps.67a3346aadf42b5f8ef28cdc.eu1.techzone.ibm.com/

go to `IBM Demo`

username: user[x]
password: passw0rd

### Login to the OpenShift CLI [IF NEEDED]

> **NOTE**: You will not need to perform this step if you completed the Semeru Cloud Compiler lab. To verify, you can use `oc whoami` to determine if you are already logged into the CLI as `kubeadmin`.

From the OpenShift console UI, click the username in the top right corner, and select `Copy login command`.

![ocp-cli-login](images/ocp-cli-login.png)

Press `Display Token` and copy the `Log in with this token` command.

![ocp-cli-token](images/ocp-cli-token.png)

Paste the command into your terminal window. You should receive a confirmation message that you are logged in.

## 2. Build and run the application locally

### Package the application

First ensure that you are in the `liberty-spgi-lab/141-Liberty-InstantOn-Serverless` directory, then run `mvn package` to build the application.

```bash
mvn package
```

### Change docker to podman

```bash
mv /usr/bin/docker /usr/bin/docker_backup
cp /usr/bin/podman.backup /usr/bin/podman
```

### Build the application image

Run the provided script:

```bash
sudo ./build-local-without-instanton.sh
```

Or type the following command:

```bash
sudo podman build -t dev.local/getting-started .
```

> **NOTE**: The Dockerfile is using a slim version of the Java 21 Open Liberty UBI.
> 
> `FROM icr.io/appcafe/open-liberty:kernel-slim-java21-openj9-ubi-minimal`
> 
> If you encounter any issues while executing this step, please refer to the [troubleshooting notes](#error-when-building-the-application-imagewithout-instanton) provided.

### Run the application in a container

Run the provided script:

```bash
./run-local-without-instanton.sh
```

Or type the following command: 

```bash
podman run --name getting-started --rm -p 9080:9080 dev.local/getting-started
```

When the application is ready, you will see the following message:

```bash
[INFO] [AUDIT] CWWKF0011I: The server defaultServer is ready to run a smarter planet.
```

Note the amount of time Open Liberty takes to report it has started (typically 3-5 seconds).

Check out the application by pointing your browser at http://localhost:9080/dev. 

To stop the running container, press `CTRL+C` in the command-line session where you ran the `podman run` command.

### Update the Dockerfile to use InstantOn

In order to convert this image to use InstantOn, modify the `Dockerfile` by adding the following line to the bottom of the file. 

```bash
RUN checkpoint.sh afterAppStart
```

You can use this command: 

```bash
echo RUN checkpoint.sh afterAppStart >> Dockerfile
```

This command will perform the following actions:
1. Run the application
1. Take a checkpoint after the application code has loaded
1. Stop the application

Note that there are 2 checkpoint options:

`beforeAppStart`: Perform a checkpoint after inspecting the application and parsing the application annotations, metadata, etc, but BEFORE any application code is run.  

`afterAppStart`: Perform a checkpoint after running any application code that has to run before the JVM can report that the app has started and is ready to receive requests. This checkpoint is inappropriate if your application start code does any of the following:
* Accessing a remote resource, such as a database
* Reading configuration that is expected to change when the application is deployed
* Starting a transaction

### Build the application image with the InstantOn checkpoint

Run the provided script:

```bash
sudo ./build-local-with-instanton.sh
```

Or type the following command: 

```bash
sudo podman build \
  -t dev.local/getting-started-instanton \
  --cap-add=CHECKPOINT_RESTORE \
  --cap-add=SYS_PTRACE\
  --cap-add=SETPCAP \
  --security-opt seccomp=unconfined .
```

> **IMPORTANT**: We need to add several Linux capabilities that are required when the checkpoint image is built.

You should see the following in the build output:

```bash
...
Performing checkpoint --at=afterAppStart
...
```

### Run the InstantOn application in a container

Run the provided script:

```bash
./run-local-with-instanton.sh
```

Or type the following command: 

```bash
podman run \
 --rm \
 --cap-add=CHECKPOINT_RESTORE \
 --cap-add=SETPCAP \
 --security-opt seccomp=unconfined \
 -p 9080:9080 \
 dev.local/getting-started-instanton
```

> **IMPORTANT**: We need to add several Linux capabilities and security options so that the container has the correct privileges when running.

When the application is ready, you will see the following message:

```bash
[INFO] [AUDIT] CWWKF0011I: The server defaultServer is ready to run a smarter planet.
```

Note the startup time and compare to the version without InstantOn. You should see a startup time in the 300 millisecond range - a 10x improvement!

Check out the application by pointing your browser at http://localhost:9080/dev. 

To stop the running container, press `CTRL+C` in the command-line session where you ran the `podman run` command.

## 3. Push the images to the OpenShift registry

### Create the namespace and set it as the default

> **NOTE**: If you are working on a cluster that is shared with others, a namespace has already been created for you, if your user is *user1* then your namespaces is *instantonlab-1*. Change [Your initial] accordingly.

```bash
export CURRENT_NS=user[User Number]-ns
```
> **NOTE**: your namespace has already been created

It should be the default project (and the only one that you have access), but ensure you are working in that namespace by executing:

```bash
oc project $CURRENT_NS
```

### Enable the default registry route in OpenShift to push images to its internal repos [not necessary when using a shared OpenShift]

> **NOTE**: You will not need to perform this step as the registry is already exposed

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```

### Log Podman into the OpenShift registry server [IF NEEDED]


> **NOTE**: this step is not necessary as the secret is already provided by the instructor and access to registry is available with the login to OCP

First we need to get the `TOKEN` that we can use to get the password for the registry.

```bash
oc get secrets -n openshift-image-registry | grep cluster-image-registry-operator-token
```

Take note of the `TOKEN` value, as you need to substitute it in the following command that sets the registry password.

```bash
export OCP_REGISTRY_PASSWORD=eyJhbGciOiJSUzI1NiIsImtpZCI6IlpwUVFqYVRFb1phdVhSUExPVmFrdXhjR1R3TmR0TU4yd216aU5UNHdEOEEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJvcGVuc2hpZnQtaW1hZ2UtcmVnaXN0cnkiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoiY2x1c3Rlci1pbWFnZS1yZWdpc3RyeS1vcGVyYXRvci10b2tlbi02ZDg2MiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJjbHVzdGVyLWltYWdlLXJlZ2lzdHJ5LW9wZXJhdG9yIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMDQwZDVhM2YtYjg1Ny00MWI2LTlhZDYtOTllOGI1YmY4ZTgwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Om9wZW5zaGlmdC1pbWFnZS1yZWdpc3RyeTpjbHVzdGVyLWltYWdlLXJlZ2lzdHJ5LW9wZXJhdG9yIn0.lVf_JvbJapMfZPHog-HqUJa54pqbJwxztNZFk_7ckhPrds0yghGbl5_EhDusdC6ovHimEeWs8UGZn4f4xRhEDcu_5ELm4QKgMNeev3oIeM-9y2Xnk-YdY7XHGGRFXET6UpkoPd5HAud0hj6wI1moxXkPG75Vy6J98L_9wl4yYansZo3Y6UHXary6ZOvpB-Fnf1yJW5ujMgrdBWjW6-Z4IlTF7UeKt9HjNTKADTskmVe-ngR1yeRMQB313Ob2btiCNHyRKHj0dV8_xzK5gCLQoEdbcInUkjLs5pcyOxMQ8TnTNlR1vnUby1qbeneCs2Ibd2xLz-3nE55yEJV88MLIZw
```

Now set the OpenShift registry host value.

```bash
export OCP_REGISTRY_HOST=default-route-openshift-image-registry.apps.67a3346aadf42b5f8ef28cdc.eu1.techzone.ibm.com
```

Finally, we have the values needed for podman to login into the OpenShift registry server.

```bash
podman login -p $OCP_REGISTRY_PASSWORD -u user[number] $OCP_REGISTRY_HOST --tls-verify=false
```

### Tag and push your 2 application images to the OpenShift registry

Use `podman images` to verify your 2 local images. The image names should be:

* dev.local/getting-started
* dev.local/getting-started-instanton

Now tag and push them to the OpenShift registry:

```bash
# base application image
podman tag dev.local/getting-started:latest $(oc registry info)/$(oc project -q)/getting-started:1.0-SNAPSHOT
podman push $(oc registry info)/$(oc project -q)/getting-started:1.0-SNAPSHOT --tls-verify=false
```
```bash
# InstantOn application image
podman tag dev.local/getting-started-instanton:latest $(oc registry info)/$(oc project -q)/getting-started-instanton:1.0-SNAPSHOT
podman push $(oc registry info)/$(oc project -q)/getting-started-instanton:1.0-SNAPSHOT --tls-verify=false
```

### Verify the images have been pushed to the OpenShift image repository

```bash
oc get imagestream
```

## 4. Enhance the OpenShift Cloud Platform (OCP) environment

> **NOTE**: this step is not necessary as open liberty operator is available. Trust the instructor


Perform the following steps to enhance OCP to better manage OCP services, such as Knative, which provides serverless or scale-to-zero functionality. 

The Liberty Operator provides resources and configurations that make it easier to run Open Liberty applications on OCP. To verify if the Liberty Operator is available on the server side, please use the following command:

```bash
oc get crd openlibertyapplications.apps.openliberty.io openlibertydumps.apps.openliberty.io openlibertytraces.apps.openliberty.io
```

You should see the following output: 

![verify-liberty-operator](images/verify-liberty-operator.png)


### Apply the Liberty Operator to your namespace [not necessary when using a shared OpenShift]

> **NOTE**: this step is not necessary as open liberty operator is available to your namespace already. Trust the instructor


```bash
OPERATOR_NAMESPACE=instantonlab-[Your initial]
WATCH_NAMESPACE=instantonlab-[Your initial]
##
curl -L https://raw.githubusercontent.com/OpenLiberty/open-liberty-operator/main/deploy/releases/1.3.1/kubectl/openliberty-app-operator.yaml \
      | sed -e "s/OPEN_LIBERTY_WATCH_NAMESPACE/${WATCH_NAMESPACE}/" \
      | oc apply -n ${OPERATOR_NAMESPACE} -f -
```

### Verify the installation of the Cert Manager [not necessary when using a shared OpenShift]

The Cert Manager adds certifications and certification issuers as resource types to Kubernetes

> **NOTE**: this step is not necessary as open liberty operator is available to your namespace already. Trust the instructor

```bash
oc get deployments -n cert-manager
```

You should see the following output: 

![verify-cert](images/verify-cert.png)


### Verify the OpenShift serverless operator is installed and ready

```bash
oc get csv
```

> **NOTE**: If you encounter any issues while executing this step, please refer to the [troubleshooting notes](#error-when-verifying-the-openshift-serverless-operator-is-installed-and-ready) provided.

You should see the following output:

![ocp-serverless](images/ocp-serverless.png)

### Verify the Knative service is ready [not necessary when using a shared OpenShift]

```bash
oc get knativeserving.operator.knative.dev/knative-serving -n knative-serving --template='{{range .status.conditions}}{{printf "%s=%s\n" .type .status}}{{end}}'
```

If the Knative service has been setup and added to the capability, your output should match the following:

![ocp-knative](images/ocp-knative.png)

> **NOTE**: If the output does not match the expected output or if there are any other issues, please contact your instructor.
>
> To learning more about setup Knative service please refer to our [knative setup instruction](https://github.com/rhagarty/techxchange-knative-setup)


### Verify the Knative containerspec-addcapabilities feature is enabled [not necessary when using a shared OpenShift]

To confirm whether the `containerspec-addcapabilities` is enabled, you can inspect the current configuration of `config-features` by executing the command 
> ```bash 
> oc -n knative-serving get cm config-features -oyaml | grep -c "kubernetes.containerspec-addcapabilities: enabled" && echo "true" || echo "false"
> ```

> **IMPORTANT**: If the command returns true, it indicates that the Knative 'containerspec-addcapabilities' feature is already enabled. Please skip the step regarding editing Knative permissions. However, if it returns false, please contact your instructor regarding this. 
In a production scenario, you may be required to enable `containerspec-addcapabilities` manually, please refer to our [knative setup instruction](https://github.com/rhagarty/techxchange-knative-setup) for further info. 

### Run the following commands to give applications the correct Service Account (SA) and Security Context Contraint (SCC) to run instantOn [not necessary when using a shared OpenShift]

```bash 
oc create serviceaccount instanton-sa-$CURRENT_NS
oc apply -f scc-cap-cr.yaml
oc adm policy add-scc-to-user cap-cr-scc -z instanton-sa-$CURRENT_NS
```

## 5. Deploy the applications to OCP

### Update the namespace in the deployment YAML files

> **NOTE**: Execute the following command to activate the script for updating the necessary YAML files with the project namespace you previously set (CURRENT_NS)

```bash
./searchReplaceNs.sh
```

### Deploy the base application

> **IMPORTANT**: Please ensure to fill in all `[Your initial]` fields with the namespace used in the creation step above before proceeding to apply the YAML file.

```bash
oc apply -f deploy-without-instanton.yaml
```

### Monitor the base application

```bash
oc get pods
```

Once the pod is running and displays a `POD NAME`, quickly take a look at the pod log to see how long the application took to start up.

```bash
oc logs <POD NAME>
```

> **NOTE**: Knative will stop the pod if it does not receive a request in the specified time frame, which is set in a configuration yaml file. For this lab, the settings are in the `serving.yaml` file, and currently set to 30 seconds (as shown below).

```bash
apiVersion: operator.knative.dev/v1beta1
kind: KnativeServing
metadata:
    name: knative-serving
    namespace: knative-serving
spec:
  config:
    autoscaler:
      scale-to-zero-grace-period: "30s"
      scale-to-zero-pod-retention-period: "0s"
```

### Deploy the application with InstantOn

> **IMPORTANT**: Please ensure to fill in all `[Your initial]` fields with the namespace used in the creation step above before proceeding to apply the YAML file.

```bash
oc apply -f deploy-with-instanton.yaml
```

Use the same `oc get pods` and `oc logs` commands as above to monitor the application.

Compare the start times of both applications and note how the InstantOn version again starts around 10x faster.

### Verify the applications running in the browser

To get the URL for the deployed applications, use the following command:

```bash
oc get ksvc
```

Check out each of the applications by pointing your browser at the listed URL.

### Visually test how long an idle application takes to re-start

As a visual test, do not click or refresh either application running in the browser for over 30 seconds. This will cause the knative service to stop the associated pod.

Use the `oc get pods` command to verify that both application pods are no longer running.

Now, one application at a time, click the refresh button on the application page to see how long it takes to refresh the content.

>**NOTE**: Knative itself can take several seconds to load from a cold start. Take this into account when determininig how long it takes for the application to refresh.

### (OPTIONAL) Stop and delete the deployed applications

```bash
oc delete -f deploy-without-instanton.yaml
oc delete -f deploy-with-instanton.yaml
```

## Troubleshooting

### Error when building the application image(without InstantOn)

  If you run into the following error when building the application image(without InstantOn):

  ```bash
  error running container: from /usr/bin/crun creating container for [/bin/sh -c configure.sh]: sd-bus call: Transport endpoint is not connected: Transport endpoint is not connected
  : exit status 1
  ERRO[0043] did not get container create message from subprocess: EOF 
  Error: building at STEP "RUN configure.sh": while running runtime: exit status 1
  ```

  Run the following command from your terminal window:

  ```bash
  sudo ./build-local-without-instanton.sh
  ```

### Error when verifying the OpenShift serverless operator is installed and ready

If you run into the following error when running `oc get csv `:

```bash
oc get csv
No resources found in instantonlab-[your initial] namespace.
```

Please wait a few more minutes and then try again. It should return the correct output.

**===== END OF LAB =====**

[Return to main labs page](../README.md)
