---
title: 外部存储
summary: 了解 BR、TiDB Lightning 和 Dumpling 中所用外部存储服务的 URL 参数。
aliases: ['/docs-cn/dev/br/backup-and-restore-storages/']
---

# 外部存储

Backup & Restore (BR)、TiDB Lightning 和 Dumpling 都支持在本地文件系统和 Amazon S3 上读写数据；另外 BR 还支持 Google Cloud Storage、Azure Blob Storage (Azblob)。通过传入不同 URL scheme 到 BR 的 `--storage` (`-s`) 参数、TiDB Lightning 的 `-d` 参数及 Dumpling 中的 `--output` (`-o`) 参数，可以区分不同的存储方式。

## Scheme

TiDB 迁移工具支持以下存储服务：

| 服务 | Scheme | 示例 |
|---------|---------|-------------|
| 本地文件系统（分布在各节点上） | local | `local:///path/to/dest/` |
| Amazon S3 及其他兼容 S3 的服务 | s3 | `s3://bucket-name/prefix/of/dest/` |
| GCS | gcs, gs | `gcs://bucket-name/prefix/of/dest/` |
| Azure Blob Storage | azure, azblob | `azure://container-name/prefix/of/dest/` |
| 不写入任何存储（仅作为基准测试） | noop | `noop://` |

## URL 参数

S3、 GCS 和 Azblob 等云存储有时需要额外的连接配置，你可以为这类配置指定参数。例如：

* 用 Dumpling 导出数据到 S3：

    ```bash
    ./dumpling -u root -h 127.0.0.1 -P 3306 -B mydb -F 256MiB \
        -o 's3://my-bucket/sql-backup'
    ```

* 用 TiDB Lightning 从 S3 导入数据：

    ```bash
    ./tidb-lightning --tidb-port=4000 --pd-urls=127.0.0.1:2379 --backend=local --sorted-kv-dir=/tmp/sorted-kvs \
        -d 's3://my-bucket/sql-backup'
    ```

* 用 TiDB Lightning 从 S3 导入数据（使用路径类型的请求模式）：

    ```bash
    ./tidb-lightning --tidb-port=4000 --pd-urls=127.0.0.1:2379 --backend=local --sorted-kv-dir=/tmp/sorted-kvs \
        -d 's3://my-bucket/sql-backup?force-path-style=true&endpoint=http://10.154.10.132:8088'
    ```

* 用 TiDB Lightning 从 S3 导入数据（使用特定 IAM 角色来访问 S3 数据）：

    ```bash
    ./tidb-lightning --tidb-port=4000 --pd-urls=127.0.0.1:2379 --backend=local --sorted-kv-dir=/tmp/sorted-kvs \
        -d 's3://my-bucket/test-data?role-arn=arn:aws:iam::888888888888:role/my-role'
    ```

* 用 BR 备份到 GCS：

    ```bash
    ./br backup full -u 127.0.0.1:2379 \
    --storage 'gcs://bucket-name/prefix'
    ```

* 用 BR 备份到 Azblob：

    ```bash
    ./br backup full -u 127.0.0.1:2379 \
    --storage 'azure://container-name/prefix'
    ```

### S3 的 URL 参数

