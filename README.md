# OpenShift ACM Cluster Bootstrap

aushacker<br/>
March 2026<br/>

## Overview

## Step 01 - Apply MirrorSets and CatalogSources (Disconnected Only)

For now this is a place holder. In a disconnected environment you need to apply the following:

- ImageDigestMirrorSet
- ImageTagMirrorSet
- CatalogSource

## Step 02 - (Optional) Install ExternalSecretsOperator

```
oc apply -k policies/operators/externals-secrets/operator/

# Manually check for operator installation complete

oc apply -k policies/operators/externals-secrets/instance/
```

## Step 03 - Install OpenShift Gitops Operator

```
oc apply -k policies/operators/openshift-gitops/operator/

# Manually check for operator installation complete

oc apply -k policies/operators/openshift-gitops/instance/
```
