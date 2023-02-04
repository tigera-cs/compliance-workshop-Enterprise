# 1. Access

## Set Up Users

Calico Enterprise supports Google OIDC, Openshift and LDAP Authentication as well as local users and connecting to your own IdP (based on OIDC).

## Local User Setup

In Kubernetes, cluster roles specify cluster permissions and are bound to users using cluster role bindings. In Calico Enterprise we provide the following predefined roles.

**tigera-network-admin**

- Superuser access for Kibana (including Elastic user and license management), and all Calico resources in *projectcalico.org* and *networking.k8s.io* API groups (get, list, watch, create, update, patch, delete)

**tigera-ui-user**

- Basic user with access to Calico Enterprise Manager UI and Kibana:
    - List/view Calico Enterprise policy and tier resources in the *projectcalico.org* and *networking.k8s.io* API groups
    - List/view logs in Kibana

**mcm-user**

- Admin user with full permissions to management and managed clusters for multi-cluster management

## OIDC User Setup

To Configure an external identity provider for user authentication refer to this [documentation](https://docs.tigera.io/v3.15/getting-started/cnx/configure-identity-provider). 


## Reference Documentation

For more details on Authentication, please see the [Authentication quickstart.](https://docs.tigera.io/v3.15/getting-started/cnx/authentication-quickstart)