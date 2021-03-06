---
title: "Run Ark on AWS"
layout: docs
---

To set up Ark on AWS, you:

* Create your S3 bucket
* Create an AWS IAM user for Ark
* Configure the server
* Create a Secret for your credentials

If you do not have the `aws` CLI locally installed, follow the [user guide][5] to set it up.

## Create S3 bucket

Heptio Ark requires an object storage bucket to store backups in, preferrably unique to a single Kubernetes cluster (see the [FAQ][20] for more details). Create an S3 bucket, replacing placeholders appropriately:

```bash
aws s3api create-bucket \
    --bucket <YOUR_BUCKET> \
    --region <YOUR_REGION> \
    --create-bucket-configuration LocationConstraint=<YOUR_REGION>
```
NOTE: us-east-1 does not support a `LocationConstraint`.  If your region is `us-east-1`, omit the bucket configuration:

```bash
aws s3api create-bucket \
    --bucket <YOUR_BUCKET> \
    --region us-east-1
```

## Create IAM user

For more information, see [the AWS documentation on IAM users][14].

1. Create the IAM user:

    ```bash
    aws iam create-user --user-name heptio-ark
    ```
    
    > If you'll be using Ark to backup multiple clusters with multiple S3 buckets, it may be desirable to create a unique username per cluster rather than the default `heptio-ark`.

2. Attach policies to give `heptio-ark` the necessary permissions:

    ```bash
    BUCKET=<YOUR_BUCKET>
    cat > heptio-ark-policy.json <<EOF
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ec2:DescribeVolumes",
                    "ec2:DescribeSnapshots",
                    "ec2:CreateTags",
                    "ec2:CreateVolume",
                    "ec2:CreateSnapshot",
                    "ec2:DeleteSnapshot"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject",
                    "s3:DeleteObject",
                    "s3:PutObject",
                    "s3:AbortMultipartUpload",
                    "s3:ListMultipartUploadParts"
                ],
                "Resource": [
                    "arn:aws:s3:::${BUCKET}/*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::${BUCKET}"
                ]
            }
        ]
    }
    EOF

    aws iam put-user-policy \
      --user-name heptio-ark \
      --policy-name heptio-ark \
      --policy-document file://heptio-ark-policy.json
    ```

3. Create an access key for the user:

    ```bash
    aws iam create-access-key --user-name heptio-ark
    ```

    The result should look like:

    ```json
     {
        "AccessKey": {
              "UserName": "heptio-ark",
              "Status": "Active",
              "CreateDate": "2017-07-31T22:24:41.576Z",
              "SecretAccessKey": <AWS_SECRET_ACCESS_KEY>,
              "AccessKeyId": <AWS_ACCESS_KEY_ID>
          }
     }
    ```

4. Create an Ark-specific credentials file (`credentials-ark`) in your local directory:

    ```
    [default]
    aws_access_key_id=<AWS_ACCESS_KEY_ID>
    aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>
    ```

    where the access key id and secret are the values returned from the `create-access-key` request.

## Credentials and configuration

In the Ark directory (i.e. where you extracted the release tarball), run the following to first set up namespaces, RBAC, and other scaffolding. To run in a custom namespace, make sure that you have edited the YAML files to specify the namespace. See [Run in custom namespace][0].

```bash
kubectl apply -f config/common/00-prereqs.yaml
```

Create a Secret. In the directory of the credentials file you just created, run:

```bash
kubectl create secret generic cloud-credentials \
    --namespace <ARK_NAMESPACE> \
    --from-file cloud=credentials-ark
```

Specify the following values in the example files:

* In `config/aws/05-ark-backupstoragelocation.yaml`:

  * Replace `<YOUR_BUCKET>` and `<YOUR_REGION>` (for S3 backup storage, region is optional and will be queried from the AWS S3 API if not provided). See the [BackupStorageLocation definition][21] for details.

* In `config/aws/06-ark-volumesnapshotlocation.yaml`:

  * Replace `<YOUR_REGION>`. See the [VolumeSnapshotLocation definition][6] for details.

* (Optional, use only to specify multiple volume snapshot locations) In `config/aws/10-deployment.yaml` (or `config/aws/10-deployment-kube2iam.yaml`, as appropriate):

  * Uncomment the `--default-volume-snapshot-locations` and replace provider locations with the values for your environment.

* (Optional) If you run the nginx example, in file `config/nginx-app/with-pv.yaml`:

    * Replace `<YOUR_STORAGE_CLASS_NAME>` with `gp2`. This is AWS's default `StorageClass` name.

