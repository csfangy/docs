To enable 90+ days of data retention in Kubecost, we recommend deploying with durable storage enabled. We provide two options for doing this: 1) in your cluster and 2) out of the cluster. This functionality also powers the Enterprise multi-cluster view, where data across clusters can be viewed in aggregate, as well as simple backup & restore capabilities.

**Note:** This feature today requires an Enterprise license.

## Option A: In cluster storage (Postgres)  

To enable Postgres-based long-term storage, complete the following:

1. **Helm chart configuration** -- in [values.yaml](https://github.com/kubecost/cost-analyzer-helm-chart/blob/master/cost-analyzer/values.yaml) set the `remoteWrite.postgres.enabled` attribute to true. The default backing disk is `200gb` but this can also be directly configured in values.yaml.  

2. **Verify successful install** -- Deploy or upgrade via install instructions at <http://kubecost.com/install>, passing this updated values.yaml file, and verify pods with the prefix `kubecost-cost-analyzer-adapter`
and `kubecost-cost-analyzer-postgres` are Running.  

3. **Confirm data is available**  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Vist this endpoint `http://<kubecost-address>/model/costDataModelRangeLarge`

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Here's an example use: `http://localhost:9090/model/costDataModelRangeLarge`

## Option B: Out of cluster storage (Thanos)  

Thanos-based durable storage provides long-term metric retention directly in a user-controlled bucket (e.g. S3 or GCS bucket) and can be enabled with the following steps:

Step 1: **Create object-store yaml file**  

This step creates a yaml file that contains your durable storage target (e.g. GCS, S3, etc.) configuration and access credentials. The details of this file are documented thoroughly in Thanos documentation: https://thanos.io/tip/thanos/storage.md/

__Google Cloud Storage__

Start by [creating a new Google Cloud Storage bucket](https://cloud.google.com/storage/docs/creating-buckets). The following example uses a bucket named `thanos-bucket`. Next, download a service account JSON file from Google's service account manager ([steps](/google-service-account-thanos.md)).

Now create a yaml file named `object-store.yaml` in the following format, using your bucket name and service account details:

```yaml
type: GCS
config:
  bucket: "thanos-bucket"
  service_account: |-
    {
      "type": "service_account",
      "project_id": "...",
      "private_key_id": "...",
      "private_key": "...",
      "client_email": "...",
      "client_id": "...",
      "auth_uri": "https://accounts.google.com/o/oauth2/auth",
      "token_uri": "https://oauth2.googleapis.com/token",
      "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
      "client_x509_cert_url": ""
    }
```
**Note:** given that this is yaml, it requires this specific indention.

**Warning:** do not apply a retention policy to your Thanos bucket, as it will prevent Thanos compaction from completing.

__AWS/S3__

Start by creating a new S3 bucket with all public access blocked. No other bucket configuration changes should be required. The following example uses a bucket named `kc-thanos-store`.

Next, add an IAM policy to access this bucket ([steps](/aws-service-account-thanos.md)).

Now create a yaml file named `object-store.yaml` with contents similar to the following example. See region to endpoint mappings here: <https://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region>

```
type: S3
config:
  bucket: "kc-thanos-store"
  endpoint: "s3.amazonaws.com"
  region: "us-east-1"
  access_key: "AKIAXW6UVLRRTDSCCU4D"
  insecure: false
  signature_version2: false
  secret_key: "<your-secret-key>"
  put_user_metadata:
      "X-Amz-Acl": "bucket-owner-full-control"
  http_config:
    idle_conn_timeout: 90s
    response_header_timeout: 2m
    insecure_skip_verify: false
  trace:
    enable: true
  part_size: 134217728
```
**Note:** given that this is yaml, it requires this specific indention.

Instead of using a service key, you can alternatively attach the policy to the Thanos pods service accounts. Your `object-store.yaml` should follow the format below when using this option, which does not contain the secret_key and access_key field.

```
type: S3
config:
  bucket: "kc-thanos-store"
  endpoint: "s3.amazonaws.com"
  region: "us-east-1"
  insecure: false
  signature_version2: false
  put_user_metadata:
      "X-Amz-Acl": "bucket-owner-full-control"
  http_config:
    idle_conn_timeout: 90s
    response_header_timeout: 2m
    insecure_skip_verify: false
  trace:
    enable: true
  part_size: 134217728
```

Then, follow the guide at [https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) to enable attaching IAM roles to pods.

You can define the IAM role to associate with a service account in your cluster by creating a service account in the same namespace as kubecost and adding an annotation to it of the form `eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<IAM_ROLE_NAME>`
 as described here: [https://docs.aws.amazon.com/eks/latest/userguide/specify-service-account-role.html](https://docs.aws.amazon.com/eks/latest/userguide/specify-service-account-role.html)

Once that annotation has been created and set, you'll need to attach it to the prometheus pod, the thanos compact pod, and the thanos store pod. 
For prometheus, set .Values.prometheus.serviceAccounts.server.create to false, and .Values.prometheus.serviceAccounts.server.name to the name of your created service account 
For thanos set `.Values.thanos.compact.serviceAccount`, and `.Values.thanos.store.serviceAccount` to the name of your created service account.

__Azure__

To use Azure Storage as Thanos object store, you need to precreate a storage account from Azure portal or using Azure CLI. Follow the instructions from Azure Storage Documentation: https://docs.microsoft.com/en-us/azure/storage/common/storage-quickstart-create-account

Now create a yaml file named `object-store.yaml` with the following format:

```
type: AZURE
config:
  storage_account: ""
  storage_account_key: ""
  container: ""
  endpoint: ""
  max_retries: 0
```

Step 2: **Create object-store secret**  

The final step prior to installation is to create a secret with the yaml file generated in the previous step:
```
$ kubectl create secret generic kubecost-thanos -n kubecost --from-file=./object-store.yaml
```

Step 3: **Deploying Kubecost with Thanos**  

The Thanos subchart includes `thanos-bucket`, `thanos-query`, `thanos-store`,  `thanos-compact`, and service discovery for `thanos-sidecar`. These components are recommended when deploying Thanos on multiple clusters.

These values can be adjusted under the `thanos` block in `values-thanos.yaml` - Available options can be observed here: [thanos/values.yaml](https://github.com/kubecost/cost-analyzer-helm-chart/blob/master/cost-analyzer/charts/thanos/values.yaml)

It's *important* to note that when running `helm install`, you must provide the base `values.yaml` followed by the override [values-thanos.yaml](https://github.com/kubecost/cost-analyzer-helm-chart/blob/master/cost-analyzer/values-thanos.yaml). For example:

```
$ helm install kubecost/cost-analyzer \
    --name kubecost \
    --namespace kubecost \
    -f values.yaml \
    -f values-thanos.yaml
```

Your deployment should now have Thanos enabled!

> Note: the `thanos-store` pod is by default configured to request 2 Gb in memory.

<a name="verify-thanos"></a>
**Verify Installation**  
In order to verify a correct installation, start by ensuring all pods are running without issue. If the pods mentioned above are not running successfully, then view pod logs for more detail. A common error is as follows, which means you do not have the correct access to the supplied bucket:

```
thanos-svc-account@project-227514.iam.gserviceaccount.com does not have storage.objects.list access to thanos-bucket., forbidden"
```

Assuming pods are running, use port forwarding to connect to the `thanos-query-http` endpoint:
```
$ kubectl port-forward svc/kubecost-thanos-query-http 8080:10902 --namespace kubecost
```
Then navigate to http://localhost:8080 in your browser. This page should look very similar to the Prometheus console.

![image](https://user-images.githubusercontent.com/334480/66616984-1076e480-eba1-11e9-8dd2-7c20541ad0b1.png)

If you navigate to the *Stores* using the top navigation bar, you should be able to see the status of both the `thanos-store` and `thanos-sidecar` which accompanied prometheus server:

![image](https://user-images.githubusercontent.com/334480/66617048-58960700-eba1-11e9-9f68-d007fcb11410.png)

Also note that the sidecar should identify with the unique `cluster_id` provided in your values.yaml in the previous step. Default value is `cluster-one`.


The default retention period for when data is moved into the object storage is currently *2h* - This configuration is based on Thanos suggested values. __By default, it will be 2 hours before data is written to the provided bucket.__

Instead of waiting *2h* to ensure that thanos was configured correctly, the default log level for the thanos workloads is `debug` (it's very light logging even on debug). You can get logs for the `thanos-sidecar`, which is part of the `prometheus-server` pod, and `thanos-store`. The logs should give you a clear indication of whether or not there was a problem consuming the secret and what the issue is. For more on Thanos architecture, view [this resource](https://github.com/thanos-io/thanos/blob/master/docs/design.md).

### Troubleshooting

#### Cluster not writing data to thanos bucket
If a cluster is not successfully writing data to the bucket, we recommend reviewing `thanos-sidecar` logs with the following command:

```
kubectl logs kubecost-prometheus-server-<your-pod-id> -n kubecost -c thanos-sidecar
```

Logs in the following format are evidence of a successful bucket write:

```
level=debug ts=2019-12-20T20:38:32.288251067Z caller=objstore.go:91 msg="uploaded file" from=/data/thanos/upload/01KL5YG9CQYZ81G9BZMTM3GJFH/meta.json dst=debug/metas/01KL5YG9CQYZ81G9BZMTM3GJFH.json bucket=kc-thanos

```

#### Stores not listed at the /stores endpoint
If thanos-query can't connect to both the sidecar and the store, you may want to directly specify the store GRPC service address instead of using DNS discovery (the default).
You can quickly test if this is the issue by running          
 `kubectl edit deployment kubecost-thanos-query -n kubecost`
 and adding 
 `- --store=kubecost-thanos-store-grpc.kubecost:10901` to the container args. This will cause a query restart and you can visit /stores again to see if the store has been added.

If it has, you'll want to use these addresses instead of DNS more permanently by setting .Values.thanos.query.stores in values-thanos.yaml
<pre>
```
...
thanos:
  store:
    enabled: true
    grpcSeriesMaxConcurrency: 20
    blockSyncConcurrency: 20
    extraEnv:
      - name: GOGC
        value: "100"
    resources: 
      requests:
        memory: "2.5Gi"
  query: 
    enabled: true
    timeout: 3m
    # Maximum number of queries processed concurrently by query node.
    maxConcurrent: 8
    # Maximum number of select requests made concurrently per a query.
    maxConcurrentSelect: 2
    resources:
      requests:
        memory: "2.5Gi"
    autoDownsampling: false
    extraEnv:
      - name: GOGC
        value: "100"
    <b>stores:</b>
      <b>- "kubecost-thanos-store-grpc.kubecost:10901"</b>
```
</pre>
