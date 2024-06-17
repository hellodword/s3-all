# s3-all

## Providers

| Provider                | S3-compatible | IAM alike | IPv6 | restic | rclone | Path-Style | Class-A | Class-B |
| ----------------------- | ------------- | --------- | ---- | ------ | ------ | ---------- | ------- | ------- |
| minio                   | Y             | Y         |      | Y      | Y      | Y          |         |         |
| AWS S3                  | Y             | Y         |      | Y      | Y      | Y          |         |         |
| GCP cloud storage       |               |           |      |        |        |            |         |         |
| Azure                   |               |           |      |        |        |            |         |         |
| Backblaze B2            |               | Y         |      | Y      | Y      | Y          |         |         |
| Cloudflare R2           | Y             | Y         |      | Y      | Y      | Y          |         |         |
| Scaleway Object Storage | Y             | Y         |      | Y      | Y      | Y          |         |         |
| IDrive e2               | Y             | Y         |      | Y      | Y      |            |         |         |
| Telnyx Cloud Storage    | Y             | N         |      | Y      | Y      |            |         |         |
| Tencent COS             | Y             | Y         |      | Y      | Y      | N          |         |         |
| Aliyun OSS              | Y             | Y         |      | Y      | Y      | Y          |         |         |

### minio

- https://github.com/minio/minio

```yml
version: "3.8"

# docker compose exec minio sh -c 'mc config host add minio http://127.0.0.1:9000 admin password && mc admin trace minio'
services:
  minio:
    hostname: minio
    image: minio/minio:latest
    restart: "no"
    ports:
      - "9000:9000"
      - "127.0.0.1:9001:9001"
    volumes:
      - ./data/minio:/data
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=password
    healthcheck:
      test:
        [
          "CMD",
          "bash",
          "-c",
          "[ -f .health ] || (mc config host add minio http://127.0.0.1:9000 $$MINIO_ROOT_USER $$MINIO_ROOT_PASSWORD && mc ping minio -c 1 && touch .health)",
        ]
      interval: 2s
      timeout: 200s
      retries: 100
      start_period: 0s

  trace:
    depends_on:
      minio:
        condition: service_healthy
    network_mode: "service:minio"
    image: minio/minio:latest
    restart: "no"
    entrypoint: "/usr/bin/env"
    command: >
      bash -x -c 'mc config host add minio http://127.0.0.1:9000 $$MINIO_ROOT_USER $$MINIO_ROOT_PASSWORD && mc ping minio -x && mc admin trace minio'
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=password
```

### AWS S3

- https://aws.amazon.com/s3/

