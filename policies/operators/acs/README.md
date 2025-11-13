## Overview of RH ACS Operator Policy

The RH ACS deployment is more complicated compared to other operators.  This is mainly caused by the need to deploy and configure the hub, create the certificates to allow managed clusters to communicate with the ACS Central component, and deploy the Sensor/Collector on the Managed clusters.

If you are making use of the same organization and policy deployment model showcased in this repo the Placements are controlled by two labels on the ManagedClusters.  `acs=hub` and `acs=managed` will control which components are deployed.  You could easily swap this out for your own policies and deployment model.

**Validated Against**
- ACS 4.2 - 4.8

### Breakdown of 
Let's take a look at the various pieces of the Policy as defined in the `generator.yaml`.

There are two PolicySets defined; one for the Hub (`acs-operator-hub`) and one for the managed clusters (`acs-operator-manged`).

***acs-operator*** policy:
> This will create the namespaces required for the operator to run along with deploying the `OperatorGroup` and `Subscription`.  There is a check to ensure the operator deployment is healthy.
```
  - name: acs-operator
    remediationAction: enforce
    policySets:
      - acs-operator-hub
      - acs-operator-managed
    manifests:
      - path: ns-rhacs-operator.yaml
      - path: ns-stackrox.yaml
      - path: operatorgroup.yaml
        extraDependencies: 
          - name: acs-operator
            kind: ConfigurationPolicy
            compliance: "Compliant"
      - path: subscription.yaml
        extraDependencies: 
          - name: acs-operator
            kind: ConfigurationPolicy
            compliance: "Compliant"
      - path: health/operator/operator-status.yaml
        remediationAction: InformOnly
        extraDependencies: 
          - name: acs-operator3
            kind: ConfigurationPolicy
            compliance: "Compliant"
          - name: acs-operator4
            kind: ConfigurationPolicy
            compliance: "Compliant"
```

***acs-central*** policy:
> *Dependent on the acs-operator policy*

> This will deploy the Central component on the hub along with a `ConsoleLink` to make accessing RH ACS easier.  There is a status check to ensure the Central deployments are created and healthy.
```
  - name: acs-central
    remediationAction: enforce
    policySets:
      - acs-operator-hub
    dependencies:
      - name: acs-operator
        compliance: "Compliant"
    manifests:
      - path: central/central.yaml
      - path: central/consolelink.yaml
      - path: health/central/central-status.yaml
        remediationAction: InformOnly
        extraDependencies: 
          - name: acs-central
            kind: ConfigurationPolicy
            compliance: "Compliant"
```

***acs-central-init-bundle*** policy:
> *Dependent on the acs-central policy*

> Deploys a job which will create the ACS init-bundle in central.  The init-bundle is named after the date it is created.
```
  - name: acs-central-init-bundle
    remediationAction: enforce
    policySets:
      - acs-operator-hub
    dependencies:
      - name: acs-central
        compliance: "Compliant"
    manifests:
      - path: central/init-bundle/serviceaccount.yaml
      - path: central/init-bundle/role.yaml
      - path: central/init-bundle/rolebinding.yaml
      - path: central/init-bundle/job.yaml
        extraDependencies: 
          - name: acs-central-init-bundle
            kind: ConfigurationPolicy
            compliance: "Compliant"
          - name: acs-central-init-bundle2
            kind: ConfigurationPolicy
            compliance: "Compliant"
          - name: acs-central-init-bundle3
            kind: ConfigurationPolicy
            compliance: "Compliant"
```

***acs-central-init-bundle-cert*** policy:
> *Dependent on acs-central-init-bundle policy*

> Deploys a `CertificatePolicy` to evaluate if the init-bundle is expired, a check if the `sensor-tls` secret exists along with the `SecureCluster` to connect the Hub as a manged cluster in the Central component.
```
  - name: acs-central-init-bundle-cert
    remediationAction: enforce
    policySets:
      - acs-operator-hub
    dependencies:
      - name: acs-central-init-bundle
        compliance: "Compliant"
    manifests:
      - path: health/central/init-bundle/sensor-tls-cert.yaml
      - path: central/init-bundle/expired-sensor-tls.yaml
        remediationAction: InformOnly
      - path: sensor/securedcluster.yaml
        extraDependencies: 
          - name: acs-central-init-bundle-cert2
            kind: ConfigurationPolicy
            compliance: "Compliant"
```

***acs-central-expired-cert*** policy:
> *Dependent on acs-central-init-bundle-cert policy being **NonCompliant***

> When the cert is within 30 days of expiration, or the `sensor-tls` secret does not exist, policy will remove existing -tls secrets and the job to create the init-bundle.  This will cause the job to be recreated, creating a new init-bundle and ultimately updating the -tls secrets on the managed clusters with an updated certificate.  Because this occurs before cert expiration there should be no downtime or loss of connectivity.
```
  - name: acs-central-expired-certs
    remediationAction: enforce
    complianceType: mustnothave
    policySets:
      - acs-operator-hub
    dependencies:
      - name: acs-central-init-bundle-cert
        compliance: "NonCompliant"
    manifests:
      - path: central/init-bundle/expired-admission-control-tls.yaml
      - path: central/init-bundle/expired-collector-tls.yaml
      - path: central/init-bundle/expired-sensor-tls.yaml
      - path: central/init-bundle/job.yaml
        extraDependencies: 
          - name: acs-central-expired-certs
            kind: ConfigurationPolicy
            compliance: "Compliant"
          - name: acs-central-expired-certs2
            kind: ConfigurationPolicy
            compliance: "Compliant"
          - name: acs-central-expired-certs3
            kind: ConfigurationPolicy
            compliance: "Compliant"
```

***acs-sensor-sync-certs*** policy:
> Dependent on the acs-central-init-bundle-cert policy

> Copies the -tls secrets from the `stackrox` namespace where the init-bundle job creates them to the root policy namespace so they can be accessed from a hub template enabling them to be propagated to the manged clusters.
```
  - name: acs-sensor-sync-certs
    remediationAction: enforce
    policySets:
      - acs-operator-hub
    dependencies:
      - name: acs-central-init-bundle-cert
        compliance: "Compliant"
    manifests:
      - path: sensor/sensor-sync-tls-certs.yaml
```

***acs-sensor*** policy:
> Propagates the -tls certificates to the managed cluster along with deploying the `SecureCluster` to integrate the managed cluster with the hub Central.
```
  - name: acs-sensor
    remediationAction: enforce
    policySets:
      - acs-operator-managed
    manifests:
      - path: sensor/propagate-admission-control-tls.yaml
      - path: sensor/propagate-collector-tls.yaml
      - path: sensor/propagate-sensor-tls.yaml
      - path: sensor/securedcluster.yaml
        extraDependencies: 
          - name: acs-sensor
            kind: ConfigurationPolicy
            compliance: "Compliant"
          - name: acs-sensor2
            kind: ConfigurationPolicy
            compliance: "Compliant"
          - name: acs-sensor3
            kind: ConfigurationPolicy
            compliance: "Compliant"
```