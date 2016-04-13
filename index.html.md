# Service Backup

BOSH operators running services (e.g. Redis service broker for Cloud Foundry) may want to back up certain files from the virtual machines running these services so that they can restore them after a disaster.

# Service Backup BOSH release

The Service Backup BOSH release backs up a directory on the instance VM it is located on to one of several supported destination types. The supported destination types are AWS S3, Azure blobstore, and SCP.

## Uploading a service backup release

Service Backup is distributed as a BOSH final release. To upload a release to your BOSH director, upload the latest final release tarball from [Github releases](https://github.com/pivotal-cf-experimental/service-backup-release/releases).

## Configuring the manifest

Service Backup is designed to be co-located on service instance VMs, and must be included in that service's BOSH deployment manifest.

### Adding service backups to your service deployment

Shown below is an template manifest, adding an S3 backup to a Redis service deployment. For further information on changing the backup destination, see [link].

```yml
---
properties:
  service-backup:
    destination:
      s3:
        bucket_name: <bucket>
        bucket_path: <path in bucket>
        access_key_id: <aws access key>
        secret_access_key: <aws secret key>
    source_folder: <directory to back up>
    source_executable: <run before each backup>
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

- **source_executable**

  Executable to run before each backup. This is useful for services that require some operation to be performed before backing files up, for example triggering a Redis memory dump to disk. This is mandatory at the moment, so use a dummy when this is not required (e.g. `true`).

- **cron_schedule**

  Backups will be triggered according to this schedule. See [robfig/cron](https://godoc.org/github.com/robfig/cron) for cron expression syntax.

The service-backup is co-located on the `redis-server` instance group.
