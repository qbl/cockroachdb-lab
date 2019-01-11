# CockroachDB Lab

Experimenting with CockroachDB to Create a Resilient Database Cluster

# Steps

## 1. Create a Kubernetes Cluster

1. For this experiment, I'm using GKE.

    ```
    gcloud container clusters create cockroachdb
    ```

2. Once done, grep account information from the cluster.

    ```
    gcloud info | grep Account
    ```

3. Create RBAC roles needed for CockroachDB to run on GKE.

    ```
    kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=<your.google.cloud.email@example.org>
    ```