| URL 参数 | 描述 |
|:----------|:---------|
| `access-key` | 访问密钥 |
| `secret-access-key` | secret 访问密钥 |
| `use-accelerate-endpoint` | 是否在 Amazon S3 上使用加速端点（默认为 `false`） |
| `endpoint` | S3 兼容服务自定义端点的 URL（例如 `https://s3.example.com/`）|
| `force-path-style` | 使用 path-style，而不是 virtual-hosted style（默认为 `true`） |
| `storage-class` | 上传对象的存储类别（例如 `STANDARD`、`STANDARD_IA`） |
| `sse` | 用于加密上传的服务器端加密算法（可以设置为空、`AES256` 或 `aws:kms`） |
| `sse-kms-key-id` | 如果 `sse` 设置为 `aws:kms`，则使用该参数指定 KMS ID |
| `acl` | 上传对象的 canned ACL（例如，`private`、`authenticated-read`） |
| `role-arn` | 当需要使用特定的 [IAM 角色](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_roles.html)来访问第三方 Amazon S3 的数据时，使用这个参数来指定 IAM 角色的对应 [Amazon Resource Name (ARN)](https://docs.aws.amazon.com/zh_cn/general/latest/gr/aws-arns-and-namespaces.html)（例如 `arn:aws:iam::888888888888:role/my-role`）。关于使用 IAM 角色访问第三方 Amazon S3 数据的场景，请参考 [AWS 相关文档介绍](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_roles_common-scenarios_third-party.html)。 |
| `external-id` | 当需要使用特定的 [IAM 角色](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_roles.html)来访问第三方 Amazon S3 的数据时，可能需要同时提供正确的[外部 ID](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html) 来确保用户有权限代入该 IAM 角色。这个参数用来指定对应的外部 ID，使得代入 IAM 角色能够顺利进行。外部 ID 可以是任意字符串，并且不是必须的，一般由控制 Amazon S3 数据访问的第三方来指定。如果第三方对于 IAM 角色没有指定特定的外部 ID，则可以不需要提供该参数也能顺利代入对应的 IAM 角色，从而访问对应的 Amazon S3 数据。 |

> **注意：**
>
> 不建议在存储 URL 中直接传递访问密钥和 secret 访问密钥，因为这些密钥是明文记录的。

如果没有指定访问密钥和 secret 访问密钥，并且也没有提供特定的 IAM 角色 ARN (`role-arn`) 和外部 ID (`external-id`)，迁移工具尝试按照以下顺序从环境中推断这些密钥：

1. `$AWS_ACCESS_KEY_ID` 和 `$AWS_SECRET_ACCESS_KEY` 环境变量
2. `$AWS_ACCESS_KEY` 和 `$AWS_SECRET_KEY` 环境变量
3. 工具节点上的共享凭证文件，路径由 `$AWS_SHARED_CREDENTIALS_FILE` 环境变量指定
4. 工具节点上的共享凭证文件，路径为 `~/.aws/credentials`
5. 当前 Amazon EC2 容器的 IAM 角色
6. 当前 Amazon ECS 任务的 IAM 角色

### GCS 的 URL 参数

| URL 参数 | 描述 |
|:----------|:---------|
| `credentials-file` | 迁移工具节点上的凭证 JSON 文件的路径 |
| `storage-class` | 上传对象的存储类别（例如 `STANDARD` 或 `COLDLINE`） |
| `predefined-acl` | 上传对象的预定义 ACL（例如 `private` 或 `project-private`） |

如果没有指定 `credentials-file`，迁移工具尝试按照以下顺序从环境中推断出凭证：

1. 工具节点上位于 `$GOOGLE_APPLICATION_CREDENTIALS` 环境变量所指定路径的文件内容
2. 工具节点上位于 `~/.config/gcloud/application_default_credentials.json` 的文件内容
3. 在 GCE 或 GAE 中运行时，从元数据服务器中获取的凭证

### Azblob 的 URL 参数

| URL 参数 | 描述 |
|:----------|:-----|
| `account-name` | 存储账户名 |
| `account-key` | 访问密钥 |
| `access-tier` | 上传对象的存储类别（例如 `Hot`、`Cool`、`Archive`）。如果没有设置 `access-tier` 的值（该值为空），此值会默认设置为 `Hot`。 |

为了保证 TiKV 和迁移工具使用了同一个存储账户，`account-name` 会由迁移工具决定（即默认 `send-credentials-to-tikv = true`）。迁移工具按照以下顺序推断密钥：

1. 如果已指定 `account-name` **和** `account-key`，则使用该参数指定的密钥。
2. 如果没有指定 `account-key`，则尝试从工具节点上的环境变量读取相关凭证。迁移工具会优先读取 `$AZURE_CLIENT_ID`、`$AZURE_TENANT_ID` 和 `$AZURE_CLIENT_SECRET`。与此同时，工具会允许 TiKV 从各自节点上读取上述三个环境变量，采用 `Azure AD` (Azure Active Directory) 访问。
3. 如果上述的三个环境变量不存在于工具节点中，则尝试读取 `$AZURE_STORAGE_KEY`，采用密钥访问。

> **注意：**
>
> - 将 Azure Blob Storage 作为外部存储时，必须设置 `send-credentials-to-tikv = true`（即默认情况），否则会导致备份失败。
> - `$AZURE_CLIENT_ID`、`$AZURE_TENANT_ID` 和 `$AZURE_CLIENT_SECRET` 分别代表 Azure 应用程序的应用程序 ID `client_id`、租户 ID `tenant_id` 和客户端密码 `client_secret`。如需了解如何确认运行环境中存在环境变量 `$AZURE_CLIENT_ID`、`$AZURE_TENANT_ID` 和 `$AZURE_CLIENT_SECRET`，或需要将环境变量配置为参数，请参考[鉴权](/br/backup-and-restore-storages.md#鉴权) Azure Blob Storage 部分。

## 命令行参数

除了使用 URL 参数，BR 和 Dumpling 工具也支持从命令行指定这些配置，例如：

```bash
./dumpling -u root -h 127.0.0.1 -P 3306 -B mydb -F 256MiB \
    -o 's3://my-bucket/sql-backup' \
    --s3.role-arn="arn:aws:iam::888888888888:role/my-role"
```

上述命令也等效于：

```bash
./dumpling -u root -h 127.0.0.1 -P 3306 -B mydb -F 256MiB \
     -o 's3://my-bucket/sql-backup&role-arn=arn:aws:iam::888888888888:role/my-role'
```

如果同时指定了 URL 参数和命令行参数，命令行参数会覆盖 URL 参数。

### S3 的命令行参数

| 命令行参数 | 描述 |
|:----------|:------|
| `--s3.endpoint` | S3 兼容服务自定义端点的 URL（例如 `https://s3.example.com/`）|
| `--s3.storage-class` | 上传对象的存储类别（例如 `STANDARD` 或 `STANDARD_IA`） |
| `--s3.sse` | 用于加密上传的服务器端加密算法（可以设置为空、`AES256` 或 `aws:kms`） |
| `--s3.sse-kms-key-id` | 如果 `--s3.sse` 设置为 `aws:kms`，则使用该参数指定 KMS ID |
| `--s3.acl` | 上传对象的 canned ACL（例如，`private` 或 `authenticated-read`） |
| `--s3.provider` | S3 兼容服务类型（支持 `aws`、`alibaba`、`ceph`、`netease` 或 `other`） |
| `--s3.role-arn` | 当需要使用特定的 [IAM 角色](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_roles.html)来访问第三方 Amazon S3 的数据时，使用这个可选参数来指定 IAM 角色的对应 [Amazon Resource Name (ARN)](https://docs.aws.amazon.com/zh_cn/general/latest/gr/aws-arns-and-namespaces.html)（例如 `arn:aws:iam::888888888888:role/my-role`）。关于使用 IAM 角色访问第三方 Amazon S3 数据的场景，请参考 [AWS 相关文档介绍](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_roles_common-scenarios_third-party.html)。 |
| `--s3.external-id` | 当需要使用特定的 [IAM 角色](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_roles.html)来访问第三方 Amazon S3 的数据时，可能需要同时提供正确的[外部 ID](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html) 来确保用户有权限代入该 IAM 角色。这个可选参数用来指定对应的外部 ID，使得代入 IAM 角色能够顺利进行。外部 ID 可以是任意字符串，并且不是必须的，一般由控制 Amazon S3 数据访问的第三方来指定。如果第三方对于 IAM 角色没有指定特定的外部 ID，则可以不需要提供该参数也能顺利代入对应的 IAM 角色，从而访问对应的 Amazon S3 数据。 |

如果要将数据导出到非 AWS 的 S3 云存储，你需要指定云服务商名字，以及是否使用 virtual-hosted style。将数据导出至阿里云的 OSS 存储为例：

* 使用 Dumpling 将数据导出至 OSS 存储：

    ```bash
    ./dumpling -h 127.0.0.1 -P 3306 -B mydb -F 256MiB \
       -o "s3://my-bucket/dumpling/" \
       --s3.endpoint="http://oss-cn-hangzhou-internal.aliyuncs.com" \
       --s3.provider="alibaba" \
       -r 200000 -F 256MiB
    ```

* 使用 BR 将数据备份至 OSS 存储：

    ```bash
    ./br backup full --pd "127.0.0.1:2379" \
        --storage "s3://my-bucket/full/" \
        --s3.endpoint="http://oss-cn-hangzhou-internal.aliyuncs.com" \
        --s3.provider="alibaba" \
        --send-credentials-to-tikv=true \
        --ratelimit 128 \
        --log-file backuptable.log
    ```

* 在 YAML 文件中指定 TiDB Lightning 将数据导出至 OSS 存储：

    ```
    [mydumper]
    data-source-dir = "s3://my-bucket/dumpling/?endpoint=http://oss-cn-hangzhou-internal.aliyuncs.com&provider=alibaba"
    ```

### GCS 的命令行参数

| 命令行参数 | 描述 |
|:----------|:---------|
| `--gcs.credentials-file` | 迁移工具节点上的凭证 JSON 文件的路径 |
| `--gcs.storage-class` | 上传对象的存储类别（例如 `STANDARD` 或 `COLDLINE`） |
| `--gcs.predefined-acl` | 上传对象的预定义 ACL（例如 `private` 或 `project-private`） |

### Azblob 的命令行参数

| 命令行参数 | 描述 |
|:----------|:-------|
| `--azblob.account-name` | 存储账户名 |
| `--azblob.account-key` | 访问密钥 |
| `--azblob.access-tier` | 上传对象的存储类别（例如 `Hot`、`Cool` 或 `Archive`）。如果没有设置 `access-tier` 的值（该值为空），此值会默认设置为 `Hot`。 |

## BR 向 TiKV 发送凭证

在默认情况下，使用 S3、GCS 或 Azblob 存储时，BR 会将凭证发送到每个 TiKV 节点，以减少设置的复杂性。

但是，这个操作不适合云端环境，因为每个节点都有自己的角色和权限。在这种情况下，你需要用 `--send-credentials-to-tikv=false`（或简写为 `-c=0`）来禁止发送凭证：

```bash
./br backup full -c=0 -u pd-service:2379 --storage 's3://bucket-name/prefix'
```

使用 SQL 进行[备份](/sql-statements/sql-statement-backup.md)[恢复](/sql-statements/sql-statement-restore.md)时，可加上 `SEND_CREDENTIALS_TO_TIKV = FALSE` 选项：

```sql
BACKUP DATABASE * TO 's3://bucket-name/prefix' SEND_CREDENTIALS_TO_TIKV = FALSE;
```

此参数不适用于 TiDB Lightning 和 Dumpling，因为目前它们都是单机程序。
