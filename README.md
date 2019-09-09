# OpenShift-Lab

This project is my Ansible Playbook to install OpenShift on my Hetzner server.

## Operating System install

Go to [access.redhat.com](https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.6/x86_64/product-software) and download the boot ISO image of the latest RHEL 7.

Upload this ISO image to any large file transfer such as [send.firefox.com](https://send.firefox.com) or [dl.free.fr](http://dl.free.fr/).

Go to your [Hetzner console](https://robot.your-server.de/server), select your server and book a KVM (**Support** > **Remote Console (KVM)** > **I would like to make an appointment**).
Choose a date, time and duration. For the duration, two hours should be enough.

In the message box, type something like:

```raw
Dear Hetzner Support team,

I would like to install RHEL 7 on my server. Could you please burn the following ISO image on a CD or prepare a USB Key accordingly for me ?

<Put the link to the ISO image here>

Many thanks for your help.

Best regards.
```

Click **Send Request**

At the specified timeframe, you should receive a mail containing the login details to connect to your KVM.

Open the KVM console. This is a Java applet, so make sure there is no security restriction on their execution.

Reboot your server using the **Ctrl+Ald+Delete** button.

When the bios shows up, press **<F11>** to enter the boot menu and boot from the CD or USB Key, according to the Hetzner instructions.

[![Hetzner install](https://img.youtube.com/vi/q-brW2_23Lo/0.jpg)](https://www.youtube.com/watch?v=q-brW2_23Lo)

In this video at 3:00, I configure in the **Installation Source** a repository that is hosted on another server.
It's just a web server that serves the content of the ISO image:

```sh
yum install lighttpd
systemctl start lighttpd
mount rhel-server-7.6-x86_64-dvd.iso /var/www/lighttpd/ -o loop,ro
```

You can verify that your setup is correct with:

```sh
curl http://localhost/.treeinfo
```

## Getting a public certificates with Let's encrypt

On the Ansible control node, install [lego](https://github.com/go-acme/lego):

```sh
brew install lego
```

Get a certificate for the wildcard domain as well as the master hostname:

```sh
GANDIV5_API_KEY=[REDACTED] lego -d openshift.itix.fr -d app.itix.fr -d '*.app.itix.fr' -a -m your.email@example.test --path $HOME/.lego --dns gandiv5 run
```

See [this guide](https://github.com/nmasse-itix/OpenShift-Examples/tree/master/Public-Certificates-with-Letsencrypt) for more details.

## Preparation

Register the server on RHN:

```sh
sudo subscription-manager register --name=openshift.itix.fr
sudo subscription-manager refresh
sudo subscription-manager list --available --matches '*Employee SKU*'
sudo subscription-manager attach --pool=8a85f9833e1404a9013e3cddf95a0599
```

Edit `/etc/sysconfig/network-scripts/ifcfg-eno1` and add:

```sh
NM_CONTROLLED="yes"
PEERDNS="yes"
DOMAIN="itix.fr"
```

## OpenShift Install

Create a file named `group_vars/OSEv3`, containing your secrets:

```sh
cat <<EOF > group_vars/OSEv3
---
# Generated on https://access.redhat.com/terms-based-registry/
oreg_auth_password: your.password.here
oreg_auth_user: '123|user-name'

openshift_additional_registry_credentials:
- host: registry.connect.redhat.com
  user: rhn-username
  password: rhn-password
  test_image: sonatype/nexus-repository-manager:latest

# see: https://github.com/nmasse-itix/OpenShift-Examples/tree/master/Login-to-OpenShift-with-your-Google-Account
openshift_master_identity_providers:
- name: RedHat
  login: true
  challenge: false
  kind: GoogleIdentityProvider
  clientID: your.client_id.apps.googleusercontent.com
  clientSecret: your.client_secret.here
  hostedDomain: redhat.com
EOF
```

Create a file named `group_vars/all/itix.yaml`, containing your global variables:

```sh
mkdir -p group_vars/all/
cat <<EOF > group_vars/all/itix.yaml
---
# The regular user account you created on your server
ansible_ssh_user: nicolas
EOF
```

Run the OpenShift install:

```sh
ansible-playbook -i prod.hosts playbooks/preparation.yml
ansible-playbook -i prod.hosts openshift-ansible/playbooks/deploy_cluster.yml
ansible-playbook -i prod.hosts playbooks/post-install.yml
```

## Deploy the Software Factory

### Red Hat SSO

```sh
oc new-project sso --display-name="Single Sign-On"
for resource in sso73-image-stream.json \
  sso73-x509-https.json \
  sso73-x509-postgresql-persistent.json
do
  oc replace -n openshift --force -f \
  https://raw.githubusercontent.com/jboss-container-images/redhat-sso-7-openshift-image/sso73-dev/templates/${resource}
done
oc -n openshift import-image redhat-sso73-openshift:1.0
oc policy add-role-to-user view system:serviceaccount:$(oc project -q):default

oc new-app --template=sso73-x509-postgresql-persistent --name=sso -p SSO_HOSTNAME=sso.app.itix.fr -p DB_USERNAME=sso -p SSO_ADMIN_USERNAME=admin -p DB_DATABASE=sso
oc delete route sso
oc create -f - <<EOF
  apiVersion: v1
  id: sso-https
  kind: Route
  metadata:
    annotations:
      description: Route for application's https service.
    labels:
      application: sso
    name: sso
  spec:
    host: sso.app.itix.fr
    tls:
      termination: reencrypt
    to:
      name: sso
EOF
```

### Jenkins

```sh
oc project factory --display-name="Software Factory"
oc new-app --template=jenkins-ephemeral --name=jenkins -p MEMORY_LIMIT=2Gi
oc set env dc/jenkins INSTALL_PLUGINS="configuration-as-code:latest,configuration-as-code-support:latest"
oc set env dc/jenkins JENKINS_OPTS=--sessionTimeout=86400
cat <<EOF > casc.yaml
jenkins:
  systemMessage: "Jenkins configured automatically by Jenkins Configuration as Code plugin\n\n"
unclassified:
  globalPluginConfiguration:
    # OpenShift Sync Plugin: list of all namespaces to watch for, separated by a space
    namespace: factory api-lifecycle
  microcksGlobalConfiguration:
    microcksInstallations:
    - microcksDisplayName: Microcks
      microcksApiURL: https://microcks.app.itix.fr/api
      microcksCredentialsId: microcks-serviceaccount
      microcksKeycloakURL: https://sso.app.itix.fr/auth/realms/microcks/
      disableSSLValidation: true
credentials:
  system:
    domainCredentials:
    - credentials:
      - usernamePassword:
          scope: SYSTEM
          id: microcks-serviceaccount
          description: Microcks service account
          username: microcks-serviceaccount
          password: '[REDACTED]'
EOF
oc create configmap jenkins-casc --from-file=casc.yaml
oc set volume dc/jenkins --add -m /casc/ --name=casc -t configmap --configmap-name=jenkins-casc
oc set env dc/jenkins CASC_JENKINS_CONFIG="/casc/"

oc delete route jenkins
oc create -f - <<EOF
  apiVersion: v1
  kind: Route
  metadata:
    annotations:
      haproxy.router.openshift.io/timeout: 4m
      template.openshift.io/expose-uri: http://{.spec.host}{.spec.path}
    name: jenkins
  spec:
    host: jenkins.app.itix.fr
    tls:
      insecureEdgeTerminationPolicy: Redirect
      termination: edge
    to:
      kind: Service
      name: jenkins
EOF

oc process -f https://raw.githubusercontent.com/microcks/microcks-jenkins-plugin/master/openshift-jenkins-master-bc.yml | oc create -f -
oc set triggers dc/jenkins --remove --from-image=openshift/jenkins:2
oc set triggers dc/jenkins --from-image=microcks-jenkins-master:latest -c jenkins
```

### Microcks

```sh
oc project factory
git clone https://github.com/microcks/microcks-ansible-operator.git
cd microcks-ansible-operator/
oc create -f deploy/crds/microcks_v1alpha1_microcksinstall_crd.yaml
oc create -f deploy/service_account.yaml
oc create -f deploy/role.yaml
oc create -f deploy/role_binding.yaml
oc create -f deploy/operator.yaml

oc replace -n factory -f - <<EOF
apiVersion: microcks.github.io/v1alpha1
kind: MicrocksInstall
metadata:
  name: microcks
spec:
  name: microcks
  version: "0.7.1"
  microcks:
    replicas: 1
    url: microcks.app.itix.fr
  postman:
    replicas: 1
  keycloak:
    install: false
    url: sso.app.itix.fr
    replicas: 1
  mongodb:
    install: true
    persistent: true
    volumeSize: 2Gi
    replicas: 1
EOF

oc create -f - <<EOF
kind: OAuthClient
apiVersion: v1
metadata:
  name: microcks
respondWithChallenges: false
secret: $(uuidgen)
redirectURIs:
- https://sso.app.itix.fr/auth/realms/microcks/broker/openshift-v3/endpoint
EOF
oc get oauthclient microcks -o yaml
```

### Apicurio

Deploy Apicurio:

```sh
oc project factory
oc create -f https://raw.githubusercontent.com/Apicurio/apicurio-studio/master/distro/openshift/apicurio-template.yml
oc create -f https://raw.githubusercontent.com/Apicurio/apicurio-studio/master/distro/openshift/apicurio-postgres-template.yml
oc new-app --template=apicurio-postgres -p GENERATED_DB_USER=apicurio -p GENERATED_DB_PASS=apicurio -p DB_NAME=apicurio
oc new-app --template=apicurio-studio -p DB_USER=apicurio -p DB_PASS=apicurio -p DB_NAME=apicurio -p KC_REALM=apicurio -p AUTH_ROUTE=sso.app.itix.fr -p WS_ROUTE=apicurio-studio-ws.app.itix.fr -p API_ROUTE=apicurio-studio-api.app.itix.fr -p UI_ROUTE=apicurio-studio.app.itix.fr -p KC_USER=dummy -p KC_PASS=dummy
```

Configure Microcks integration:

```sh
oc set env dc/apicurio-studio-api APICURIO_MICROCKS_API_URL=https://microcks.app.itix.fr/api APICURIO_MICROCKS_CLIENT_ID=microcks-serviceaccount APICURIO_MICROCKS_CLIENT_SECRET=[REDACTED]
oc set env dc/apicurio-studio-ui APICURIO_UI_FEATURE_MICROCKS=true
```

Configure Red Hat SSO for Apicurio:

```sh
curl -o realm-template.json https://raw.githubusercontent.com/Apicurio/apicurio-studio/master/distro/openshift/auth/realm.json
m4 -DAPICURIO_UI_URL=https://apicurio-studio.app.itix.fr realm-template.json > realm-to-be-imported.json
```

- Create a new realm named "apicurio" by importing the file `realm-to-be-imported.json`
- Go to "Identity Provider" and add an "openshift-v3" IdP
- Create the corresponding OpenShift OAuthClient

```sh
oc create -f - <<EOF
kind: OAuthClient
apiVersion: v1
metadata:
  name: apicurio
respondWithChallenges: false
secret: $(uuidgen)
redirectURIs:
- https://sso.app.itix.fr/auth/realms/apicurio/broker/openshift-v3/endpoint
EOF
oc get oauthclient apicurio -o yaml
```

- Go to "Authentication" and configure the Identity Provider Redirector to use the openshift-v3 IdP
- Go to "Realm Settings" > "Themes" and set the themes to "rh-sso"
- Configure [Account Linking](https://apicurio-studio.readme.io/docs/setting-up-keycloak-for-use-with-apicurio#section-7-configure-keycloak-for-account-linking)

### Nexus

```sh
oc project factory
oc create secret docker-registry partner-registry --docker-username=your.rhn.login --docker-password=your.rhn.password --docker-email=your.email@example.test --docker-server=registry.connect.redhat.com
oc secrets link default partner-registry --for=pull
oc import-image nexus-repository-manager:latest --confirm --scheduled --from=registry.connect.redhat.com/sonatype/nexus-repository-manager:latest

oc new-app nexus-repository-manager --name=nexus
oc patch dc/nexus -p '{"spec":{"strategy":{"type":"Recreate"}}}'
oc expose svc/nexus --hostname=nexus.app.itix.fr
oc patch route/nexus -p '{"spec":{"tls":{"insecureEdgeTerminationPolicy":"Redirect","termination":"edge"}}}'

oc set probe dc/nexus --liveness --failure-threshold 3 --initial-delay-seconds 30 --open-tcp=8081
oc set probe dc/nexus --readiness --failure-threshold 3 --initial-delay-seconds 30 --get-url=http://:8081/service/rest/repository/browse/maven-public/

oc set volumes dc/nexus --add --name 'nexus-volume-1' --type 'pvc' --mount-path '/nexus-data/' --claim-name 'nexus' --claim-size '1Gi' --overwrite

curl -o /tmp/nexus-functions -s https://raw.githubusercontent.com/OpenShiftDemos/nexus/master/scripts/nexus-functions
source /tmp/nexus-functions
add_nexus3_redhat_repos admin admin123 https://nexus.app.itix.fr
```

### Ansible Tower

```sh
oc new-project tower --display-name="Ansible Tower"
oc create -f - <<EOF
kind: List
apiVersion: v1
items:
- apiVersion: v1
  kind: ImageStream
  metadata:
    name: ansible-tower-with-venv
- apiVersion: v1
  kind: ImageStream
  metadata:
    name: ansible-tower
  spec:
    dockerImageRepository: registry.access.redhat.com
    tags:
      - name: '3.4'
        from:
          kind: DockerImage
          name: 'registry.access.redhat.com/ansible-tower-34/ansible-tower:3.4.3'
        importPolicy:
          scheduled: true
- apiVersion: v1
  kind: BuildConfig
  metadata:
    name: ansible-tower-with-venv
  spec:
    output:
      to:
        kind: ImageStreamTag
        name: ansible-tower-with-venv:latest
    runPolicy: Serial
    source:
      dockerfile: |-
        FROM registry.access.redhat.com/ansible-tower-34/ansible-tower:latest
        USER root
        RUN yum install -y gcc
        RUN mkdir -p /var/lib/awx/venv/jinja2.8
        RUN virtualenv --system-site-packages /var/lib/awx/venv/jinja2.8
        RUN sh -c ". /var/lib/awx/venv/jinja2.8/bin/activate ; /var/lib/awx/venv/jinja2.8/bin/pip install python-memcached psutil ; pip install --upgrade jinja2;"
    strategy:
      type: Docker
      dockerStrategy:
        from:
          kind: ImageStreamTag
          name: ansible-tower:3.4
    triggers:
    - type: ImageChange
    - type: ConfigChang
EOF
```

```sh
curl -L -o tower-setup.tar.gz https://releases.ansible.com/ansible-tower/setup_openshift/ansible-tower-openshift-setup-3.4.3.tar.gz
tar zxvf tower-setup.tar.gz
cd ansible-tower-openshift-setup-3.4.3/

cat <<EOF > inventory
localhost ansible_connection=local ansible_python_interpreter="/usr/bin/env python"

[all:vars]
admin_user=admin
admin_password="$(head -c16 /dev/urandom |openssl dgst -sha1)"
pg_username=tower
pg_password="$(head -c16 /dev/urandom |openssl dgst -sha1)"
pg_database='tower'
pg_port=5432
secret_key="$(head -c16 /dev/urandom |openssl dgst -sha1)"
rabbitmq_password="$(head -c16 /dev/urandom |openssl dgst -sha1)"
rabbitmq_erlang_cookie="$(head -c16 /dev/urandom |openssl dgst -sha1)"
openshift_skip_tls_verify=true
openshift_password=dummy # Not used but required by the installer
EOF

oc apply -f - <<EOF
apiVersion: "v1"
kind: "PersistentVolumeClaim"
metadata:
  name: "postgresql"
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "5Gi"
EOF

./setup_openshift.sh -e openshift_host="$(oc whoami --show-server)" -e openshift_project=tower -e openshift_user="$(oc whoami)" -e openshift_token="$(oc whoami -t)" -e kubernetes_web_image="docker-registry.default.svc.cluster.local:5000/tower/ansible-tower-with-venv" -e kubernetes_task_image="docker-registry.default.svc.cluster.local:5000/tower/ansible-tower-with-venv" -e kubernetes_task_version=latest -e kubernetes_web_version=latest

oc delete route ansible-tower-web-svc
oc create -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ansible-tower-web-svc
  namespace: tower
spec:
  host: ansible.app.itix.fr
  port:
    targetPort: http
  tls:
    insecureEdgeTerminationPolicy: Redirect
    termination: edge
  to:
    kind: Service
    name: ansible-tower-web-svc
EOF
```

## Deploy Integr8ly

```sh
cd integr8ly
ansible-playbook -i ../prod.hosts playbooks/install.yml
```
