# Velero plugins for AWS

## Overview

This repository contains these plugins to support running Velero on AWS:

- An object store plugin for persisting and retrieving backups on AWS S3. Content of backup is log files, warning/error files, restore logs.

- A volume snapshotter plugin for creating snapshots from volumes (during a backup) and volumes from snapshots (during a restore) on AWS EBS.


## Compatibility

Below is a listing of plugin versions and respective Velero versions that are compatible.

| Plugin Version  | Velero Version |
|-----------------|----------------|
| v1.2.x          | v1.6.x         |
| v1.1.x          | v1.5.x         |
| v1.1.x          | v1.4.x         |
| v1.0.x          | v1.3.x         |
| v1.0.x          | v1.2.0         |

## Filing issues

If you would like to file a GitHub issue for the plugin, please open the issue on the [core Velero repo][103]


## Setup

To set up Velero on AWS, you:

* [Create an S3 bucket][1]
* [Set permissions for Velero][2]
* [Install and start Velero][3]

You can also use this plugin to [migrate PVs across clusters][5] or create an additional [Backup Storage Location][12].

If you do not have the `aws` CLI locally installed, follow the [user guide][6] to set it up.

## Create S3 bucket

Velero requires an object storage bucket to store backups in, preferably unique to a single Kubernetes cluster (see the [FAQ][11] for more details). Create an S3 bucket, replacing placeholders appropriately:


aws s3api create-bucket \
    --bucket velero-bucket-nitin \
    --region us-west-2
```

## Set permissions for Velero

### Option 1: Set permissions with an IAM user

For more information, see [the AWS documentation on IAM users][10].

1. Create the IAM user:

    ```bash
    aws iam create-user --user-name velero
    ```

    If you'll be using Velero to backup multiple clusters with multiple S3 buckets, it may be desirable to create a unique username per cluster rather than the default `velero`.

2. Attach policies to give `velero` the necessary permissions:

    ```
    cat > velero-policy.json ------> check at the top
    
    aws iam put-user-policy \
      --user-name velero \
      --policy-name velero \
      --policy-document file://velero-policy.json
    ```

3. Create an access key for the user:

    ```bash
    aws iam create-access-key --user-name velero
    ```

    **The result should look like**:

    ```json
    {
      "AccessKey": {
            "UserName": "velero",
            "Status": "Active",
            "CreateDate": "2017-07-31T22:24:41.576Z",
            "SecretAccessKey": <AWS_SECRET_ACCESS_KEY>,
            "AccessKeyId": <AWS_ACCESS_KEY_ID>
      }
    }
    ```

4. Create a Velero-specific credentials file (`credentials-velero`) in your local directory:

    ```bash
    [default]
    aws_access_key_id=<AWS_ACCESS_KEY_ID>
    aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>
    ```

    where the access key id and secret are the values returned from the `create-access-key` request.


## Install and start Velero

[Download][4] Velero

Install Velero, including all prerequisites, into the cluster and start the deployment. This will create a namespace called `velero`, and place a deployment named `velero` in it.

**If using IAM user and access key**:

```bash
velero install \
     --provider aws \
     --plugins velero/velero-plugin-for-aws:v1.2.0 \
     --bucket velero-bucket-nitin \
     --backup-location-config region=us-west-2 \
     --snapshot-location-config region=us-west-2 \
     --use-restic \
     --secret-file ./credentials-velero
```

### Prerequisites

* Velero 1.6.0 or later
* AWS plugin must be installed, either at install time, or by running `velero plugin add velero/velero-plugin-for-aws:v1.2.0`

### Configure S3 bucket and credentials

To configure a new Backup Storage Location with its own credentials, it is necessary to follow the steps above to [create the bucket to use][15] and to [generate the credentials file][16] to interact with that bucket.
Once you have created the credentials file, create a [Kubernetes Secret][17] in the Velero namespace that contains these credentials:

```bash
kubectl create secret generic -n velero bsl-credentials --from-file=aws=credentials-velero
```

This will create a secret named `bsl-credentials` with a single key (`aws`) which contains the contents of your credentials file.
The name and key of this secret will be given to Velero when creating the Backup Storage Location, so it knows which secret data to use.

### Create Backup Storage Location

Once the bucket and credentials have been configured, these can be used to create the new Backup Storage Location:

```bash
velero backup-location create **backup** \
 --provider aws \
 --bucket velero-bucket-nitin \
 --config region=us-west-2 \
 --credential=bsl-credentials=aws
```

```bash
velero backup-location get
```
