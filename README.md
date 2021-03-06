# CockroachDB Lab

Experimenting with CockroachDB to Create a Resilient Database Cluster

# Running CockroachDB on Local Machine

## 1. Install CockroachDB

1. On Mac OS, we can use `brew` to install CockroachDB.

    ```
    brew install cockroach
    ```

## 2. Setup Local Cluster

1. Start the first node.

    ```
    cockroach start --insecure --listen-addr=localhost
    ```

2. Add nodes to the cluster.

    We are going to add two nodes to our CockroachDB cluster.

    Node 2:
    ```
    cockroach start \
    --insecure \
    --store=node2 \
    --listen-addr=localhost:26258 \
    --http-addr=localhost:8081 \
    --join=localhost:26257
    ```

    Node 3
    ```
    cockroach start \
    --insecure \
    --store=node3 \
    --listen-addr=localhost:26259 \
    --http-addr=localhost:8082 \
    --join=localhost:26257
    ```
# Experiments

## 1. Data Replication

1. Write data to the first node.

    Generate an example intro database:
    ```
    cockroach workload init intro \
    'postgresql://root@localhost:26257?sslmode=disable'
    ```

    Open the SQL client:
    ```
    cockroach sql --insecure --host=localhost:26257
    ```

    Verify that intro database exists:
    ```
    SHOW DATABASES;
    SHOW TABLES FROM INTRO;
    SELECT * FROM intro.mytable WHERE (l % 2) = 0;
    \q
    ```

2. Open SQL client in the other nodes, and see if the data replicated to those nodes.

    Node 2:
    ```
    cockroach sql --insecure --host=localhost:26258
    SHOW DATABASES;
    SHOW TABLES FROM INTRO;
    SELECT * FROM intro.mytable WHERE (l % 2) = 0;
    \q
    ```

    Node 3:
    ```
    cockroach sql --insecure --host=localhost:26259
    SHOW DATABASES;
    SHOW TABLES FROM INTRO;
    SELECT * FROM intro.mytable WHERE (l % 2) = 0;
    \q
    ```

## 2. Fault Tolerance & Recovery

1. Remove a node temporarily.

    ```
    cockroach quit --insecure --host=localhost:26258
    ```

2. Verify that the cluster is not affected.

    ```
    cockroach sql --insecure --host=localhost:26259
    SHOW DATABASES;
    \q
    ```

3. Write new data while a node is offline.

    Generate new sample workload:
    ```
    cockroach workload init startrek \
    'postgresql://root@localhost:26257?sslmode=disable'
    ```

    Open SQL client and verify that new data is there:
    ```
    cockroach sql --insecure --host=localhost:26259
    SHOW DATABASES;
    SHOW TABLES FROM startrek;
    SELECT * FROM startrek.eipsodes WHERE stardate > 5500;
    ```

4. Rejoin the previously removed node.

    ```
    cockroach start \
    --insecure \
    --store=node2 \
    --listen-addr=localhost:26258 \
    --http-addr=localhost:8081 \
    --join=localhost:26257

    ```

5. Verify that the rejoined node catches up with the whole cluster.

    ```
    cockraoch sql --insecure --host=localhost:26258
    SHOW DATABASES;
    SHOW TABLES FROM startrek;
    SELECT * FROM startrek.episodes WHERE stardate > 5500;
    ```

## 3. Automatic Rebalancing

1. Run a sample workload.

    ```
    cockroach workload init tpcc \
    'postgresql://root@localhost:26257?sslmode=disable'
    ```

2. Add 2 more nodes.

    Node 4:
    ```
    cockroach start \
    --insecure \
    --store=scale-node4 \
    --listen-addr=localhost:26260 \
    --http-addr=localhost:8083 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ```

    Node 5:
    ```
    cockroach start \
    --insecure \
    --store=scale-node5 \
    --listen-addr=localhost:26261 \
    --http-addr=localhost:8084 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ```

3. Watch in dashboard as the new nodes catch up their number of replicas.

# Running CockroachDB on Kubernetes Cluster

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

## 2. Deploy CockroachDB

1. Install Helm client, if not installed already.

    For Mac OS, the command is as follow:
    ```
    brew install kubernetes-helm
    ```

2. Install Helm server and its required RBAC.

    Create a `rbac-config.yaml` to define a role and service account.

    ```
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: tiller
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: tiller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
      - kind: ServiceAccount
        name: tiller
        namespace: kube-system
    ```

    Create the service account.

    ```
    kubectl create -f rbac-config.yaml
    ```

   Init Helm server:

    ```
    helm init --service-account tiller
    ```

3. Install CockroachDB Helm chart.

    Install with the following command:

    ```
    helm install --name qbl-cockroach --set Secure.Enabled=true stable/cockroachdb
    ```

    After that, we see approve CSR for pods generated by CockroachDB
chart:

    ```
    kubectl get csr
    kubectl certificate approve default.node.qbl-cockroach-cockroachdb-0
    kubectl certificate approve default.node.qbl-cockroach-cockroachdb-1
    kubectl certificate approve default.node.qbl-cockroach-cockroachdb-2
    kubectl certificate approve default.client.root
    ```

## 3. Use Built-In SQL Client

1. Create a client pod

    Create `client-secure.yaml` pod definition as follow:

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: cockroachdb-client-secure
      labels:
        app: cockroachdb-client
    spec:
      serviceAccountName: qbl-cockroach-cockroachdb
      initContainers:
      - name: init-certs
        image: cockroachdb/cockroach-k8s-request-cert:0.4
        imagePullPolicy: IfNotPresent
        command:
        - "/bin/ash"
        - "-ecx"
        - "/request-cert -namespace=${POD_NAMESPACE} -certs-dir=/cockroach-certs -type=client -user=root -symlink-ca-from=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: client-certs
          mountPath: /cockroach-certs
      containers:
      - name: cockroachdb-client
        image: cockroachdb/cockroach:v2.1.3
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: client-certs
          mountPath: /cockroach-certs
        command:
        - sleep
        - "2147483648" # 2^31
      terminationGracePeriodSeconds: 0
      volumes:
      - name: client-certs
        emptyDir: {}
    ```

    Create the pod:

    ```
    kubectl create -f client-secure.yaml
    ```

2. SSH into the client pod.

    ```
    kubectl exec -it cockroachdb-client-secure -- ./cockroach sql --certs-dir=/cockroach-certs --host=qbl-cockroach-cockroachdb-public
    ```

3. Try out some SQL commands.

    ```
    CREATE DATABASE bank;
    CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
    INSERT INTO bank.accounts VALUES (1, 1000.50);
    SELECT * FROM bank.accounts;
    ```

4. Create a user with a password.

    Replace <username> and <password> in the following command with
username and password that you want.

    ```
    CREATE USER <username> WITH PASSWORD '<password>';
    ```

5. Exit the SQL shell and pod with `\q` command.

## 4. Accessing The Admin UI

1. Port-forward from local machine.

    ```
    kubectl port-forward qbl-cockroach-cockroachdb-0 8080
    ```

2. Access from `localhost:8080`.
