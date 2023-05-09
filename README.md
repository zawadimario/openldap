# Openldap Deployment
This deployment consists of the following resources

* OpenLDAP deployment configuration
* Service Definition
* Service Account
* Secret that holds LDAP_ADMIN_PASSWORD. This is optionally included but can be added as environment variable in the deployment
* Kustomization file to automate the deployment
* Sample ldif, which is to be used as template only to add users' entries to the database

Run the below command to deploy OpenLDAP Server
```
kustomize build openldap | kubectl apply -f - 
```
### Client side configurations

OIDC Auth parameters

```
OIDC_PROVIDER=http://dex.auth.svc.cluster.local:5556/dex
OIDC_AUTH_URL=/dex/auth
OIDC_SCOPES=profile email groups openid
REDIRECT_URL=/login/oidc
SKIP_AUTH_URI=/dex
USERID_HEADER=kubeflow-userid
USERID_PREFIX=
USERID_CLAIM=email
PORT="8080"
STORE_PATH=/var/lib/authservice/data.db
```

Dex IdP client config

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: dex
data:
  config.yaml: |
    issuer: http://dex.auth.svc.cluster.local:5556/dex
    storage:
      type: kubernetes
      config:
        inCluster: true
    web:
      http: 0.0.0.0:5556
    logger:
      level: "debug"
      format: text
    connectors:
      - type: ldap
        id: ldap
        name: OpenLDAP
        config:
          host: openldap.auth:389
          insecureNoSSL: true
          insecureSkipVerify: true
          startTLS: false
          bindDN: cn=adminuser,dc=techtel,dc=com
          bindPW: admin123

          usernamePrompt: "Email Address"

          userSearch:
            baseDN: dc=techtel,dc=com
            username: sn
            idAttr: uid
            emailAttr: mail
            nameAttr: sn
          
          groupSearch:
            baseDN: dc=techtel,dc=com
            userMatchers:
            - userAttr: DN
              groupAttr: member
            nameAttr: sn

    oauth2:
      skipApprovalScreen: true
    staticClients:
    - idEnv: OIDC_CLIENT_ID
      redirectURIs: ["/login/oidc"]
      name: 'Dex Login Application'
      secretEnv: OIDC_CLIENT_SECRET
```
Note that bind password used here is only for testing purpose and should never be used in production. A stronger password should be picked. Also note that a shorter notation of LDAP host is used. If this doesn't work consider using full form (K8s FQDN) as <u> openldap.auth.svc.cluster.local:389 </u>

### Adding user entries

Exec into the OpenLDAP pod and create a ldif file and load onto the server

```
kubectl exec -it -n auth openldap-xyz-xyz -- bash
```
Find the user-entry.ldif file in the home directory, modify accordingly and load it onto the server.
```
ldapadd -x -D "cn=adminuser,dc=techtel,dc=com" -w admin123 -f user-entry.ldif
```
Verify that the user was created, as well as preexisting users, group, etc
```
ldapsearch -x -b 'dc=techtel,dc=com' 'objectclass=*'
```
Proceed to create a profile that is to be bound to the email of the user/s created and login to Kubeflow dashboard

### Logs
Use kubectl to check logs on Dex, OIDC Auth and OpenLDAP pods. It should be seen that the user/s were successfully added and can login to Kubeflow Dashboard.


### Links
https://github.com/bitnami/containers/tree/main/bitnami/openldap

### Acknowledgement
lernmat.mario@gmail.com