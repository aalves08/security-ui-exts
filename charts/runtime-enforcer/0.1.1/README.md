# SUSE Security Runtime Enforcer Extension for Rancher Manager

An extension for Rancher Manager which allows you to interact with SUSE Security Runtime Enforcer.

After installation, go to a cluster and you will see a new side navigation entry 'Runtime Enforcer'. This will allow you to install SUSE Security Runtime Enforcer into the cluster and manage `runtimeenforcer.kubewarden.io` resources and configuration.

## Preconditions for installing backend applications

1. Ensure the following required applications are installed. Remove any corresponding CRDs that do not have their backend application running.
    
    > Cert-manager
    * certificaterequests.cert-manager.io
    * certificates.cert-manager.io
    * challenges.acme.cert-manager.io
    * clusterissuers.cert-manager.io
    * issuers.cert-manager.io
    * orders.acme.cert-manager.io

    > Cert-manager-csi-driver
    No CRDs

    > SUSE Security Runtime Enforcer
    * workloadpolicies.runtimeenforcer.kubewarden.io
    * workloadpolicyproposals.runtimeenforcer.kubewarden.io

2. When installing `cert-manager`, check `Customize Helm options before install` on the installation page. Then in the next step, set `crds.enabled` to `true` in the YAML.


    