1. [Create bucket](https://s3.console.aws.amazon.com/s3/bucket/create?region=us-east-1) , no ACLs, no Public Access, no Bucket Versioning, no Object Lock
2. [Create Policy](https://us-east-1.console.aws.amazon.com/iam/home#/policies$new?step=edit) - JSON

> `s3:DeleteObject` is required for `restic forget --prune`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:GetObjectAcl",
        "s3:GetObject"
      ],
      "Resource": ["arn:aws:s3:::mybucket", "arn:aws:s3:::mybucket/*"]
    },
    {
      "Effect": "Allow",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::mybucket/locks/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:CreateBucket", "s3:GetBucketLocation", "s3:ListBucket"],
      "Resource": "arn:aws:s3:::mybucket"
    }
  ]
}
```

3. [IAM - Add User](https://console.aws.amazon.com/iam/home#/users$new?step=details) : `Attach policies directly`, `Create access key - Programmaticaccess` - `Use a permissions boundary to control the maximum userpermissions`

```sh
RESTIC_REPOSITORY=s3:s3.amazonaws.com/mybucket
AWS_ACCESS_KEY_ID=***
AWS_SECRET_ACCESS_KEY=***
AWS_DEFAULT_REGION=***
```

### GCP cloud storage

### Azure

### Backblaze B2

- https://www.backblaze.com/docs/cloud-storage-integrate-restic-with-backblaze-b2
- restic: https://github.com/restic/restic/blob/267cd62ae43124c80b3bed0106695eff6f7585dd/doc/030_preparing_a_new_repo.rst#backblaze-b2

1. [Create Bucket](https://secure.backblaze.com/b2_buckets.htm) , Private, no Encryption, no Object Lock
2. **important** `Lifecycle Settings` - `Keep only the last version of the file`
3. [Add a New Application Key](https://secure.backblaze.com/app_keys.htm) : `Read and Write`

```sh
RESTIC_REPOSITORY=b2:mybucket
B2_ACCOUNT_ID=***
B2_ACCOUNT_KEY=***
```

### Cloudflare R2

- https://developers.cloudflare.com/r2/
- https://github.com/cloudflare/cloudflare-docs/commit/f4386f52f42359b48c531a1fc47cbc6cfbbcfa35

1. [Create Bucket](https://dash.cloudflare.com/?to=/:account/r2/new)
2. [Create Token](https://dash.cloudflare.com/?to=/:account/r2/api-tokens)

```sh
RESTIC_REPOSITORY=s3:https://myendpoint.r2.cloudflarestorage.com/mybucket
AWS_ACCESS_KEY_ID=***
AWS_SECRET_ACCESS_KEY=***
```

### Scaleway Object Storage

- https://www.scaleway.com/en/docs/storage/object/
- https://www.scaleway.com/en/docs/tutorials/restic-s3-backup/
- https://status.scaleway.com/uptime/6q1d32x121dx

1. Create a Project, don't use the `default`, switch to the new Project
2. [Create a Bucket](https://console.scaleway.com/object-storage/buckets/create) , Private, no Bucket versioning
3. [Create a Policy](https://console.scaleway.com/iam/policies/create) , `Scope` - `Access to resources` - select the new Project, `Validate` - `ObjectStorageObjectsDelete ObjectStorageObjectsRead ObjectStorageObjectsWrite ObjectStorageReadOnly`
4. [Create a Application](https://console.scaleway.com/iam/applications/create) , select the policy, `API keys` - `Generate an API key`

### IDrive e2

- https://github.com/restic/restic/pull/4279/files
- https://status.idrivecompute.com/

1. [Create Bucket](https://app.idrivee2.com/buckets) , no Versioning, no Object Locking
2. [Create Access Key](https://app.idrivee2.com/access-key)

```sh
RESTIC_REPOSITORY="s3:https://<the endpoint>/mybucket"
AWS_ACCESS_KEY_ID=***
AWS_SECRET_ACCESS_KEY=***
```

### Telnyx Cloud Storage

- https://support.telnyx.com/en/articles/8047928-use-dragondisk-with-telnyx-storage
  > the secret key is not used by Telnyx Storage, you can type out anything you want here

1. [Create Bucket](https://portal.telnyx.com/#/app/next/storage/buckets/create) , no Bucket Versioning
2. [Create API Key](https://portal.telnyx.com/#/app/next/api-keys)

```sh
RESTIC_REPOSITORY="s3:https://us-central-1.telnyxstorage.com/mybucket"
AWS_ACCESS_KEY_ID=***
AWS_SECRET_ACCESS_KEY=anything
```

### Tencent COS

1. [创建子用户](https://console.cloud.tencent.com/cam/user/create?systemType=FastCreateV2) , 访问方式 `编程访问`, 用户权限 `-（无）`
2. [创建存储桶](https://console.cloud.tencent.com/cos/bucket) , no 版本控制, no 日志存储, no 服务端加密
3. `权限管理` - `存储桶访问权限` - `添加用户` - `数据读取、数据写入`

```sh
RESTIC_REPOSITORY="s3:https://cos.ap-singapore.myqcloud.com/mybucket"
AWS_ACCESS_KEY_ID=***
AWS_SECRET_ACCESS_KEY=***
AWS_DEFAULT_REGION="ap-singapore"
S3_VIRTUAL_HOSTED_STYLE="yes"
```

### Aliyun OSS

- restic: https://github.com/restic/restic/blob/267cd62ae43124c80b3bed0106695eff6f7585dd/doc/030_preparing_a_new_repo.rst#alibaba-cloud-aliyun-object-storage-system-oss

1. [创建 Bucket](https://oss.console.aliyun.com/bucket) , no 公共访问, no 版本控制, no 服务端加密方式, no 定时备份
2. [创建权限策略](https://ram.console.aliyun.com/policies/new) - `脚本编辑`

> `oss:DeleteObject` is required for `restic forget --prune`

```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "oss:ListObjects",
        "oss:PutObject",
        "oss:PutObjectAcl",
        "oss:GetObjectAcl",
        "oss:GetObject"
      ],
      "Resource": ["acs:oss:*:*:mybucket", "acs:oss:*:*:mybucket/*"]
    },
    {
      "Effect": "Allow",
      "Action": "oss:DeleteObject",
      "Resource": "acs:oss:*:*:mybucket/locks/*"
    },
    {
      "Effect": "Allow",
      "Action": ["oss:CreateBucket", "oss:GetBucketLocation", "oss:ListBucket"],
      "Resource": "acs:oss:*:*:mybucket"
    }
  ]
}
```

3. [RAM 创建用户](https://ram.console.aliyun.com/users/new) , `OpenAPI 调用访问`, `权限管理` - `新增授权`

```sh
RESTIC_REPOSITORY="s3:https://oss-cn-hongkong.aliyuncs.com/mybucket"
AWS_ACCESS_KEY_ID=***
AWS_SECRET_ACCESS_KEY=***
AWS_DEFAULT_REGION="oss-cn-hongkong"
S3_VIRTUAL_HOSTED_STYLE="yes"
```

### Wasabi

- https://status.wasabi.com/uptime/5w7g77w19r8z?page=1

1. [create policy](https://console.wasabisys.com/policies)

   > Same with AWS

2. [create user](https://console.wasabisys.com/users) , `Programmatic`, create group and select policy

---

## ref

- https://forum.restic.net/t/append-only-mode-with-s3-wasabi/845/4
- https://web.archive.org/web/20240616113817/https://forum.restic.net/t/minimal-rights-in-s3-amazon/4009/2