* (Optional) If you have multiple clusters and you want to support migration of resources between them, in file `config/aws/10-deployment.yaml`:

    * Uncomment the environment variable `AWS_CLUSTER_NAME` and replace `<YOUR_CLUSTER_NAME>` with the current cluster's name. When restoring backup, it will make Ark (and cluster it's running on) claim ownership of AWS volumes created from snapshots taken on different cluster.
    The best way to get the current cluster's name is to either check it with used deployment tool or to read it directly from the EC2 instances tags. 
    
      The following listing shows how to get the cluster's nodes EC2 Tags. First, get the nodes external IDs (EC2 IDs):

        ```bash
        kubectl get nodes -o jsonpath='{.items[*].spec.externalID}'
        ```
    
      Copy one of the returned IDs `<ID>` and use it with the `aws` CLI tool to search for one of the following:

      * The `kubernetes.io/cluster/<AWS_CLUSTER_NAME>` tag of the value `owned`. The `<AWS_CLUSTER_NAME>` is then your cluster's name:

          ```bash
          aws ec2 describe-tags --filters "Name=resource-id,Values=<ID>" "Name=value,Values=owned"
          ```
    
      * If the first output returns nothing, then check for the legacy Tag `KubernetesCluster` of the value `<AWS_CLUSTER_NAME>`:

          ```bash
          aws ec2 describe-tags --filters "Name=resource-id,Values=<ID>" "Name=key,Values=KubernetesCluster"
        ```

## Start the server

In the root of your Ark directory, run:

  ```bash
  kubectl apply -f config/aws/05-ark-backupstoragelocation.yaml
  kubectl apply -f config/aws/06-ark-volumesnapshotlocation.yaml
  kubectl apply -f config/aws/10-deployment.yaml
  ```

## ALTERNATIVE: Setup permissions using kube2iam

[Kube2iam](https://github.com/jtblin/kube2iam) is a Kubernetes application that allows managing AWS IAM permissions for pod via annotations rather than operating on API keys.

> This path assumes you have `kube2iam` already running in your Kubernetes cluster. If that is not the case, please install it first, following the docs here: [https://github.com/jtblin/kube2iam](https://github.com/jtblin/kube2iam)

It can be set up for Ark by creating a role that will have required permissions, and later by adding the permissions annotation on the ark deployment to define which role it should use internally.

1. Create a Trust Policy document to allow the role being used for EC2 management & assume kube2iam role:

    ```bash
    cat > heptio-ark-trust-policy.json <<EOF
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "ec2.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            },
            {
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/<ROLE_CREATED_WHEN_INITIALIZING_KUBE2IAM>"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }
    EOF
    ```

2. Create the IAM role:

    ```bash
    aws iam create-role --role-name heptio-ark --assume-role-policy-document file://./heptio-ark-trust-policy.json
    ```

3. Attach policies to give `heptio-ark` the necessary permissions:

    ```bash
    BUCKET=<YOUR_BUCKET>
    cat > heptio-ark-policy.json <<EOF
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ec2:DescribeVolumes",
                    "ec2:DescribeSnapshots",
                    "ec2:CreateTags",
                    "ec2:CreateVolume",
                    "ec2:CreateSnapshot",
                    "ec2:DeleteSnapshot"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject",
                    "s3:DeleteObject",
                    "s3:PutObject",
                    "s3:AbortMultipartUpload",
                    "s3:ListMultipartUploadParts"
                ],
                "Resource": [
                    "arn:aws:s3:::${BUCKET}/*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::${BUCKET}"
                ]
            }
        ]
    }
    EOF

    aws iam put-role-policy \
      --role-name heptio-ark \
      --policy-name heptio-ark-policy \
      --policy-document file://./heptio-ark-policy.json
    ```
4. Update `AWS_ACCOUNT_ID` & `HEPTIO_ARK_ROLE_NAME` in the file `config/aws/10-deployment-kube2iam.yaml`:

    ```
    ---
    apiVersion: apps/v1beta1
    kind: Deployment
    metadata:
        namespace: heptio-ark
        name: ark
    spec:
        replicas: 1
        template:
            metadata:
                labels:
                    component: ark
                annotations:
                    iam.amazonaws.com/role: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<HEPTIO_ARK_ROLE_NAME>
    ...
    ```

5. Run Ark deployment using the file `config/aws/10-deployment-kube2iam.yaml`.

[0]: namespace.md
[5]: https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html
[6]: api-types/volumesnapshotlocation.md#aws
[14]: http://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html
[20]: faq.md
[21]: api-types/backupstoragelocation.md#aws
