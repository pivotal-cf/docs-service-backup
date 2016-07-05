---
title: Service Backups for Pivotal Cloud Foundry&reg;
author: London Enablement Team

---

BOSH operators running services (e.g. Redis service broker for Cloud Foundry) may want to back up certain files from the virtual machines running these services so that they can restore them after a disaster.

<a id="configuration"></a>
## Configuration

The Service Backup BOSH release backs up a directory on the instance VM it is located on to one of several supported destination types. The supported destination types are AWS S3, Azure blobstore, and SCP.

<a id="uploading"></a>
### Uploading a service backup release

Service Backup is distributed as a BOSH final release. To upload a release to your BOSH director, upload the latest final release tarball from [Github releases](https://github.com/pivotal-cf-experimental/service-backup-release/releases), or from [S3](https://s3-eu-west-1.amazonaws.com/cf-services-external-builds/service-backup/final/).

<a id="configuring"></a>
### Configuring the manifest

Service Backup is designed to be co-located on service instance VMs, and must be included in that service's BOSH deployment manifest.

<a id="adding-service"></a>
#### Adding service backups to your service deployment

Shown below is an template manifest, adding an S3 backup destination to a Redis service deployment. For further information on changing the backup destination, see [link].

```yml
---
properties:
  service-backup:
    destinations:
    - type: s3
      name: <destination name> # optional
      config:
        bucket_name: <bucket>
        bucket_path: <path in bucket>
        access_key_id: <aws access key>
        secret_access_key: <aws secret key>
    source_folder: <directory to back up>
    source_executable: <run before each backup> # optional
    cron_schedule: <cron schedule>

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

<a id="configuring-backup-schedule"></a>
#### Configuring the backup schedule

Backups will be triggered according to the schedule given by the `cron_schedule` property. See [robfig/cron](https://godoc.org/github.com/robfig/cron) for cron expression syntax.

<a id="defining-files"></a>
#### Defining the files to be be backed up

The `source_folder` property names a local path from which backups are uploaded. All files in here are uploaded.

<a id="preparing-files"></a>
#### Preparing the files to be backed up

The `source_executable` property names an executable to run before each backup. This is useful for services that require some operation to be performed before backing files up, for example triggering a Redis memory dump to disk. If a suitable executable is not included in the service release, you can add one by publishing it in a separate release, as its own package and job, and colocating it into the deployment. Tokens are split on spaces; first is command to execute and remaining are passed as args to command. This property is optional. If the field is not specified, it will simply be ignored and nothing will be executed. 

<a id="cleaning-files"></a>
#### Cleaning up the files which were backed up

The optional `cleanup_executable` property names a local executable to cleanup backups. Tokens are split on spaces; first is command to execute and remaining are passed as args to command.

<a id="correlating"></a>
#### Correlating BOSH instances to Cloud Foundry service instances

BOSH operators might want to correlate BOSH-deployed VM instances with CF service instances, in which case the Service Author must provide a binary that returns a string identifier for your service instance. This will appear in all log messages under the data element `identifier`. For e.g.

`{ "source": "ServiceBackup", "message": "doing-stuff", "data": { "backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61", "identifier": "service_identifier" }, "timestamp": 1232345, "log_level": 1 }`

Add the optional `service_identifier_executable` key to your manifest (tokens are split on spaces; first is command to execute and remaining are passed as args to command):

```yml
properties:
  service-backup:
    service_identifier_executable: replace-with-service-identifier-executable #optional
```

<a id="naming-backup-destinations"></a>
#### Naming backup destinations

Each destination can be given an optional `name` property. This will appear in the log messages for uploads to that destination. For example:

```
{ "timestamp":"1467629245.010814428", "source":"ServiceBackup", "message":"ServiceBackup.WithIdentifier.about to upload /path/to/file to S3 remote path bucket_name/2016/07/04", "log_level":1, "data": { "backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61", "destination_name": "some-destination-name", "identifier": "service_identifier", "session":"1" } }
```

<a id="identifying-logs-for-a-backup-run"></a>
#### Identifying logs for a backup run

The log lines of a particular backup run can be identified by correlating their unique `backup_guid` For example

```
{"timestamp":"1467649696.229321241","source":"ServiceBackup","message":"ServiceBackup.WithIdentifier.Upload backup started","log_level":1,"data":{"backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61","identifier":"service_identifier","session":"1"}}
{"timestamp":"1467649696.229343414","source":"ServiceBackup","message":"ServiceBackup.WithIdentifier.Upload backup completed successfully","log_level":1,"data":{"backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61","duration_in_seconds":8.081000000000001e-06,"identifier":"service_identifier","session":"1","size_in_bytes":200
{"timestamp":"1467649696.229365349","source":"ServiceBackup","message":"ServiceBackup.WithIdentifier.Cleanup started","log_level":1,"data":{"backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61","identifier":"service_identifier","session":"1}}
{"timestamp":"1467649696.232805967","source":"ServiceBackup","message":"ServiceBackup.WithIdentifier.Cleanup debug info","log_level":0,"data":{"backup_guid":"244eadb0-91e7-45da-9a7f-3616a59a6e61","cmd":"creator-cmd","identifier":"service_identifier","out":"Cleanup Complete\n","session":"1"}}
```


<a id="manual-backup"></a>
#### Triggering manual service backups

BOSH operators might want to trigger a one-off, manual backup. To do this:

1. ssh onto a BOSH-deployed VM that has a service-backup job running on it.
1. Execute `/var/vcap/jobs/service-backup/bin/manual-backup`.

<a id="disabling"></a>
#### Disabling Service Backups

Backups can be disabled by removing the `service-backup` section from your manifest and then redeploying. You can still leave the job on your instance group if you wish.

<a id="backup-destinations"></a>
### Backup destinations

Service Backup supports S3 (AWS, Ceph s3, Swift w/ S3 compatibility module), Azure blobstore, and SCP. To change the backup destination change the manifest `destinations` value:

<a id="s3"></a>
#### S3

```yml
properties:
  service-backup:
    destinations:
    - type: s3
      config:
        bucket_name: <bucket>
        bucket_path: <path in bucket>
        access_key_id: <aws access key>
        secret_access_key: <aws secret key>
```

##### AWS S3

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

By default, backups are sent to AWS S3. To use an S3-compatible blobstore like RiakS2, set the `endpoint_url` property.

<a id="azure"></a>
#### Azure

```yml
properties:
  service-backup:
    destinations:
    - type: azure
      config:
        storage_account: <storage account>
        storage_access_key: <storage key>
        container: <container name>
        path: <path in container>
```

By default, backups are sent to the public Azure blobstore. To use an on-premise blobstore, set the `blob_store_base_url` property.

<a id="scp"></a>
#### SCP

```yml
properties:
  service-backup:
    destinations:
    - type: scp
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

<a id="multiple-destinations"></a>
#### Multiple destinations

```yml
properties:
  service-backup:
    destinations:
    - type: s3
      config:
        bucket_name: <bucket>
        bucket_path: <path in bucket>
        access_key_id: <aws access key>
        secret_access_key: <aws secret key>
    - type: scp
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

<a id="operating"></a>
## Operating

<a id="locating-the-backups"></a>
### Locating the backups

The tool will create a date-based folder structure in your destination bucket / folder as follows: `YYYY/MM/DD` and uses the BOSH VM it is running on to calculate the date. For example if your VM is using UTC time, then the folder structure will reflect this.

For example, on S3 the provided path is appended with the current date such that the resultant path is `/my/remote/path/inside/bucket/YYYY/MM/DD/` and hence the backups are accessible at `s3://my-bucket-name/my/remote/path/inside/bucket/YYYY/MM/DD/`.
