# Velero with EKS

### Prerequisites
```bash
* Velero 1.6.0 or later
* AWS plugin must be installed, either at install time, or by running `velero plugin add velero/velero-plugin-for-aws:v1.2.0
```
## What all the things will be covered in DR kubernetes
```bash
1). Deploying two sample applications & PV/PVC, Pods/Deploy, Replicas, cofigmapps, secrets
2). Configuring and deploying Velero.
3). Creating S3 bucket for storing cluster backup.
4). Creating IAM user (Velero) and attaching right policies with it.
5). Taking application backup.
6). Destroying current cluster.
7). Restore all the applications & PV/PVC, Pods/Deploy, Replicas, cofigmapps, secrets to the new kubernetes cluster.
```

## Compatibility
```bash
Below is a listing of plugin versions and respective Velero versions that are compatible.

| Plugin Version  | Velero Version |
|-----------------|----------------|
| v1.2.x          | v1.6.x         |
| v1.1.x          | v1.5.x         |
| v1.1.x          | v1.4.x         |
| v1.0.x          | v1.3.x         |
| v1.0.x          | v1.2.0         |
```

## Setup

```bash
To set up Velero on AWS, you:

Create an S3 bucket
Set permissions for Velero
Install and start Velero
```

## Create S3 bucket

```bash
aws s3api create-bucket \
    --bucket velero-bucket-nitin \
    --region us-west-2
```

## Set permissions for Velero

### Option 1: Set permissions with an IAM user

1. Create the IAM user:

    ```bash
    aws iam create-user --user-name velero
    ```


2. Attach policies to give `velero` the necessary permissions:

    ```
    cat > velero-policy.json <<EOF
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
                    "arn:aws:s3:::velero-bucket-nitin/*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::velero-bucket-nitin"
                ]
            }
        ]
    }
    EOF
    ```
    ```bash
    aws iam put-user-policy \
      --user-name velero \
      --policy-name velero \
      --policy-document file://velero-policy.json
    ```

3. Create an access key for the user:

    ```bash
    aws iam create-access-key --user-name velero
    ```

    The result should look like:

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


## Install and start Velero

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

### Configure S3 bucket and credentials

```bash
kubectl create secret generic -n velero bsl-credentials --from-file=aws=credentials-velero
```

### Create Backup Storage Location

Once the bucket and credentials have been configured, these can be used to create the new Backup Storage Location:

```bash
velero backup-location create backup \
     --provider aws \
     --bucket velero-bucket-nitin \
     --config region=us-west-2 \
     --credential=bsl-credentials=aws
```

### Driver snapshots:
```bash
kubectl -n wordpress annotate pod/wordpress-cd444bcd7-z28wr backup.velero.io/backup-volumes=wordpress-persistent-storage
kubectl -n wordpress annotate pod/wordpress-mysql-55ffc4bb89-nx2rc backup.velero.io/backup-volumes=mysql-persistent-storage
```

### velero commands:
```bash
velero backup-location get
velero backup get
velero backup create wordpress-backup --include-namespaces wordpress --wait
velero restore create --from-backup wordpress-backup
velero schedule create <SCHEDULE_NAME> --schedule="@daily"
```
