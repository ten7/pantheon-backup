# Pantheon Backup

An Ansible role which automates backing up your sites on Pantheon to S3 or SFTP.

## Requirements

* The `terminus` command must be installed and executable by the user running the role.
* PHP must be installed for the `terminus` command to function. 
* The `s3cmd` command must be installed and executable by the user running the role.
* The `scp` command must be installed and executable by the user running the role.
* The `ssh` command must be installed and executable by the user running the role.

Please see [Pantheon's Terminus documentation](https://pantheon.io/docs/terminus/install) on how to install the command.

## Dependencies

The following collections must be installed:

* cloud.common
* amazon.aws
* community.general

## Role Variables

This role requires one dictionary as configuration, `pantheon_backup`:

```yaml
    pantheon_backup:
      terminusPath: "/usr/local/bin/terminus"
      debug: true
      stopOnFailure: false
      sources: {}
      remotes: {}
      backups: []
```

Where:
* **terminusPath** is the full path to the `terminus` executable. Optional, defaults to `terminus`.
* **debug** is `true` to enable debugging output. Optional, defaults to `false`.
* **stopOnFailure** is `true` to stop the entire role if any one backup fails. Optional, defaults to `false`.
* **sources** is a dictionary of sites and environments. Required.
* **remotes** is a dictionary of remote upload locations. Required.
* **backups** is a list of backups to perform. Required.

### Specifying Sources

In this role, "sources" specify the source from which to download backups. Each must have a unique key which is later used in the `pantheon_backup.backups` list.

```yaml
pantheon_backup:
  sources:
    example.com:
      site_id: '12345678-abcd-ef01-2345-67890abcdef0'
      machineTokenFile: "/path/to/pantheon-machine-token.txt"
      retryCount: 3
      retryDelay: 30
```

Where, in each entry:
* **site_id** is the Pantheon site ID. Required.
* **machineTokenFile** is the path to a file containing the Pantheon machine token. Optional if `machineToken` is specified.
* **machineToken** contains the Pantheon Machine token. Ignored if `machineTokenFile` is specified.
* **retryCount** is the number of time to retry `terminus` commands if they fail. Optional, defaults to `3`.
* **retryDelay** is the time in seconds to wait before retrying a failed `terminus` command. Optional, defaults to `30`.

### Specifying remotes

In this role "remotes" are upload destinations for backups. This role supports S3 or SFTP as remotes. Each remote must have a unique key which is later used in the `pantheon_backup.backups` list.

```yaml
    - hosts: servers
      vars:
        pantheon_backup:
          remotes:
            example-s3-bucket:
              type: "s3"
              bucket: "my-s3-bucket"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              region: "us-east-1"
            sftp.example.com:
              type: "sftp"
              host: "sftp.example.com"
              user: "example_user"
              keyFile: "/config/id_example_sftp"
              pubKeyFile: "/config/id_example_sftp.pub"
```

For `s3` type remotes:
* **bucket** is the name of the S3 bucket. 
* **accessKeyFile** is the path to a file containing the access key. Optional if `accessKey` is specified.
* **accessKey** is the value of the access key necessary to access the bucket. Ignored if `accessKeyFile` is specified.
* **secretKeyFile** is the path to a file containing the secret key. Optional if `secretKey` is specified.
* **secretKey** is the value of the access key necessary to secret the bucket. Ignored if `secretKeyFile` is specified.
* **region** is the AWS region in which the bucket resides. Required if using AWS S3, may be optional for other providers.
* **endpoint** is the S3 endpoint to use. Optional if using AWS, required for other providers.

For `sftp` type remotes:
* **host** is the hostname of the SFTP server. Required.
* **user** is the username necessary to login to the SFTP server. Required.
* **keyFile** is the path to a file containing the SSH private key. Required.
* **pubKeyFile** si the path to a file containing the SSH public key. Required.

### Specifying backups

The `pantheon_backup.backups` list specifies the backups to ultimately perform, referencing the `pantheon_backup.sources` and `pantheon_backup.remotes` sections for connectivity details.

```yaml
pantheon_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      env: "live"
      element: "database"
      disabled: false
      targets: []
```

Where:
* **name** is the display name of the backup. Optional, but makes the logs easier.
* **source** is the name of the key under `pantheon_backups.sources` from which to generate the backup. Required.
* **env** is the Pantheon environment to backup. Required.
* **element** is the element to backup. This can be `all`, `database`, `files`, `code`, or `db`. Required.
* **disabled** is `true` to disable (skip) the backup. Optional, defaults to `false`.
* **targets** is a list of remotes and additional destination information about where to upload backups. Required.

### Backup targets

Backup targets reference a key in `pantheon_backup.remotes`, and combine that with additional information used to upload this specific backup.

```yaml
pantheon_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      env: "live"
      element: "database"
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          disabled: false
```

Where:
* **remote** is the key under `pantheon_backup.remotes` to use when uploading the backup. Required.
* **path** is the path on the remote to upload the backup. Optional.
* **disabled** is `true` to skip uploading to the specifed `remote`. Optional, defaults to `false`.

### Rotating backups

Backups are uplaoded to the remote with the `&lt;site_id&gt;.&lt;env&gt;.&lt;element&gt;-0.tar.gz`. Often, you'll want to retain previous backups in the case an older backup can aid in research or recovery. This role supports retaining and rotating multiple backups using the `retainCount` key.

```yaml
pantheon_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      env: "live"
      element: "database"
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          retainCount: 3
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          retainCount: 3
          disabled: false
```

Where:
* **retainCount** is the total number of backups to retain in the directory. Optional. Defaults to `1`, or no rotation.

During a backup, if `retainCount` is set:
1. The backup with the ending `&lt;retainCount - 1&gt;.tar.gz` is deleted.
2. Starting with `&lt;retainCount - 2&gt;.tar.gz`, each backup is renamed incremending the ending index.
3. The new backup is uploaded with a `0` index as `&lt;site_id&gt;.&lt;env&gt;.&lt;element&gt;-0.tar.gz`.

This feature works both in S3 and SFTP.

### Creating a backup when the role runs

By default, this role does **not** create a backup when it runs. For live sites on Pantheon, scheduled backups are already configured for the `live` environment. This removes the need for this role to wait and generate the backup on Pantheon prior to downloading it.

In some cases, however, you may need to force Pantheon to create the backup from this role on demand. In that case, you can use the `createBackupNow` when defining you backup:


```yaml
pantheon_backup:
  backups:
    - name: "example.com files"
      source: "example.com"
      env: "live"
      element: "files"
      createBackupNow: true
      targets: []
```

Where:
* **createBackupNow** is `true` to instruct the role to create a new backup on demand. Optional, defaults to `false`.

It is highly recommended to rely on Pantheon's scheduled backups and set this option to `false` in except in special circumstances.

## Example Playbook

```yaml
    - hosts: servers
      vars:
        pantheon_backup:
          terminusPath: "/usr/local/bin/terminus"
          debug: true
          sources:
            example.com:
              site_id: '12345678-abcd-ef01-2345-67890abcdef0'
              machineTokenFile: "/path/to/pantheon-machine-token.txt"
              retryCount: 3
              retryDelay: 30
          remotes:
            example-s3-bucket:
              type: "s3"
              bucket: "my-s3-bucket"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              region: "us-east-1"
            sftp.example.com:
              type: "sftp"
              host: "sftp.example.com"
              user: "example_user"
              keyFile: "/config/id_example_sftp"
              pubKeyFile: "/config/id_example_sftp.pub"
          backups:
            - name: "example.com database"
              source: "example.com"
              env: "live"
              element: "database"
              createBackupNow: true
              targets:
                - remote: "example-s3-bucket"
                  path: "example.com/database"
                  retainCount: 3
                  disabled: true
                - remote: "sftp.example.com"
                  path: "backups/example.com/database"
                  retainCount: 3
                  disabled: false
            - name: "example.com files"
              source: "example.com"
              env: "live"
              element: "files"
              createBackupNow: true
              targets:
                - remote: "example-s3-bucket"
                  path: "example.com/files"
                  retainCount: 3
                  disabled: true
                - remote: "sftp.example.com"
                  path: "backups/example.com/files"
                  retainCount: 3
                  disabled: false
      roles:
         - { role: ten7.pantheon_deploy }
```

## License

GPL v3

## Author Information

This role was created by [TEN7](https://ten7.com/).
