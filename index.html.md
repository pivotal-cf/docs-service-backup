---
title: Service Backups for Pivotal Cloud Foundry
author: London Enablement Team

---

BOSH operators running services (e.g. Redis service broker for Cloud Foundry) may want to back up certain files from the virtual machines running these services so that they can restore them after a disaster.


## <a id="configuration"></a>Configuration

The Service Backup BOSH release backs up a directory on the instance VM it is located on to one of several supported destination types. The supported destination types are AWS S3, Azure blobstore, and SCP.

### <a id="uploading"></a>Uploading a service backup release

Service Backup is distributed as a BOSH final release. To upload a release to your BOSH director, upload the latest final release tarball from [Pivnet](https://network.pivotal.io/products/service-backups-sdk).

### <a id="configuring"></a>Configuring the manifest

Service Backup is designed to be co-located on service instance VMs, and must be included in that service's BOSH deployment manifest.

#### <a id="adding-service"></a>Adding service backups to your service deployment

Shown below is an template manifest, adding an S3 backup destination to a Redis service deployment. For further information on changing the backup destination, see [link](#backup-destinations).

```yml
---
properties:
  service-backup:
    destinations:
    - type: s3
      name: <OPTIONAL: destination name>
      config:
        bucket_name: <bucket>
        bucket_path: <path in bucket>
        access_key_id: <aws access key>
        secret_access_key: <aws secret key>
        endpoint_url: <OPTIONAL: S3 compatible endpoint URL>
        region: <OPTIONAL: S3 region, required for Signature Version 4 regions>
    source_folder: <directory to back up>
    cron_schedule: <cron schedule>
    backup_user: <OPTIONAL: backup user>
    source_executable: <OPTIONAL: run before each backup>
    exit_if_in_progress: <OPTIONAL: exits if another backup is already running. defaults to false>
    missing_properties_message: <OPTIONAL: message to log when properties are missing in the manifest. defaults to 'Provide these missing fields in your manifest.'>
    service_identifier_executable: <OPTIONAL: command that prints service instance ID on stdout>
    cleanup_executable: <OPTIONAL: command to run after each backup>
    alerts: # optional
      product_name: <product name>
      config:
        cloud_controller:
          url: <Cloud Foundry API URL>
          user: <Cloud Foundry username with SpaceAuditor role in cf_space>
          password: <Cloud Foundry password>
        notifications:
          service_url: <Cloud Foundry notification service URL>
          cf_org: <Cloud Foundry org name>
          cf_space: <Cloud Foundry space name>
          reply_to: <OPTIONAL: email reply-to address. This is required for some SMTP servers>
          client_id: <UAA client ID with authorities to send notifications>
          client_secret: <UAA client secret>
        timeout_seconds: <OPTIONAL: default is 60>
        skip_ssl_validation: <OPTIONAL: ignore TLS certification verification errors>

releases:
- name: redis
  version: latest
- name: service-backup
  version: latest

instance_groups:
- name: redis-server
  jobs:
  - name: redis-server
    release: redis
  - name: service-backup
    release: service-backup
```

#### <a id="configuring-backup-schedule"></a>Configuring the backup schedule

Backups will be triggered according to the schedule given by the `cron_schedule` property. See [robfig/cron](https://godoc.org/github.com/robfig/cron) for cron expression syntax.

#### <a id="configuring-backup-user"></a>Configuring the backup user

By default the service backup process will run as 'vcap'. This can be configured by setting the `backup_user` property.

If you are also providing your own `source_executable` be sure that the `backup_user` you create
has execute permissions for it.

#### <a id="defining-files"></a>Defining the files to be be backed up

The `source_folder` property names a local path from which backups are uploaded. All files in here are uploaded.

#### <a id="preparing-files"></a>Preparing the files to be backed up

The `source_executable` property names an executable to run before each backup. This is useful for services that require some operation to be performed before backing files up, for example triggering a Redis memory dump to disk. If a suitable executable is not included in the service release, you can add one by publishing it in a separate release, as its own package and job, and colocating it into the deployment. Tokens are split on spaces; first is command to execute and remaining are passed as args to command. This property is optional. If the field is not specified, it will simply be ignored and nothing will be executed.

#### <a id="cleaning-files"></a>Cleaning up the files which were backed up

The optional `cleanup_executable` property names a local executable to cleanup backups. Tokens are split on spaces; first is command to execute and remaining are passed as args to command.

#### <a id="alerts"></a>Sending an alert when a backup fails

When the optional `alerts` properties are configured an alert will be sent to SpaceDevelopers in the configured `cf_space` when a backup fails. Alerts are sent using the [Cloud Foundry Notifications Service](https://github.com/cloudfoundry-incubator/notifications).

#### <a id="correlating"></a>Correlating BOSH instances to Cloud Foundry service instances

BOSH operators might want to correlate BOSH-deployed VM instances with CF service instances, in which case the Service Author must provide a binary that returns a string identifier for your service instance. This will appear in all log messages under the data element `identifier`. For e.g.

`{ "source": "ServiceBackup", "message": "doing-stuff", "data": { "backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61", "identifier": "service_identifier" }, "timestamp": 1232345, "log_level": 1 }`

Add the optional `service_identifier_executable` key to your manifest (tokens are split on spaces; first is command to execute and remaining are passed as args to command).

#### <a id="naming-backup-destinations"></a>Naming backup destinations

Each destination can be given an optional `name` property. This will appear in the log messages for uploads to that destination. For example:

```json
{ "timestamp":"1467629245.010814428", "source":"ServiceBackup", "message":"ServiceBackup.WithIdentifier.about to upload /path/to/file to S3 remote path bucket_name/2016/07/04", "log_level":1, "data": { "backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61", "destination_name": "some-destination-name", "identifier": "service_identifier", "session":"1" } }
```

#### <a id="identifying-logs-for-a-backup-run"></a>Identifying logs for a backup run

The log lines of a particular backup run can be identified by correlating their unique `backup_guid` For example

```json
{"timestamp":"1467649696.229321241","source":"ServiceBackup","message":"ServiceBackup.WithIdentifier.Upload backup started","log_level":1,"data":{"backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61","identifier":"service_identifier","session":"1"}}
{"timestamp":"1467649696.229343414","source":"ServiceBackup","message":"ServiceBackup.WithIdentifier.Upload backup completed successfully","log_level":1,"data":{"backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61","duration_in_seconds":8.081000000000001e-06,"identifier":"service_identifier","session":"1","size_in_bytes":200}}
{"timestamp":"1467649696.229365349","source":"ServiceBackup","message":"ServiceBackup.WithIdentifier.Cleanup started","log_level":1,"data":{"backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61","identifier":"service_identifier","session":"1"}}
{"timestamp":"1467649696.232805967","source":"ServiceBackup","message":"ServiceBackup.WithIdentifier.Cleanup debug info","log_level":0,"data":{"backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61","cmd":"creator-cmd","identifier":"service_identifier","out":"Cleanup Complete\n","session":"1"}}
```

#### <a id="manual-backup"></a>Triggering manual service backups

BOSH operators might want to trigger a one-off, manual backup. To do this:

1. ssh onto a BOSH-deployed VM that has a service-backup job running on it.
1. Execute `/var/vcap/jobs/service-backup/bin/manual-backup`.

#### <a id="disabling"></a>Disabling Service Backups

Backups can be disabled by removing the `service-backup` section from your manifest and then redeploying. You can still leave the job on your instance group if you wish.

### <a id="backup-destinations"></a>Backup destinations

Service Backup supports S3 (AWS, Ceph s3, Swift w/ S3 compatibility module), Azure blobstore, and SCP. To change the backup destination change the manifest `destinations` value:

#### <a id="s3"></a>S3

```yml
properties:
  service-backup:
    destinations:
    - type: s3
      name: <OPTIONAL: destination name>
      config:
        bucket_name: <bucket>
        bucket_path: <path in bucket>
        access_key_id: <aws access key>
        secret_access_key: <aws secret key>
        endpoint_url: <OPTIONAL: url for S3-compatible blobstore>
        region: <OPTIONAL: S3 region, required for Signature Version 4 regions>
```

##### New S3 buckets

If the bucket does not exist in S3, then it will be created in the `us-east-1` region. To create the bucket in another region, configure the `region` property.

##### Existing S3 buckets

If the bucket exists in S3 and requires the [Signature Version 4 Signing Process](http://docs.aws.amazon.com/general/latest/gr/signature-version-4.html) then the `region` property must be configured. Some S3 regions require Signature Version 4, e.g. `eu-central-1`. To obtain a bucket's region run `aws s3api get-bucket-location --bucket BUCKET_NAME`.

##### AWS IAM

If you are using AWS ensure that the IAM user has the right permissions. Create a new custom policy (IAM > Policies > Create Policy > Create Your Own Policy) and paste in the following permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ServiceBackupPolicy",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:ListBucketMultipartUploads",
        "s3:ListMultipartUploadParts",
        "s3:CreateBucket",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::MY_BUCKET_NAME/*",
        "arn:aws:s3:::MY_BUCKET_NAME"
      ]
    }
  ]
}
```

The `s3:CreateBucket` permission is required because the tool will attempt to create the bucket if it does not already exist. If the desired bucket already exists, the `s3:CreateBucket` permission is not required.

Finally, attach this policy to your AWS user (IAM > Policies > Policy Actions > Attach).

##### <a id="s3"></a>AWS CLI version

The current release uses the `aws` CLI version `aws-cli/1.10.47 Python/2.7.11 Linux/4.4.0-64-generic botocore/1.4.37` to upload.

##### S3-compatible blobstores

By default, backups are sent to AWS S3. To use an S3-compatible blobstore like RiakS2, set the `endpoint_url` property.

Service Backup uses the AWS CLI to backup files to S3, or S3-compatible blobstores. If the endpoint has a self-signed SSL server certificate, then the root CA certificate must be added to the default system trust store. This can be done using [BOSH trusted certs](https://bosh.io/docs/trusted-certs.html).

#### <a id="azure"></a>Azure

```yml
properties:
  service-backup:
    destinations:
    - type: azure
      name: <OPTIONAL: destination name>
      config:
        storage_account: <storage account>
        storage_access_key: <storage key>
        container: <container name>
        path: <path in container>
```

By default, backups are sent to the public Azure blobstore. To use an on-premise blobstore, set the `blob_store_base_url` property.

##### <a id="azure"></a>Azure client version

The current release uses `blobxfer` CLI version `0.11.1` to upload.

#### <a id="gcs"></a>Google Cloud Storage

```yml
properties:
  service-backup:
    destinations:
    - type: gcs
      name: <OPTIONAL: destination name>
      config:
        service_account_json: |
          <GCP service account key JSON literal>
        project_id: <GCP project ID>
        bucket_name: <GCP Storage bucket name, does not have to already exist>
```

The service account must have "Storage Admin" IAM permissions. You can generate the JSON key for a service account with `gcloud iam service-accounts keys create key-lives-here.json --iam-account $IAM_ACCOUNT_ADDRESS`.

If the bucket does not already exist, service-backup will create it. It uses the API default attributes for the bucket, summarised here:
Bucket ACL rules: for the project that owns the configured service account, owners own the bucket, editors can write the bucket (CRUD objects), viewers can read the bucket.
Objects in bucket ACL rules: for the project that owns the configured service account, owners and editors can read/write/delete the object, viewers can read the object.
Location: US
Storage class: Standard. See [storage classes documentation](https://cloud.google.com/storage/docs/storage-classes).

If you create the bucket in advance, then you must ensure that the service account has access to write it. "Storage Admin" IAM permission should ensure this.

##### <a id="gcs"></a>Google Cloud Storage version

The current release uses the Google GCS storage golang package at [`commit  86c12b7`](https://github.com/GoogleCloudPlatform/google-cloud-go/tree/86c12b7c8187ffdfe4c155cc6bfca41ad0cabbc5) to upload.

#### <a id="scp"></a>SCP

```yml
properties:
  service-backup:
    destinations:
    - type: scp
      name: <OPTIONAL: destination name>
      config:
        user: <ssh username>
        server: <ssh server>
        destination: <path to upload to on server>
        fingerprint: <host-fingerprint> #optional
        key: |
          -----BEGIN RSA PRIVATE KEY-----
            ...
          -----END RSA PRIVATE KEY-----
        port: <optional ssh port. Defaults to 22>
```

The `fingerprint` field expects the entire output in the format returned by the `ssh-keyscan` utility for the host. If the fingerprint is provided and doesn't match, then the backup will fail. If it's empty then the fingerprint of the host will be requested right before the upload and this would be used instead. A fingerprint should be configured to prevent server spoofing or man-in-the-middle attacks. For more information refer: http://man.openbsd.org/ssh#authentication

##### <a id="scp"></a>SCP version
The current release leverages the `scp` that is included in the stemcell. Check your stemcell for its `scp` version.

#### <a id="multiple-destinations"></a>Multiple destinations

```yml
properties:
  service-backup:
    destinations:
    - type: s3
      name: <OPTIONAL: destination name>
      config:
        bucket_name: <bucket>
        bucket_path: <path in bucket>
        access_key_id: <aws access key>
        secret_access_key: <aws secret key>
    - type: scp
      name: <OPTIONAL: destination name>
      config:
        user: <ssh username>
        server: <ssh server>
        destination: <path to upload to on server>
        key: |
          -----BEGIN RSA PRIVATE KEY-----
            ...
          -----END RSA PRIVATE KEY-----
        port: <optional ssh port. Defaults to 22>
```

The tool can be provided with configuration for multiple destinations in the `destinations` property. The tool will upload backups to all the provided destinations sequentially.

You can configure multiple destinations of the same type, for example: two S3 buckets in different regions.

## <a id="operating"></a>Operating

### <a id="locating-the-backups"></a>Locating the backups

The tool will create a date-based folder structure in your destination bucket / folder as follows: `YYYY/MM/DD` and uses the BOSH VM it is running on to calculate the date. For example if your VM is using UTC time, then the folder structure will reflect this.

For example, on S3 the provided path is appended with the current date such that the resultant path is `/my/remote/path/inside/bucket/YYYY/MM/DD/` and hence the backups are accessible at `s3://my-bucket-name/my/remote/path/inside/bucket/YYYY/MM/DD/`.

### <a id="logging"></a> Logging

Service backup logs to files in `/var/vcap/sys/log/service-backup`, and also to syslog.

For forwarding syslog to a third party syslog drain (e.g. papertrail) we recommend co-locating the [syslog-release.](https://github.com/cloudfoundry/syslog-release)

### <a id="monitoring"></a> Monitoring

Here are log messages you may choose to monitor for. The log messages will appear on one line and have been formatted here for easy reading.

### <a id="troubleshooting"></a> Troubleshooting

#### ServiceBackup.Error scheduling job

This error occurs when the cron schedule is invalid, e.g. "* * * * * 99":

```json
{
  "timestamp": "1475848593.233430624",
  "source": "ServiceBackup",
  "message": "ServiceBackup.Error scheduling job",
  "log_level": 2,
  "data": {
    "error": "End of range (99) above maximum (6): 99"
  }
}
```

#### ServiceBackup.No destination provided - skipping backup

This warning will be logged when no destination is provided.

```json
{
  "timestamp": "1475848311.113728523",
  "source": "ServiceBackup",
  "message": "ServiceBackup.No destination provided - skipping backup",
  "log_level": 1,
  "data": {}
}
```

#### ServiceBackup.Perform backup completed with error

This error will be logged when performing a backup fails, e.g. when the backup command exits status 1:

```json
{
  "timestamp": "1475848125.018121719",
  "source": "ServiceBackup",
  "message": "ServiceBackup.Perform backup completed with error",
  "log_level": 2,
  "data": {
    "backup_guid": "3ebc4d08-62a3-4919-a02c-841c1afe51d0",
    "error": "exit status 1"
  }
}
```

#### ServiceBackup.Upload backup completed with error

This error will occur when uploading a backup fails. For example, when the S3 credentials provided are not authorised to create a bucket:

```json
{
  "timestamp": "1475847877.024758816",
  "source": "ServiceBackup",
  "message": "ServiceBackup.Upload backup completed with error",
  "log_level": 2,
  "data": {
    "backup_guid": "3ae36910-3c1d-4e53-b28e-62105aee1e79",
    "error": "error in create bucket: exit status 1, output: make_bucket failed: s3://doesnotexist5f7c0f16-bba4-49c9-9f60-371366edea3b An error occurred (AccessDenied) when calling the CreateBucket operation: Access Denied\n"
  }
}
```

And when the SCP host fingerprint is invalid:

```json
{
  "timestamp": "1475850125.042400837",
  "source": "ServiceBackup",
  "message": "ServiceBackup.Upload backup completed with error",
  "log_level": 2,
  "data": {
    "backup_guid": "7bd784ae-2375-43fd-bd9b-b4a43c2368c2",
    "error": "error checking if remote path exists: 'exit status 255', output: 'No ECDSA host key is known for localhost and you have requested strict checking.\r\nHost key verification failed.\r\n'"
  }
}
```

#### ServiceBackup.Backup currently in progress

This error will occur when the tool is configured with `exit_if_in_progress: true` and a backup starts whilst another backup is in progress.

```json
{
  "timestamp": "1475846665.002161264",
  "source": "ServiceBackup",
  "message": "ServiceBackup.Backup currently in progress, exiting. Another backup will not be able to start until this is completed.",
  "log_level": 2,
  "data": {
    "backup_guid": "94ca42ed-c289-4b15-a76a-23fe55d4955b",
    "error": "backup operation rejected"
  }
}
```

#### ServiceBackup.Error running

An unexpected error has occurred.
