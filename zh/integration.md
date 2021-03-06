# 集成

> 贡献者：[@morefreeze](https://github.com/morefreeze) [@zhongjiajie](https://github.com/zhongjiajie)

* [反向代理](#反向代理)
* [Azure：Microsoft Azure](#azuremicrosoft-azure)
* [AWS：亚马逊网络服务](#aws亚马逊网络服务)
* [Databricks](#databricks)
* [GCP：Google云端平台](#gcpgoogle云端平台)

## 反向代理

Airflow 可以通过设置反向代理，使其可以灵活设置访问地址。

例如，您可以这样配置反向代理：

```
https://lab.mycompany.com/myorg/airflow/
```

为此，您需要在`airflow.cfg中`设置：

```config
base_url = http://my_host/myorg/airflow
```

此外，如果您使用 Celery Executor，您可以配置`myorg/flower`的地址：

```config
flower_url_prefix = /myorg/flower
```

您的反向代理（例如：nginx）应配置如下：

*   传递 url 和 http 头给 Airflow 服务器，不需要重写，例如：

    ```
    server {
      listen 80;
      server_name lab.mycompany.com;

      location /myorg/airflow/ {
          proxy_pass http://localhost:8080;
          proxy_set_header Host $host;
          proxy_redirect off;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
      }
    }
    ```

*   重写 flower 的地址：

    ```py
    server {
        listen 80;
        server_name lab.mycompany.com;

        location /myorg/flower/ {
            rewrite ^/myorg/flower/(.*)$ /$1 break;  # remove prefix from http header
            proxy_pass http://localhost:5555;
            proxy_set_header Host $host;
            proxy_redirect off;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }

    ```

## Azure：Microsoft Azure

Airflow 对 Microsoft Azure 的支持有限：仅支持 Azure Blob Storage 和 Azure Data Lake 的接口。 对 Blob Storage 的 Hook, Sensor 和 Operator 以及 Azure Data Lake 的 Hook 都在 contrib 部分。

### Azure Blob Storage

所有类都通过 Window Azure Storage Blob 协议进行通信。 确保 Airflow 的 wasb 连接存在。 可以通过在额外字段中指定登录用户名（=Storage account name）和密码（=KEY），或用 SAS 令牌来完成授权（参看 wasb_default 连接例子）。

*   [WasbBlobSensor](#WasbBlobSensor) ：检查一个 blob 是否在 Azure Blob Storage 上。
*   [WasbPrefixSensor](#WasbPrefixSensor) ：检查满足前缀匹配的 blob 是否在 Azure Blob Storage 上。
*   [FileToWasbOperator](#FileToWasbOperator) ：将本地文件作为blob 上传到容器。
*   [WasbHook](#WasbHook) ：与 Azure Blob Storage 的接口。

#### WasbBlobSensor

```py
class airflow.contrib.sensors.wasb_sensor.WasbBlobSensor(container_name, blob_name, wasb_conn_id='wasb_default', check_options=None, *args, **kwargs)
```

基类： `airflow.sensors.base_sensor_operator.BaseSensorOperator`

等待 blob 到达 Azure Blob Storage。

参数：

*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `wasb_conn_id(str)` - 对 wasb 连接的引用。
*   `check_options(dict)` - 传给`WasbHook.check_for_blob()`的可选关键字参数。


```py
poke(context)
```

Operator 在继承此类时应该覆盖以上函数。

#### WasbPrefixSensor

```py
class airflow.contrib.sensors.wasb_sensor.WasbPrefixSensor(container_name, prefix, wasb_conn_id='wasb_default', check_options=None, *args, **kwargs)
```

基类： `airflow.sensors.base_sensor_operator.BaseSensorOperator`

等待前缀匹配的 blob 到达 Azure Blob Storage。

参数：

*   `container_name(str)` - 容器的名称。
*   `prefix(str)` - blob 的前缀。
*   `wasb_conn_id(str)` - 对 wasb 连接的引用。
*   `check_options(dict)` - 传给`WasbHook.check_for_prefix()`的可选关键字参数。


```py
poke(context)
```

Operator 在继承此类时应该覆盖以上函数。

#### FileToWasbOperator

```py
class airflow.contrib.operators.file_to_wasb.FileToWasbOperator(file_path, container_name, blob_name, wasb_conn_id='wasb_default', load_options=None, *args, **kwargs)
```

基类： `airflow.models.BaseOperator`

将文件上传到 Azure Blob Storage。

参数：

*   `file_path(str)` - 要加载的文件的路径。（模板渲染后）
*   `container_name(str)` - 容器的名称。（模板渲染后）
*   `blob_name(str)` - blob 的名称。（模板渲染后）
*   `wasb_conn_id(str)` - 对 wasb 连接的引用。
*   `load_options(dict)` - 传给`WasbHook.load_file()`的可选关键字参数。


```py
execute(context)
```

将文件上传到 Azure Blob Storage。

#### WasbHook

```py
class airflow.contrib.hooks.wasb_hook.WasbHook(wasb_conn_id='wasb_default')
```

基类： `airflow.hooks.base_hook.BaseHook`

通过 wasb:// 协议与 Azure Blob Storage进行交互。

在连接的"extra"字段中传递的其他参数将传递给`BlockBlockService()`构造函数。 例如，通过添加{"sas_token"："YOUR_TOKEN"}使用 SAS 令牌进行身份验证。

参数：`wasb_conn_id(str)` - 对 wasb 连接的引用。


```py
check_for_blob(container_name, blob_name, **kwargs)
```

检查 Azure Blob Storage 上是否存在 Blob。

参数：

*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `kwargs(object)` - 传给`BlockBlobService.exists()`的可选关键字参数。

返回：如果 blob 存在则为 True，否则为 False。

返回类型：bool

```py
check_for_prefix(container_name, prefix, **kwargs)
```

检查 Azure Blob Storage 上是否存在前缀匹配的 blob。

参数：

*   `container_name(str)` - 容器的名称。
*   `prefix(str)` - blob 的前缀。
*   `kwargs(object)` - 传给`BlockBlobService.list_blobs()`的可选关键字参数。

返回：如果存在与前缀匹配的 blob，则为 True，否则为 False。

返回类型：bool

```py
get_conn()
```

返回 BlockBlobService 对象。

```py
get_file(file_path, container_name, blob_name, **kwargs)
```

从 Azure Blob Storage 下载文件。

参数：

*   `file_path(str)` - 要下载的文件的路径。
*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `kwargs(object)` - 传给`BlockBlobService.create_blob_from_path()`的可选关键字参数。


```py
load_file(file_path, container_name, blob_name, **kwargs)
```

将文件上传到 Azure Blob Storage。

参数：

*   `file_path(str)` - 要加载的文件的路径。
*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `kwargs(object)` - 传给`BlockBlobService.create_blob_from_path()`的可选关键字参数。


```py
load_string(string_data, container_name, blob_name, **kwargs)
```

将字符串上传到 Azure Blob Storage。

参数：

*   `string_data(str)` - 要上传的字符串。
*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `kwargs(object)` - 传给`BlockBlobService.create_blob_from_text()`的可选关键字参数。


```py
read_file(container_name, blob_name, **kwargs)
```

从 Azure Blob Storage 读取文件并以字符串形式返回。

参数：

*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `kwargs(object)` - 传给`BlockBlobService.create_blob_from_path()`的可选关键字参数。


### Azure File Share

SMB 文件共享的云变体。 确保 Airflow 存在类型为`wasb`的连接。 可以通过在额外字段中提供登录（=Storage 帐户名称）和密码（=Storage 帐户密钥）或通过 SAS 令牌来完成授权（请参阅连接`wasb_default`示例）。

#### AzureFileShareHook

```py
class airflow.contrib.hooks.azure_fileshare_hook.AzureFileShareHook(wasb_conn_id='wasb_default')
```

基类： `airflow.hooks.base_hook.BaseHook`

与 Azure FileShare Storage 交互。

在连接的"extra"字段中传递的参数将传递给`FileService()`构造函数。

参数：`wasb_conn_id(str)` - 对 wasb 连接的引用。


```py
check_for_directory(share_name, directory_name, **kwargs)
```

检查 Azure File Share 上是否存在目录。

参数：

*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `kwargs(object)` - 传给`FileService.exists()`的可选关键字参数。

返回：如果文件存在则为 True，否则为 False。

返回类型：bool

```py
check_for_file(share_name, directory_name, file_name, **kwargs)
```

检查 Azure File Share 上是否存在文件。

参数：

*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `kwargs(object)` - 传给`FileService.exists()`的可选关键字参数。

返回：如果文件存在则为 True，否则为 False。

返回类型：bool

```py
create_directory(share_name, directory_name, **kwargs)
```

在 Azure File Share 上创建目录。

参数：

*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `kwargs(object)` - 传给`FileService.create_directory()`的可选关键字参数。

返回：文件和目录列表

返回类型：list

```py
get_conn()
```

返回 FileService 对象。

```py
get_file(file_path, share_name, directory_name, file_name, **kwargs)
```

从 Azure File Share 下载文件。

参数：

*   `file_path(str)` - 存储文件的位置。
*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `kwargs(object)` - 传给`FileService.get_file_to_path()`的可选关键字参数。


```py
get_file_to_stream(stream, share_name, directory_name, file_name, **kwargs)
```

以流的形式从 Azure File Share 下载文件。

参数：

*   `stream(类文件对象)` - 用于存储文件的文件句柄。
*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `kwargs(object)` - 传给`FileService.get_file_to_stream()`的可选关键字参数。


```py
list_directories_and_files(share_name, directory_name=None, **kwargs)
```

返回存储在 Azure File Share 中的目录和文件列表。

参数：

*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `kwargs(object)` - 传给`FileService.list_directories_and_files()`的可选关键字参数。

返回：文件和目录列表

返回类型：list

```py
load_file(file_path, share_name, directory_name, file_name, **kwargs)
```

将文件上传到 Azure File Share。

参数：

*   `file_path(str)` - 要上传的文件路径。
*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `kwargs(object)` - 传给`FileService.create_file_from_path()`的可选关键字参数。


```py
load_stream(stream, share_name, directory_name, file_name, count, **kwargs)
```

将流上传到 Azure File Share。

参数：

*   `stream(类文件对象)` - 打开的文件/流作为文件内容上传。
*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `count(int)` - 流的大小（以字节为单位）
*   `kwargs(object)` - 传给`FileService.create_file_from_stream()`的可选关键字参数。


```py
load_string(string_data, share_name, directory_name, file_name, **kwargs)
```

将字符串上传到 Azure File Share。

参数：

*   `string_data(str)` - 要加载的字符串。
*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `kwargs(object)` - 传给`FileService.create_file_from_text()`的可选关键字参数。


### Logging

可以将 Airflow 配置为在 Azure Blob Storage 中读取和写入任务日志。 请参阅[将日志写入 Azure Blob Storage](zh/howto/write-logs.md#将日志写入azure-blob-storage) 。

### Azure Data Lake

AzureDataLakeHook 通过与 WebHDFS 兼容的 REST API 进行通信。 确保 Airflow 存在`azure_data_lake`类型的连接。 可以通过提供用户名（=客户端 ID），密码（=客户端密钥），额外字段可以提供租户和帐户名称来完成授权。
> （请参阅链接`azure_data_lake_default`示例）。

* [AzureDataLakeHook](#AzureDataLakeHook)：与 Azure Data Lake 的接口。

#### AzureDataLakeHook

```py
class airflow.contrib.hooks.azure_data_lake_hook.AzureDataLakeHook(azure_data_lake_conn_id='azure_data_lake_default')
```

基类： `airflow.hooks.base_hook.BaseHook`

与 Azure Data Lake 进行交互。

客户端 ID 和客户端密钥应该在用户和密码参数中。 租户和帐户名称应为额外字段，如 {"tenant": "<TENANT>", "account_name": "ACCOUNT_NAME"}

参数：`azure_data_lake_conn_id(str)` - 对 Azure Data Lake 连接的引用。

```py
check_for_file(file_path)
```

检查 Azure Data Lake 上是否存在文件。

参数：`file_path(str)` - 文件的路径和名称。

返回：如果文件存在则为 True，否则为 False。

返回类型：bool

```py
download_file(local_path, remote_path, nthreads=64, overwrite=True, buffersize=4194304, blocksize=4194304)
```

从 Azure Blob Storage 下载文件。

参数：

*   `local_path(str)` - 本地路径。 如果下载单个文件，将写入此文件，如果它是现有目录，将在其中创建文件。 如果下载多个文件，这是要写入的根目录。将根据需要创建目录。
*   `remote_path(str)` - 用于查找远程文件的远程路径，可以使用通配符。 不支持使用`**`的递归 glob 模式。
*   `nthreads(int)` - 要使用的线程数。 如果为 None，则使用核心数。
*   `overwrite(bool)` - 是否强制覆盖现有文件/目录。 如果 False 并且远程路径是目录，则无论是否覆盖任何文件都将退出。 如果为 True，则实际仅覆盖匹配的文件名。
*   `buffersize(int)` - 支持最大 2 ** 22 内部缓冲区的字节数。 block 不能大于 trunk。
*   `blocksize(int)` - 一个 block 支持最大 2 ** 22 字节数。 在每个 trunk 中，我们为每个 API 调用写一个较小的block。 这个 block 不能大于 trunk。


```py
get_conn()
```

返回 AzureDLFileSystem 对象。

```py
upload_file(local_path, remote_path, nthreads=64, overwrite=True, buffersize=4194304, blocksize=4194304)
```

将文件上传到 Azure Data Lake。

参数：

*   `local_path(str)` - 本地路径。 可以是单个文件，目录（在这种情况下，递归上传）或 glob 模式。 不支持使用`**`的递归 glob 模式。
*   `remote_path(str)` - 要上传的远程路径; 如果有多个文件，这就是要写入的根目录。
*   `nthreads(int)` - 要使用的线程数。 如果为 None，则使用核心数。
*   `overwrite(bool)` - 是否强制覆盖现有文件/目录。 如果 False 并且远程路径是目录，则无论是否覆盖任何文件都将退出。 如果为 True，则实际仅覆盖匹配的文件名。
*   `buffersize(int)` - 支持最大 2 ** 22 内部缓冲区的字节数。 block 不能大于 trunk。
*   `blocksize(int)` - 一个 block 支持最大 2 ** 22 字节数。 在每个 trunk 中，我们为每个 API 调用写一个较小的block。 这个 block 不能大于 trunk。


## AWS: Amazon Web Services

Airflow 大量支持 Amazon Web Services。 但请注意，Hook，Sensors 和 Operators 都在 contrib 部分。

### AWS EMR

*   [EmrAddStepsOperator](#EmrAddStepsOperator) ：向现有 EMR JobFlow 添加步骤。
*   [EmrCreateJobFlowOperator](#EmrCreateJobFlowOperator) ：创建 EMR JobFlow，从 EMR 连接读取配置。
*   [EmrTerminateJobFlowOperator](#EmrTerminateJobFlowOperator) ：终止 EMR JobFlow。
*   [EmrHook](#EmrHook) ：与 AWS EMR 互动。

#### EmrAddStepsOperator

```py
class airflow.contrib.operators.emr_add_steps_operator.EmrAddStepsOperator(job_flow_id, aws_conn_id='s3_default', steps=None, *args, **kwargs)
```

基类： `airflow.models.BaseOperator`

向现有 EMR job_flow 添加步骤的 operator。

参数：

*   `job_flow_id` - 要添加步骤的 JobFlow 的 ID。（模板渲染后）
*   `aws_conn_id(str)` - 与使用的 aws 连接
*   `steps(list)` - 要添加到作业流的 boto3 步骤。 （模板渲染后）


#### EmrCreateJobFlowOperator

```py
class airflow.contrib.operators.emr_create_job_flow_operator.EmrCreateJobFlowOperator(aws_conn_id='s3_default', emr_conn_id='emr_default', job_flow_overrides=None, *args, **kwargs)
```

基类： `airflow.models.BaseOperator`

创建 EMR JobFlow，从 EMR 连接读取配置。 可以向 JobFlow 传递参数以覆盖连接中的配置。

参数：

*   `aws_conn_id(str)` - 要使用的 aws 连接
*   `emr_conn_id(str)` - 要使用的 emr 连接
*   `job_flow_overrides` - 用于覆盖 emr_connection extra 的 boto3 式参数。 （模板渲染后）


#### EmrTerminateJobFlowOperator

```py
class airflow.contrib.operators.emr_terminate_job_flow_operator.EmrTerminateJobFlowOperator(job_flow_id, aws_conn_id='s3_default', *args, **kwargs)
```

基类： `airflow.models.BaseOperator`

终止 EMR JobFlows 的 operator。

参数：

*   `job_flow_id` - 要终止的 JobFlow 的 id。（模板渲染后）
*   `aws_conn_id(str)` - 要使用的 aws 连接


#### EmrHook

```py
class airflow.contrib.hooks.emr_hook.EmrHook(emr_conn_id=None, *args, **kwargs)
```

基类： `airflow.contrib.hooks.aws_hook.AwsHook`

与 AWS EMR 交互。 emr_conn_id 是使用 create_job_flow 方法唯一必需的。

```py
create_job_flow(job_flow_overrides)
```

使用 EMR 连接中的配置创建作业流。 json_flow_overrrides 是传给 run_job_flow 方法的参数。

### AWS S3

*   [S3Hook](#S3Hook) ：与 AWS S3 交互。
*   [S3FileTransformOperator](#S3FileTransformOperator) ：将数据从 S3 源位置复制到本地文件系统上的临时位置。
*   [S3ListOperator](#S3ListOperator) ：列出与 S3 位置的键前缀匹配的文件。
*   [S3ToGoogleCloudStorageOperator](#S3ToGoogleCloudStorageOperator) ：将 S3 位置与 Google 云端存储[分区](28)同步。
*   [S3ToHiveTransfer](#S3ToHiveTransfer) ：将数据从 S3 移动到 Hive。 operator 从 S3 下载文件，在将文件加载到 Hive 表之前将其存储在本地。

#### S3Hook

```py
class airflow.hooks.S3_hook.S3Hook(aws_conn_id='aws_default')
```

基类： `airflow.contrib.hooks.aws_hook.AwsHook`

使用 boto3 库与 AWS S3 交互。

```py
check_for_bucket(bucket_name)
```

检查 bucket_name 是否存在。

参数：`bucket_name(str)` - 存储桶的名称


```py
check_for_key(key, bucket_name=None)
```

检查存储桶中是否存在密钥

参数：

*   `key(str)` - 指向文件的 S3 的 key
*   `bucket_name(str)` - 存储桶的名称


```py
check_for_prefix(bucket_name, prefix, delimiter)
```

检查存储桶中是否存在前缀

```py
check_for_wildcard_key(wildcard_key, bucket_name=None, delimiter='')
```

检查桶中是否存在与通配符表达式匹配的密钥

```py
get_bucket(bucket_name)
```

返回 boto3.S3.Bucket 对象

参数：`bucket_name(str)` - 存储桶的名称


```py
get_key(key, bucket_name=None)
```

返回 boto3.s3.Object

参数：

*   `key(str)` - 密钥的路径
*   `bucket_name(str)` - 存储桶的名称


```py
get_wildcard_key(wildcard_key, bucket_name=None, delimiter='')
```

返回与通配符表达式匹配的 boto3.s3.Object 对象

参数：

*   `wildcard_key(str)` - 密钥的路径
*   `bucket_name(str)` - 存储桶的名称


```py
list_keys(bucket_name, prefix='', delimiter='', page_size=None, max_items=None)
```

列出前缀下的存储桶中的密钥，但不包含分隔符

参数：

*   `bucket_name(str)` - 存储桶的名称
*   `prefix(str)` - 一个密钥前缀
*   `delimiter(str)` - 分隔符标记键层次结构。
*   `page_size(int)` - 分页大小
*   `max_items(int)` - 要返回的最大项目数


```py
list_prefixes(bucket_name, prefix='', delimiter='', page_size=None, max_items=None)
```

列出前缀下的存储桶中的前缀

参数：

*   `bucket_name(str)` - 存储桶的名称
*   `prefix(str)` - 一个密钥前缀
*   `delimiter(str)` - 分隔符标记键层次结构。
*   `page_size(int)` - 分页大小
*   `max_items(int)` - 要返回的最大项目数


```py
load_bytes(bytes_data, key, bucket_name=None, replace=False, encrypt=False)
```

将字节加载到 S3

这是为了方便在 S3 中删除字符串。 它使用 boto 基础结构将文件发送到 s3。

参数：

*   `bytes_data(bytes)` - 设置为密钥内容的字节。
*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储桶的名称
*   `replace(bool)` - 一个标志，用于决定是否覆盖密钥（如果已存在）
*   `encrypt(bool)` - 如果为 True，则文件将在服务器端由 S3 加密，并在 S3 中静止时以加密形式存储。


```py
load_file(filename, key, bucket_name=None, replace=False, encrypt=False)
```

将本地文件加载到 S3

参数：

*   `filename(str)` - 要加载的文件的名称。
*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储桶的名称
*   `replace(bool)` - 一个标志，用于决定是否覆盖密钥（如果已存在）。 如果 replace 为 False 且密钥存在，则会引发错误。
*   `encrypt(bool)` - 如果为 True，则文件将在服务器端由 S3 加密，并在 S3 中静止时以加密形式存储。


```py
load_string(string_data, key, bucket_name=None, replace=False, encrypt=False, encoding='utf-8')
```

将字符串加载到 S3

这是为了方便在 S3 中删除字符串。 它使用 boto 基础结构将文件发送到 s3。

参数：

*   `string_data(str)` - 要设置为键的内容的字符串。
*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储桶的名称
*   `replace(bool)` - 一个标志，用于决定是否覆盖密钥（如果已存在）
*   `encrypt(bool)` - 如果为 True，则文件将在服务器端由 S3 加密，并在 S3 中静止时以加密形式存储。


```py
read_key(key, bucket_name=None)
```

从 S3 读取密钥

参数：

*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储桶的名称


```py
select_key(key, bucket_name=None, expression='SELECT * FROM S3Object', expression_type='SQL', input_serialization={'CSV': {}}, output_serialization={'CSV': {}})
```

使用 S3 Select 读取密钥。

参数：

*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储桶的名称
*   `expression(str)` - S3 选择表达式
*   `expression_type(str)` - S3 选择表达式类型
*   `input_serialization(dict)` - S3 选择输入数据序列化格式
*   `output_serialization(dict)` - S3 选择输出数据序列化格式

返回：通过 S3 Select 检索原始数据的子集

返回类型：str

也可以看看

有关 S3 Select 参数的更多详细信息： [http://boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.select_object_content](http://boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.select_object_content)

#### S3FileTransformOperator

```py
class airflow.operators.s3_file_transform_operator.S3FileTransformOperator(source_s3_key, dest_s3_key, transform_script=None, select_expression=None, source_aws_conn_id='aws_default', dest_aws_conn_id='aws_default', replace=False, *args, **kwargs)
```

基类： `airflow.models.BaseOperator`

将数据从源 S3 位置复制到本地文件系统上的临时位置。 用转换脚本对此文件运行转换，并将输出上传到目标 S3 位置。

本地文件系统中的源文件和目标文件的位置作为转换脚本的第一个和第二个参数提供。 转换脚本应该从源读取数据，转换它并将输出写入本地目标文件。 然后，operator 将本地目标文件上传到 S3。

S3 Select 也可用于过滤源内容。 如果指定了 S3 Select 表达式，则用户可以省略转换脚本。

参数：

*   `source_s3_key(str)` - 源 S3 的密钥。（模板渲染后）
*   `source_aws_conn_id(str)` - 源 s3 连接
*   `dest_s3_key(str)` - 目的 S3 的密钥。（模板渲染后）
*   `dest_aws_conn_id(str)` - 目标 s3 连接
*   `replace(bool)` - 替换 dest S3 密钥（如果已存在）
*   `transform_script(str)` - 可执行转换脚本的位置
*   `select_expression(str)` - S3 选择表达式


#### S3ListOperator

```py
class airflow.contrib.operators.s3listoperator.S3ListOperator(bucket, prefix='', delimiter='', aws_conn_id='aws_default', *args, **kwargs)
```

基类： `airflow.models.BaseOperator`

列出桶中具有给定前缀的所有对象。

此 operator 返回一个 python 列表，其中包含可由`xcom`在下游任务中使用的对象名称。

参数：

*   `bucket(str)` - S3 存储桶在哪里找到对象。（模板渲染后）
*   `prefix(str)` - 用于过滤名称以此字符串为前缀的对象。（模板渲染后）
*   `delimiter(str)` - 分隔符标记键层次结构。（模板渲染后）
*   `aws_conn_id(str)` - 连接到 S3 存储时使用的连接 ID。


以下 operator 将列出`data`存储区中 S3 `customers/2018/04/` key 的所有文件（不包括子文件夹）。

```py
 s3_file = S3ListOperator (
    task_id = 'list_3s_files' ,
    bucket = 'data' ,
    prefix = 'customers/2018/04/' ,
    delimiter = '/' ,
    aws_conn_id = 'aws_customers_conn'
)

```

#### S3ToGoogleCloudStorageOperator

```py
class airflow.contrib.operators.s3_to_gcs_operator.S3ToGoogleCloudStorageOperator(bucket, prefix='', delimiter='', aws_conn_id='aws_default', dest_gcs_conn_id=None, dest_gcs=None, delegate_to=None, replace=False, *args, **kwargs)
```

基类： `airflow.contrib.operators.s3listoperator.S3ListOperator`

将 S3 密钥（可能是前缀）与 Google 云端存储目标路径同步。

参数：

*   `bucket(str)` - S3 存储桶在哪里找到对象。（模板渲染后）
*   `prefix(str)` - 用于过滤名称以此字符串为前缀的对象。（模板渲染后）
*   `delimiter(str)` - 分隔符标记键层次结构。（模板渲染后）
*   `aws_conn_id(str)` - 源 S3 连接
*   `dest_gcs_conn_id(str)` - 连接到 Google 云端存储时要使用的目标连接 ID。
*   `dest_gcs(str)` - 要存储文件的目标 Google 云端存储**分区**和前缀。（模板渲染后）
*   `delegate_to(str)` - 代理的帐户（如果有）。 为此，发出请求的服务帐户必须启用域范围委派。
*   `replace(bool)` - 是否要替换现有目标文件。


例子
```py
s3_to_gcs_op = S3ToGoogleCloudStorageOperator(
task_id ='s3_to_gcs_example'，bucket ='my-s3-bucket'，prefix ='data / customers-201804'，dest_gcs_conn_id ='google_cloud_default'，dest_gcs ='gs：//my.gcs.bucket/some/customers/' ，replace = False，dag = my-dag）
```


请注意， `bucket` ， `prefix` ， `delimiter`和`dest_gcs`是模板化的，因此如果您愿意，可以在其中使用变量。

#### S3ToHiveTransfer

```py
class airflow.operators.s3_to_hive_operator.S3ToHiveTransfer(s3_key, field_dict, hive_table, delimiter=', ', create=True, recreate=False, partition=None, headers=False, check_headers=False, wildcard_match=False, aws_conn_id='aws_default', hive_cli_conn_id='hive_cli_default', input_compressed=False, tblproperties=None, select_expression=None, *args, **kwargs)
```

基类： `airflow.models.BaseOperator`

将数据从 S3 移动到 Hive。 operator 从 S3 下载文件，在将文件加载到 Hive 表之前将其存储在本地。 如果`create`或`recreate`参数设置为`True` ，则会生成`CREATE TABLE`和`DROP TABLE`语句。 Hive 数据类型是从游标的元数据中推断出来的。

请注意，Hive 中生成的表使用`STORED AS textfile` ，这不是最有效的序列化格式。 如果加载了大量数据和/或表格被大量查询，您可能只想使用此 operator 将数据暂存到临时表中，然后使用`HiveOperator`将其加载到最终目标表中。

参数：

*   `s3_key(str)` - 从 S3 检索的密钥。（模板渲染后）
*   `field_dict(dict)` - 字段的字典在文件中命名为键，其 Hive 类型为值
*   `hive_table(str)` - 目标 Hive 表，使用点表示法来定位特定数据库。（模板渲染后）
*   `create(bool)` - 是否创建表，如果它不存在
*   `recreate(bool)` - 是否在每次执行时删除并重新创建表
*   `partition(dict)` - 将目标分区作为分区列和值的字典。（模板渲染后）
*   `headers(bool)` - 文件是否包含第一行的列名
*   `check_headers(bool)` - 是否应该根据 field_dict 的键检查第一行的列名
*   `wildcard_match(bool)` - 是否应将 s3_key 解释为 Unix 通配符模式
*   `delimiter(str)` - 文件中的字段分隔符
*   `aws_conn_id(str)` - 源 s3 连接
*   `hive_cli_conn_id(str)` - 目标配置单元连接
*   `input_compressed(bool)` - 布尔值，用于确定是否需要文件解压缩来处理标头
*   `tblproperties(dict)` - 正在创建的 hive 表的 TBLPROPERTIES
*   `select_expression(str)` - S3 选择表达式


### AWS EC2 容器服务

*   [ECSOperator](#ECSOperator) ：在 AWS EC2 容器服务上执行任务。

#### ECSOperator

```py
class airflow.contrib.operators.ecs_operator.ECSOperator(task_definition, cluster, overrides, aws_conn_id=None, region_name=None, launch_type='EC2', **kwargs)
```

基类： `airflow.models.BaseOperator`

在 AWS EC2 Container Service 上执行任务

参数：

*   `task_definition(str)` - EC2 容器服务上的任务定义名称
*   `cluster(str)` - EC2 Container Service 上的集群名称
*   `aws_conn_id(str)` - AWS 的连接 ID。 如果为 None，将使用 boto3 凭证（ [http://boto3.readthedocs.io/en/latest/guide/configuration.html](http://boto3.readthedocs.io/en/latest/guide/configuration.html) ）。
*   `region_name` - 要在 AWS Hook 中使用的 region 名称。 覆盖连接中的 region_name（如果提供）
*   `launch_type` - 运行任务的启动类型（'EC2'或'FARGATE'）

参数：boto3 将接收的相同参数（模板化）： [http://boto3.readthedocs.org/en/latest/reference/services/ecs.html#ECS.Client.run_task](http://boto3.readthedocs.org/en/latest/reference/services/ecs.html#ECS.Client.run_task)

类型：dict

类型：launch_type：str

### AWS Batch Service

*   [AWSBatchOperator](#AWSBatchOperator) ：在 AWS Batch Service 上执行任务。

#### AWSBatchOperator

```py
class airflow.contrib.operators.awsbatch_operator.AWSBatchOperator(job_name, job_definition, job_queue, overrides, max_retries=4200, aws_conn_id=None, region_name=None, **kwargs)
```

基类： `airflow.models.BaseOperator`

在 AWS Batch Service 上执行作业

参数：

*   `job_name(str)` - 将在 AWS Batch 上运行的作业的名称
*   `job_definition(str)` - AWS Batch 上的作业定义名称
*   `job_queue(str)` - AWS Batch 上的队列名称
*   `max_retries(int)` - 服务器未合并时的指数退避重试，4200 = 48 小时
*   `aws_conn_id(str)` - AWS 的连接 ID。 如果为 None，将使用凭证 boto3 策略（ [http://boto3.readthedocs.io/en/latest/guide/configuration.html](http://boto3.readthedocs.io/en/latest/guide/configuration.html) ）。
*   **region_name** - 要在 AWS Hook 中使用的区域名称。 覆盖连接中的 region_name（如果提供）

参数：boto3 将在 containerOverrides 上接收的相同参数（模板化）： [http://boto3.readthedocs.io/en/latest/reference/services/batch.html#submit_job](http://boto3.readthedocs.io/en/latest/reference/services/batch.html) 

类型：dict

### AWS RedShift

*   [AwsRedshiftClusterSensor](#AwsRedshiftClusterSensor) ：等待 Redshift 集群达到特定状态。
*   [RedshiftHook](#RedshiftHook) ：使用 boto3 库与 AWS Redshift 交互。
*   [RedshiftToS3Transfer](#RedshiftToS3Transfer) ：对带有或不带标头的 CSV 执行卸载命令。
*   [S3ToRedshiftTransfer](#S3ToRedshiftTransfer) ：从 S3 执行复制命令为 CSV，带或不带标题。

#### AwsRedshiftClusterSensor

```py
class airflow.contrib.sensors.aws_redshift_cluster_sensor.AwsRedshiftClusterSensor(cluster_identifier, target_status='available', aws_conn_id='aws_default', *args, **kwargs)
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`]

等待 Redshift 集群达到特定状态。

参数：

*   `cluster_identifier(str)` - 要 ping 的集群的标识符。
*   `target_status(str)` - 所需的集群状态。


```py
poke(context)
```

Operator 在继承此类时应该覆盖以上函数。

#### RedshiftHook

```py
class airflow.contrib.hooks.redshift_hook.RedshiftHook(aws_conn_id='aws_default')
```

基类： [`airflow.contrib.hooks.aws_hook.AwsHook`]

使用 boto3 库与 AWS Redshift 交互

```py
cluster_status(cluster_identifier)
```

返回集群的状态

参数：`cluster_identifier(str)` - 集群的唯一标识符


```py
create_cluster_snapshot(snapshot_identifier, cluster_identifier)
```

创建集群的快照

参数：

*   `snapshot_identifier(str)` - 集群快照的唯一标识符
*   `cluster_identifier(str)` - 集群的唯一标识符


```py
delete_cluster(cluster_identifier, skip_final_cluster_snapshot=True, final_cluster_snapshot_identifier='')
```

删除集群并可选择创建快照

参数：

*   `cluster_identifier(str)` - 集群的唯一标识符
*   `skip_final_cluster_snapshot(bool)` - 确定集群快照创建
*   `final_cluster_snapshot_identifier(str)` - 最终集群快照的名称


```py
describe_cluster_snapshots(cluster_identifier)
```

获取集群的快照列表

参数：`cluster_identifier(str)` - 集群的唯一标识符


```py
restore_from_cluster_snapshot(cluster_identifier, snapshot_identifier)
```

从其快照还原集群

参数：

*   `cluster_identifier(str)` - 集群的唯一标识符
*   `snapshot_identifier(str)` - 集群快照的唯一标识符


#### RedshiftToS3Transfer

```py
class airflow.operators.redshift_to_s3_operator.RedshiftToS3Transfer(schema，table，s3_bucket，s3_key，redshift_conn_id ='redshift_default'，aws_conn_id ='aws_default'，unload_options =()，autocommit = False，parameters = None，include_header = False，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

执行 UNLOAD 命令，将 s3 作为带标题的 CSV

参数：

*   `schema(str)` - 对 redshift 数据库中特定模式的引用
*   `table(str)` - 对 redshift 数据库中特定表的引用
*   `s3_bucket(str)` - 对特定 S3 存储桶的引用
*   `s3_key(str)` - 对特定 S3 密钥的引用
*   `redshift_conn_id(str)` - 对特定 redshift 数据库的引用
*   `aws_conn_id(str)` - 对特定 S3 连接的引用
*   `unload_options(list)` - 对 UNLOAD 选项列表的引用


#### S3ToRedshiftTransfer

```py
class airflow.operators.s3_to_redshift_operator.S3ToRedshiftTransfer(schema，table，s3_bucket，s3_key，redshift_conn_id ='redshift_default'，aws_conn_id ='aws_default'，copy_options =()，autocommit = False，parameters = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

执行 COPY 命令将文件从 s3 加载到 Redshift

参数：

*   `schema(str)` - 对 redshift 数据库中特定模式的引用
*   `table(str)` - 对 redshift 数据库中特定表的引用
*   `s3_bucket(str)` - 对特定 S3 存储桶的引用
*   `s3_key(str)` - 对特定 S3 密钥的引用
*   `redshift_conn_id(str)` - 对特定 redshift 数据库的引用
*   `aws_conn_id(str)` - 对特定 S3 连接的引用
*   `copy_options(list)` - 对 COPY 选项列表的引用


## Databricks

[Databricks](https://databricks.com/)贡献了一个 Airflow operator，可以将运行提交到 Databricks 平台。在运营商内部与`api/2.0/jobs/runs/submit` [端点进行通信](https://docs.databricks.com/api/latest/jobs.html)。

### DatabricksSubmitRunOperator

```py
class airflow.contrib.operators.databricks_operator.DatabricksSubmitRunOperator(json = None，spark_jar_task = None，notebook_task = None，new_cluster = None，existing_cluster_id = None，libraries = None，run_name = None，timeout_seconds = None，databricks_conn_id ='databricks_default'，polling_period_seconds = 30，databricks_retry_limit = 3，do_xcom_push = False，**kwargs)
```

基类： `airflow.models.BaseOperator`

使用[api / 2.0 / jobs / runs / submit](https://docs.databricks.com/api/latest/jobs.html) API 端点向 Databricks 提交 Spark 作业运行。

有两种方法可以实例化此 operator。

在第一种方式，你可以把你通常用它来调用的 JSON 有效载荷`api/2.0/jobs/runs/submit`端点并将其直接传递到我们`DatabricksSubmitRunOperator`通过`json`参数。例如

```py
json = {
  'new_cluster' : {
    'spark_version' : '2.1.0-db3-scala2.11' ,
    'num_workers' : 2
  },
  'notebook_task' : {
    'notebook_path' : '/Users/airflow@example.com/PrepareData' ,
  },
}
notebook_run = DatabricksSubmitRunOperator ( task_id = 'notebook_run' , json = json )

```

另一种完成同样事情的方法是直接使用命名参数`DatabricksSubmitRunOperator`。请注意，`runs/submit`端点中的每个顶级参数都只有一个命名参数。在此方法中，您的代码如下所示：

```py
 new_cluster = {
  'spark_version' : '2.1.0-db3-scala2.11' ,
  'num_workers' : 2
}
notebook_task = {
  'notebook_path' : '/Users/airflow@example.com/PrepareData' ,
}
notebook_run = DatabricksSubmitRunOperator (
    task_id = 'notebook_run' ,
    new_cluster = new_cluster ,
    notebook_task = notebook_task )

```

在提供 json 参数**和**命名参数的情况下，它们将合并在一起。如果在合并期间存在冲突，则命名参数将优先并覆盖顶级`json`键。

```py
 目前 DatabricksSubmitRunOperator 支持的命名参数是
```

*   `spark_jar_task`
*   `notebook_task`
*   `new_cluster`
*   `existing_cluster_id`
*   `libraries`
*   `run_name`
*   `timeout_seconds`

参数：

*   `json(dict)` -

    包含 API 参数的 JSON 对象，将直接传递给`api/2.0/jobs/runs/submit`端点。其他命名参数（即`spark_jar_task`，`notebook_task`..）到该运营商将与此 JSON 字典合并如果提供他们。如果在合并期间存在冲突，则命名参数将优先并覆盖顶级 json 键。（模板渲染后）

    也可以看看

    有关模板的更多信息，请参阅[Jinja 模板](concepts.html)。[https://docs.databricks.com/api/latest/jobs.html#runs-submit](https://docs.databricks.com/api/latest/jobs.html)

*   `spark_jar_task(dict)` -

    JAR 任务的主要类和参数。请注意，实际的 JAR 在`libraries`。中指定。_ 无论是 _ `spark_jar_task` _ 或 _ `notebook_task`应符合规定。该字段将被模板化。

    也可以看看

    [https://docs.databricks.com/api/latest/jobs.html#jobssparkjartask](https://docs.databricks.com/api/latest/jobs.html)

*   `notebook_task(dict)` -

    笔记本任务的笔记本路径和参数。_ 无论是 _ `spark_jar_task` _ 或 _ `notebook_task`应符合规定。该字段将被模板化。

    也可以看看

    [https://docs.databricks.com/api/latest/jobs.html#jobsnotebooktask](https://docs.databricks.com/api/latest/jobs.html)

*   `new_cluster(dict)` -

    将在其上运行此任务的新集群的规范。_ 无论是 _ `new_cluster` _ 或 _ `existing_cluster_id`应符合规定。该字段将被模板化。

    也可以看看

    [https://docs.databricks.com/api/latest/jobs.html#jobsclusterspecnewcluster](https://docs.databricks.com/api/latest/jobs.html)

*   `existing_cluster_id(str)` - 要运行此任务的现有集群的 ID。_ 无论是 _ `new_cluster` _ 或 _ `existing_cluster_id`应符合规定。该字段将被模板化。
*   `libraries(list 或 dict)` -

    这个运行的库将使用。该字段将被模板化。

    也可以看看

    [https://docs.databricks.com/api/latest/libraries.html#managedlibrarieslibrary](https://docs.databricks.com/api/latest/libraries.html)

*   `run_name(str)` - 用于此任务的运行名称。默认情况下，这将设置为 Airflow `task_id`。这`task_id`是超类`BaseOperator`的必需参数。该字段将被模板化。
*   `timeout_seconds(int32)` - 此次运行的超时。默认情况下，使用值 0 表示没有超时。该字段将被模板化。
*   `databricks_conn_id(str)` - 要使用的 Airflow 连接的名称。默认情况下，在常见情况下，这将是`databricks_default`。要使用基于令牌的身份验证，请在连接的额外字段`token`中提供密钥。
*   `polling_period_seconds(int)` - 控制我们轮询此运行结果的速率。默认情况下，operator 每 30 秒轮询一次。
*   `databricks_retry_limit(int)` - 如果 Databricks 后端无法访问，则重试的次数。其值必须大于或等于 1。
*   `do_xcom_push(bool)` - 我们是否应该将 run_id 和 run_page_url 推送到 xcom。


## GCP：Google云端平台

Airflow 广泛支持 Google Cloud Platform。但请注意，大多数 Hooks 和 Operators 都在 contrib 部分。这意味着他们具有 _beta_ 状态，这意味着他们可以在次要版本之间进行重大更改。

请参阅[GCP 连接类型](howto/manage-connections.html)文档以配置与 GCP 的连接。

### 记录

可以将 Airflow 配置为在 Google 云端存储中读取和写入任务日志。请参阅[将日志写入 Google 云端存储](howto/write-logs.html)。

### BigQuery 的

#### BigQuery Operator

*   [BigQueryCheckOperator](#BigQueryCheckOperator)：对 SQL 查询执行检查，该查询将返回具有不同值的单行。
*   [BigQueryValueCheckOperator](#BigQueryValueCheckOperator)：使用 SQL 代码执行简单的值检查。
*   [BigQueryIntervalCheckOperator](#BigQueryIntervalCheckOperator)：检查作为 SQL 表达式给出的度量值是否在 days_back 之前的某个容差范围内。
*   [BigQueryCreateEmptyTableOperator](#BigQueryCreateEmptyTableOperator)：在指定的 BigQuery 数据集中创建一个新的空表，可选择使用模式。
*   [BigQueryCreateExternalTableOperator](#BigQueryCreateExternalTableOperator)：使用 Google Cloud Storage 中的数据在数据集中创建新的外部表。
*   [BigQueryDeleteDatasetOperator](#BigQueryDeleteDatasetOperator)：删除现有的 BigQuery 数据集。
*   [BigQueryOperator](#BigQueryOperator)：在特定的 BigQuery 数据库中执行 BigQuery SQL 查询。
*   [BigQueryToBigQueryOperator](#BigQueryToBigQueryOperator)：将 BigQuery 表复制到另一个 BigQuery 表。
*   [BigQueryToCloudStorageOperator](#BigQueryToCloudStorageOperator)：将 BigQuery 表传输到 Google Cloud Storage 存储桶

##### BigQueryCheckOperator

```py
class airflow.contrib.operators.bigquery_check_operator.BigQueryCheckOperator(sql，bigquery_conn_id ='bigquery_default'，*args，**kwargs)
```

基类： [`airflow.operators.check_operator.CheckOperator`]

对 BigQuery 执行检查。该`BigQueryCheckOperator`预期的 SQL 查询将返回一行。使用 python `bool`强制转换第一行的每个值。如果任何值返回，`False`则检查失败并输出错误。

请注意，Python bool 强制转换如下`False`：

*   `False`
*   `0`
*   空字符串（`""`）
*   空列表（`[]`）
*   空字典或集（`{}`）

给定一个查询，`SELECT COUNT(*) FROM foo` 当且仅当`== 0`时会失败。您可以制作更复杂的查询，例如，可以检查表与上游源表的行数相同，或者今天的分区计数大于昨天的分区，或者一组指标是否更少 7 天平均值超过 3 个标准差。

此 Operator 可用作管道中的数据质量检查，并且根据您在 DAG 中的位置，您可以选择在关键路径停止，防止发布可疑数据，或者接收电子邮件报警而不阻止 DAG 的继续。

参数：

*   `sql(str)` - 要执行的 sql
*   `bigquery_conn_id(str)` - 对 BigQuery 数据库的引用


##### BigQueryValueCheckOperator

```py
class airflow.contrib.operators.bigquery_check_operator.BigQueryValueCheckOperator(sql，pass_value，tolerance = None，bigquery_conn_id ='bigquery_default'，*args，**kwargs）
```

基类： [`airflow.operators.check_operator.ValueCheckOperator`]

使用 sql 代码执行简单的值检查。

参数：`sql(str)` - 要执行的 sql


##### BigQueryIntervalCheckOperator

```py
class airflow.contrib.operators.bigquery_check_operator.BigQueryIntervalCheckOperator(table，metrics_thresholds，date_filter_column ='ds'，days_back = -7，bigquery_conn_id ='bigquery_default'，*args，**kwargs)
```

基类： [`airflow.operators.check_operator.IntervalCheckOperator`]

检查作为 SQL 表达式给出的度量值是否在 days_back 之前的某个容差范围内。

此方法构造一个类似的查询

```py
 SELECT { metrics_thresholddictkey } FROM { table }
    WHERE { date_filter_column } =< date >

```

参数：

*   `table(str)` - 表名
*   `days_back(int)` - ds 与我们要检查的 ds 之间的天数。默认为 7 天
*   `metrics_threshold(dict)` - 由指标索引的比率字典，例如'COUNT(*)'：1.5 将需要当前日和之前的 days_back 之间 50％或更小的差异。


##### BigQueryGetDataOperator

```py
class airflow.contrib.operators.bigquery_get_data.BigQueryGetDataOperator(dataset_id，table_id，max_results ='100'，selected_fields = None，bigquery_conn_id ='bigquery_default'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

从 BigQuery 表中获取数据（或者为所选列获取数据）并在 python 列表中返回数据。返回列表中的元素数将等于获取的行数。列表中的每个元素将是一个列表，其中元素将表示该行的列值。

**结果示例**：`[['Tony', '10'], ['Mike', '20'], ['Steve', '15']]`

注意

如果传递的字段`selected_fields`的顺序与 BQ 表中已有的列的顺序不同，则数据仍将按 BQ 表的顺序排列。例如，如果 BQ 表有 3 列，`[A,B,C]`并且您传递'B, A'，`selected_fields` 仍然是表格'A,B'。

**示例** ：

```py
 get_data = BigQueryGetDataOperator (
    task_id = 'get_data_from_bq' ,
    dataset_id = 'test_dataset' ,
    table_id = 'Transaction_partitions' ,
    max_results = '100' ,
    selected_fields = 'DATE' ,
    bigquery_conn_id = 'airflow-service-account'
)

```

参数：

*   `dataset_id` - 请求的表的数据集 ID。（模板渲染后）
*   `table_id(str)` - 请求表的表 ID。（模板渲染后）
*   `max_results(str)` - 从表中获取的最大记录数（行数）。（模板渲染后）
*   `selected_fields(str)` - 要返回的字段列表（逗号分隔）。如果未指定，则返回所有字段。
*   `bigquery_conn_id(str)` - 对特定 BigQuery 的引用。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


##### BigQueryCreateEmptyTableOperator

```py
class airflow.contrib.operators.bigquery_operator.BigQueryCreateEmptyTableOperator(dataset_id，table_id，project_id = None，schema_fields = None，gcs_schema_object = None，time_partitioning = {}，bigquery_conn_id ='bigquery_default'，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，*args ，**kwargs)
```

基类： `airflow.models.BaseOperator`

在指定的 BigQuery 数据集中创建一个新的空表，可选择使用模式。

可以用两种方法之一指定用于 BigQuery 表的模式。您可以直接传递架构字段，也可以将运营商指向 Google 云存储对象名称。Google 云存储中的对象必须是包含架构字段的 JSON 文件。您还可以创建没有架构的表。

参数：

*   `project_id(str)` - 将表创建的项目。（模板渲染后）
*   `dataset_id(str)` - 用于创建表的数据集。（模板渲染后）
*   `table_id(str)` - 要创建的表的名称。（模板渲染后）
*   `schema_fields(list)` -

    如果设置，则此处定义的架构字段列表：[https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs#configuration.load.schema](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs#configuration.load.schema)

    **示例** ：

    ```py
     schema_fields = [{ "name" : "emp_name" , "type" : "STRING" , "mode" : "REQUIRED" },
                   { "name" : "salary" , "type" : "INTEGER" , "mode" : "NULLABLE" }]

    ```

*   `gcs_schema_object(str)` - 包含模式（模板化）的 JSON 文件的完整路径。例如：`gs://test-bucket/dir1/dir2/employee_schema.json`
*   `time_partitioning(dict)` -

    配置可选的时间分区字段，即按 API 规范按字段，类型和到期分区。

    也可以看看

    [https://cloud.google.com/bigquery/docs/reference/rest/v2/tables#timePartitioning](https://cloud.google.com/bigquery/docs/reference/rest/v2/tables#timePartitioning)

*   `bigquery_conn_id(str)` - 对特定 BigQuery 的引用。
*   `google_cloud_storage_conn_id(str)` - 对特定 Google 云存储的引用。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


**示例（在 GCS 中使用 JSON schema）**：

```py
 CreateTable = BigQueryCreateEmptyTableOperator (
    task_id = 'BigQueryCreateEmptyTableOperator_task' ,
    dataset_id = 'ODS' ,
    table_id = 'Employees' ,
    project_id = 'internal-gcp-project' ,
    gcs_schema_object = 'gs://schema-bucket/employee_schema.json' ,
    bigquery_conn_id = 'airflow-service-account' ,
    google_cloud_storage_conn_id = 'airflow-service-account'
)

```

**对应的 Schema 文件**（`employee_schema.json`）：

```py
 [
  {
    "mode" : "NULLABLE" ,
    "name" : "emp_name" ,
    "type" : "STRING"
  },
  {
    "mode" : "REQUIRED" ,
    "name" : "salary" ,
    "type" : "INTEGER"
  }
]

```

**示例（在 DAG 中使用 schema）**：

```py
 CreateTable = BigQueryCreateEmptyTableOperator (
    task_id = 'BigQueryCreateEmptyTableOperator_task' ,
    dataset_id = 'ODS' ,
    table_id = 'Employees' ,
    project_id = 'internal-gcp-project' ,
    schema_fields = [{ "name" : "emp_name" , "type" : "STRING" , "mode" : "REQUIRED" },
                   { "name" : "salary" , "type" : "INTEGER" , "mode" : "NULLABLE" }],
    bigquery_conn_id = 'airflow-service-account' ,
    google_cloud_storage_conn_id = 'airflow-service-account'
)

```

##### BigQueryCreateExternalTableOperator

```py
class airflow.contrib.operators.bigquery_operator.BigQueryCreateExternalTableOperator(bucket，source_objects，destination_project_dataset_table，schema_fields = None，schema_object = None，source_format ='CSV'，compression ='NONE'，skip_leading_rows = 0，field_delimiter ='，'，max_bad_records = 0 ，quote_character = None，allow_quoted_newlines = False，allow_jagged_rows = False，bigquery_conn_id ='bigquery_default'，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，src_fmt_configs = {}，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

使用 Google 云端存储中的数据在数据集中创建新的外部表。

可以用两种方法之一指定用于 BigQuery 表的模式。您可以直接传递架构字段，也可以将运营商指向 Google 云存储对象名称。Google 云存储中的对象必须是包含架构字段的 JSON 文件。

参数：

*   `bucket(str)` - 指向外部表的存储桶。（模板渲染后）
*   **source_objects** - 指向表格的 Google 云存储 URI 列表。（模板化）如果 source_format 是'DATASTORE_BACKUP'，则列表必须只包含一个 URI。
*   `destination_project_dataset_table(str)` - 用于将数据加载到（模板化）的表\1\<project>。）<dataset>。<table> BigQuery 表。如果未包\1\<project>，则项目将是连接 json 中定义的项目。
*   `schema_fields(list)` -

    如果设置，则此处定义的架构字段列表：[https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs#configuration.load.schema](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs#configuration.load.schema)

    **示例** ：

    ```py
     schema_fields = [{ "name" : "emp_name" , "type" : "STRING" , "mode" : "REQUIRED" },
                   { "name" : "salary" , "type" : "INTEGER" , "mode" : "NULLABLE" }]

    ```

    当 source_format 为'DATASTORE_BACKUP'时，不应设置。

*   **schema_object** - 如果设置，则指向包含表的架构的.json 文件的 GCS 对象路径。（模板渲染后）
*   **schema_object** - 字符串
*   `source_format(str)` - 数据的文件格式。
*   `compression(str)` - [可选]数据源的压缩类型。可能的值包括 GZIP 和 NONE。默认值为 NONE。Google Cloud Bigtable，Google Cloud Datastore 备份和 Avro 格式会忽略此设置。
*   `skip_leading_rows(int)` - 从 CSV 加载时要跳过的行数。
*   `field_delimiter(str)` - 用于 CSV 的分隔符。
*   `max_bad_records(int)` - BigQuery 在运行作业时可以忽略的最大错误记录数。
*   `quote_character(str)` - 用于引用 CSV 文件中数据部分的值。
*   `allow_quoted_newlines(bool)` - 是否允许引用的换行符（true）或不允许（false）。
*   `allow_jagged_rows(bool)` - 接受缺少尾随可选列的行。缺失值被视为空值。如果为 false，则缺少尾随列的记录将被视为错误记录，如果错误记录太多，则会在作业结果中返回无效错误。仅适用于 CSV，忽略其他格式。
*   `bigquery_conn_id(str)` - 对特定 BigQuery 挂钩的引用。
*   `google_cloud_storage_conn_id(str)` - 对特定 Google 云存储挂钩的引用。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `src_fmt_configs(dict)` - 配置特定于源格式的可选字段


##### BigQueryDeleteDatasetOperator

##### BigQueryOperator

```py
class airflow.contrib.operators.bigquery_operator.BigQueryOperator(bql = None，sql = None，destination_dataset_table = False，write_disposition ='WRITE_EMPTY'，allow_large_results = False，flatten_results = False，bigquery_conn_id ='bigquery_default'，delegate_to = None，udf_config = False ，use_legacy_sql = True，maximum_billing_tier = None，maximumbytesbilled = None，create_disposition ='CREATE_IF_NEEDED'，schema_update_options =()，query_params = None，priority ='INTERACTIVE'，time_partitioning = {}，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

在特定的 BigQuery 数据库中执行 BigQuery SQL 查询

参数：

*   `BQL(可接收表示 SQL 语句中的海峡，海峡列表（SQL 语句），或参照模板文件模板引用在“.SQL”结束海峡认可。)` - （不推荐使用。`SQL`参数代替）要执行的 sql 代码（模板化）
*   `SQL(可接收表示 SQL 语句中的海峡，海峡列表（SQL 语句），或参照模板文件模板引用在“.SQL”结束海峡认可。)` - SQL 代码被执行（模板渲染后）
*   `destination_dataset_table(str)` - 目的数据集表（\<project>|\<project>:)\<dataset>.\<table>，如果设置，将存储查询结果。（模板渲染后）
*   `write_disposition(str)` - 指定目标表已存在时发生的操作。（默认：'WRITE_EMPTY'）
*   `create_disposition(str)` - 指定是否允许作业创建新表。（默认值：'CREATE_IF_NEEDED'）
*   `allow_large_results(bool)` - 是否允许大结果。
*   `flatten_results(bool)` - 如果为 true 且查询使用旧版 SQL 方言，则展平查询结果中的所有嵌套和重复字段。`allow_large_results`必须是`true`如果设置为`false`。对于标准 SQL 查询，将忽略此标志，并且结果永远不会展平。
*   `bigquery_conn_id(str)` - 对特定 BigQuery 钩子的引用。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `udf_config(list)` - 查询的用户定义函数配置。有关详细信息，请参阅[https://cloud.google.com/bigquery/user-defined-functions](https://cloud.google.com/bigquery/user-defined-functions)。
*   `use_legacy_sql(bool)` - 是使用旧 SQL（true）还是标准 SQL（false）。
*   `maximum_billing_tier(int)` - 用作基本价格乘数的正整数。默认为 None，在这种情况下，它使用项目中设置的值。
*   `maximumbytesbilled(float)` - 限制为此作业计费的字节数。超出此限制的字节数的查询将失败（不会产生费用）。如果未指定，则将其设置为项目默认值。
*   `schema_update_options(tuple)` - 允许更新目标表的模式作为加载作业的副作用。
*   `query_params(dict)` - 包含查询参数类型和值的字典，传递给 BigQuery。
*   `priority(str)` - 指定查询的优先级。可能的值包括 INTERACTIVE 和 BATCH。默认值为 INTERACTIVE。
*   `time_partitioning(dict)` - 配置可选的时间分区字段，即按 API 规范按字段，类型和到期分区。请注意，'field'不能与 dataset.table $ partition 一起使用。


##### BigQueryTableDeleteOperator

```py
class airflow.contrib.operators.bigquery_table_delete_operator.BigQueryTableDeleteOperator(deletion_dataset_table，bigquery_conn_id ='bigquery_default'，delegate_to = None，ignore_if_missing = False，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

删除 BigQuery 表

参数：

*   `deletion_dataset_table(str)` - 删除的数据集表（\<project>|\<project>:)\<dataset>.\<table>，指示将删除哪个表。（模板渲染后）
*   `bigquery_conn_id(str)` - 对特定 BigQuery 钩子的引用。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `ignore_if_missing(bool)` - 如果为 True，则即使请求的表不存在也返回成功。


##### BigQueryToBigQueryOperator

```py
class airflow.contrib.operators.bigquery_to_bigquery.BigQueryToBigQueryOperator(source_project_dataset_tables，destination_project_dataset_table，write_disposition ='WRITE_EMPTY'，create_disposition ='CREATE_IF_NEEDED'，bigquery_conn_id ='bigquery_default'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

将数据从一个 BigQuery 表复制到另一个。

也可以看看

有关这些参数的详细信息，请访问：[https://cloud.google.com/bigquery/docs/reference/v2/jobs#configuration.copy](https://cloud.google.com/bigquery/docs/reference/v2/jobs#configuration.copy)

参数：

*   `source_project_dataset_tables(list|str)` - 一个或多个点(project:|project.)\<dataset>.\<table>用作源数据的 BigQuery 表。如果未包含\<project>，则 project 应当定义在连接 json 中。如果有多个源表，请使用列表。（模板渲染后）
*   `destination_project_dataset_table(str)` - 目标 BigQuery 表。格式为：（`project:`|`project`）<dataset>.<table>（模板化）
*   `write_disposition(str)` - 表已存在时的处理。
*   `create_disposition(str)` - 如果表不存在，则创建处理。
*   `bigquery_conn_id(str)` - 对特定 BigQuery 的引用。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


##### BigQueryToCloudStorageOperator

```py
class airflow.contrib.operators.bigquery_to_gcs.BigQueryToCloudStorageOperator(source_project_dataset_table，destination_cloud_storage_uris，compression ='NONE'，export_format ='CSV'，field_delimiter =','，print_header = True，bigquery_conn_id ='bigquery_default'，delegate_to = None，*args， **kwargs)
```

基类： `airflow.models.BaseOperator`

将 BigQuery 表传输到 Google Cloud Storage 存储桶。

也可以看看

有关这些参数的详细信息，请访问：[https://cloud.google.com/bigquery/docs/reference/v2/jobs](https://cloud.google.com/bigquery/docs/reference/v2/jobs)

参数：

*   `source_project_dataset_table(str)` - 用作源数据的(\<project>.|<project>:)\<dataset>.\<table> BigQuery 表。如果未包含\<project>，则 project 应当定义在连接 json 中。（模板渲染后）
*   `destination_cloud_storage_uris(list)` - 目标 Google 云端存储 URI（例如 gs://some-bucket/some-file.txt）。（模板化）遵循此处定义的惯例：https：//cloud.google.com/bigquery/exporting-data-from-bigquery#exportingmultiple
*   `compression(str)` - 要使用的压缩类型。
*   `export_format` - 要导出的文件格式。
*   `field_delimiter(str)` - 提取到 CSV 时使用的分隔符。
*   `print_header(bool)` - 是否打印 CSV 文件头。
*   `bigquery_conn_id(str)` - 对特定 BigQuery 的引用。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

#### BigQueryHook

```py
class airflow.contrib.hooks.bigquery_hook.BigQueryHook(bigquery_conn_id ='bigquery_default'，delegate_to = None，use_legacy_sql = True)
```

基类：[`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`]，[`airflow.hooks.dbapi_hook.DbApiHook`]，`airflow.utils.log.logging_mixin.LoggingMixin`

与 BigQuery 交互。此挂钩使用 Google Cloud Platform 连接。

```py
get_conn()
```

返回 BigQuery PEP 249 连接对象。

```py
get_pandas_df(sql，parameters = None，dialect = None)
```

返回 BigQuery 查询生成的结果的 Pandas DataFrame。必须重写 DbApiHook 方法，因为 Pandas 不支持 PEP 249 连接，但 SQLite 除外。参看：

[https://github.com/pydata/pandas/blob/master/pandas/io/sql.py#L447 ](https://github.com/pydata/pandas/blob/master/pandas/io/sql.py#L447) [https://github.com/pydata/pandas/issues/6900](https://github.com/pydata/pandas/issues/6900)

参数：

*   `sql(str)` - 要执行的 BigQuery SQL。
*   `parameters(map 或 iterable)` - 用于呈现 SQL 查询的参数（未使用，请保留覆盖超类方法）
*   `dialect({'legacy', 'standard'})` - BigQuery SQL 的方言 - 遗留 SQL 或标准 SQL 默认使用`self.use_legacy_sql（`如果未指定）

```py
get_service()
```

返回一个 BigQuery 服务对象。

```py
insert_rows(table，rows，target_fields = None，commit_every = 1000)
```

目前不支持插入。从理论上讲，您可以使用 BigQuery 的流 API 将行插入表中，但这尚未实现。

```py
table_exists(project_id，dataset_id，table_id)
```

检查 Google BigQuery 中是否存在表格。

参数：

*   `project_id(str)` - 要在其中查找表的 Google 云项目。连接必须有对指定项目的访问权限。
*   `dataset_id(str)` - 要在其中查找表的数据集的名称。
*   `table_id(str)` - 要检查的表的名称。


### 云 DataFlow

#### DataFlow Operator

*   [DataFlowJavaOperator](#DataFlowJavaOperator)：启动用 Java 编写的 Cloud Dataflow 作业。
*   [DataflowTemplateOperator](#DataflowTemplateOperator)：启动模板化的 Cloud DataFlow 批处理作业。
*   [DataFlowPythonOperator](#DataFlowPythonOperator)：启动用 python 编写的 Cloud Dataflow 作业。

##### DataFlowJavaOperator

```py
class airflow.contrib.operators.dataflow_operator.DataFlowJavaOperator(jar，dataflow_default_options = None，options = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，poll_sleep = 10，job_class = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

启动 Java Cloud DataFlow 批处理作业。操作的参数将传递给作业。

在 dag 的 default_args 中定义 dataflow_*参数是一个很好的做法，例如项目，区域和分段位置。

```py
 default_args = {
    'dataflow_default_options' : {
        'project' : 'my-gcp-project' ,
        'zone' : 'europe-west1-d' ,
        'stagingLocation' : 'gs://my-staging-bucket/staging/'
    }
}

```

您需要使用`jar`参数将路径作为文件引用传递给数据流，jar 需要是一个自动执行的 jar（请参阅以下文档：[https://beam.apache.org/documentation/runners/dataflow/](https://beam.apache.org/documentation/runners/dataflow/)。使用`options`传递你的参数。

```py
 t1 = DataFlowOperation (
    task_id = 'datapflow_example' ,
    jar = '{{var.value.gcp_dataflow_base}}pipeline/build/libs/pipeline-example-1.0.jar' ,
    options = {
        'autoscalingAlgorithm' : 'BASIC' ,
        'maxNumWorkers' : '50' ,
        'start' : '{{ds}}' ,
        'partitionType' : 'DAY' ,
        'labels' : { 'foo' : 'bar' }
    },
    gcp_conn_id = 'gcp-airflow-service-account' ,
    dag = my - dag )

```

这两个`jar`和`options`模板化，所以你可以在其中使用变量。

```py
 default_args = {
    'owner' : 'airflow' ,
    'depends_on_past' : False ,
    'start_date' :
        ( 2016 , 8 , 1 ),
    'email' : [ 'alex@vanboxel.be' ],
    'email_on_failure' : False ,
    'email_on_retry' : False ,
    'retries' : 1 ,
    'retry_delay' : timedelta ( minutes = 30 ),
    'dataflow_default_options' : {
        'project' : 'my-gcp-project' ,
        'zone' : 'us-central1-f' ,
        'stagingLocation' : 'gs://bucket/tmp/dataflow/staging/' ,
    }
}

dag = DAG ( 'test-dag' , default_args = default_args )

task = DataFlowJavaOperator (
    gcp_conn_id = 'gcp_default' ,
    task_id = 'normalize-cal' ,
    jar = '{{var.value.gcp_dataflow_base}}pipeline-ingress-cal-normalize-1.0.jar' ,
    options = {
        'autoscalingAlgorithm' : 'BASIC' ,
        'maxNumWorkers' : '50' ,
        'start' : '{{ds}}' ,
        'partitionType' : 'DAY'

    },
    dag = dag )

```

##### DataflowTemplateOperator

```py
class airflow.contrib.operators.dataflow_operator.DataflowTemplateOperator(template，dataflow_default_options = None，parameters = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，poll_sleep = 10，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

启动模板化云 DataFlow 批处理作业。操作的参数将传递给作业。在 dag 的 default_args 中定义 dataflow_ *参数是一个很好的做法，例如项目，区域和分段位置。

也可以看看

[https://cloud.google.com/dataflow/docs/reference/rest/v1b3/LaunchTemplateParameters ](https://cloud.google.com/dataflow/docs/reference/rest/v1b3/LaunchTemplateParameters)[https://cloud.google.com/dataflow/docs/reference/rest/v1b3/RuntimeEnvironment](https://cloud.google.com/dataflow/docs/reference/rest/v1b3/RuntimeEnvironment)

```py
 default_args = {
    'dataflow_default_options' : {
        'project' : 'my-gcp-project'
        'zone' : 'europe-west1-d' ,
        'tempLocation' : 'gs://my-staging-bucket/staging/'
        }
    }
}

```

您需要将路径作为带`template`参数的文件引用传递给数据流模板。使用`parameters`来传递参数给你的工作。使用`environment`对运行环境变量传递给你的工作。

```py
 t1 = DataflowTemplateOperator (
    task_id = 'datapflow_example' ,
    template = '{{var.value.gcp_dataflow_base}}' ,
    parameters = {
        'inputFile' : "gs://bucket/input/my_input.txt" ,
        'outputFile' : "gs://bucket/output/my_output.txt"
    },
    gcp_conn_id = 'gcp-airflow-service-account' ,
    dag = my - dag )

```

`template`，`dataflow_default_options`并且`parameters`是模板化的，因此您可以在其中使用变量。

##### DataFlowPythonOperator

```py
class airflow.contrib.operators.dataflow_operator.DataFlowPythonOperator(py_file，py_options = None，dataflow_default_options = None，options = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，poll_sleep = 10，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

```py
execute(context)
```

执行 python 数据流作业。

#### DataFlowHook

```py
class airflow.contrib.hooks.gcp_dataflow_hook.DataFlowHook(gcp_conn_id ='google_cloud_default'，delegate_to = None，poll_sleep = 10)
```

基类： [`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`]

```py
get_conn()
```

返回 Google 云端存储服务对象。

### Cloud DataProc

#### DataProc Operator

*   [DataprocClusterCreateOperator](#DataprocClusterCreateOperator)：在 Google Cloud Dataproc 上创建新集群。
*   [DataprocClusterDeleteOperator](#DataprocClusterDeleteOperator)：删除 Google Cloud Dataproc 上的集群。
*   [DataprocClusterScaleOperator](#DataprocClusterScaleOperator)：在 Google Cloud Dataproc 上向上或向下扩展集群。
*   [DataProcPigOperator](#DataProcPigOperator)：在 Cloud DataProc 集群上启动 Pig 查询作业。
*   [DataProcHiveOperator](#DataProcHiveOperator)：在 Cloud DataProc 集群上启动 Hive 查询作业。
*   [DataProcSparkSqlOperator](#DataProcSparkSqlOperator)：在 Cloud DataProc 集群上启动 Spark SQL 查询作业。
*   [DataProcSparkOperator](#DataProcSparkOperator)：在 Cloud DataProc 集群上启动 Spark 作业。
*   [DataProcHadoopOperator](#DataProcHadoopOperator)：在 Cloud DataProc 集群上启动 Hadoop 作业。
*   [DataProcPySparkOperator](#DataProcPySparkOperator)：在 Cloud DataProc 集群上启动 PySpark 作业。
*   [DataprocWorkflowTemplateInstantiateOperator](#DataprocWorkflowTemplateInstantiateOperator)：在 Google Cloud Dataproc 上实例化 WorkflowTemplate。
*   [DataprocWorkflowTemplateInstantiateInlineOperator](#DataprocWorkflowTemplateInstantiateInlineOperator)：在 Google Cloud Dataproc 上实例化 WorkflowTemplate 内联。

##### DataprocClusterCreateOperator

```py
class airflow.contrib.operators.dataproc_operator.DataprocClusterCreateOperator(cluster_name，project_id，num_workers，zone，network_uri = None，subnetwork_uri = None，internal_ip_only = None，tags = None，storage_bucket = None，init_actions_uris = None，init_action_timeout ='10m'，metadata = None，image_version = None，属性= None，master_machine_type ='n1-standard-4'，master_disk_size = 500，worker_machine_type ='n1-standard-4'，worker_disk_size = 500，num_preemptible_workers = 0，labels = None，region =' global'，gcp_conn_id ='google_cloud_default'，delegate_to = None，service_account = None，service_account_scopes = None，idle_delete_ttl = None，auto_delete_time = None，auto_delete_ttl = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

在 Google Cloud Dataproc 上创建新集群。operator 将等待创建成功或创建过程中发生错误。

参数允许配置集群。请参阅

[https://cloud.google.com/dataproc/docs/reference/rest/v1/projects.regions.clusters](https://cloud.google.com/dataproc/docs/reference/rest/v1/projects.regions.clusters)

有关不同参数的详细说明。链接中详述的大多数配置参数都可作为此 operator 的参数。

参数：

*   `cluster_name(str)` - 要创建的 DataProc 集群的名称。（模板渲染后）
*   `project_id(str)` - 用于创建集群的 Google 云项目的 ID。（模板渲染后）
*   `num_workers(int)` - 工人数量
*   `storage_bucket(str)` - 要使用的存储桶，设置为 None 允许 dataproc 为您生成自定义存储桶
*   `init_actions_uris(list[str])` - 包含数据空间初始化脚本的 GCS uri 列表
*   `init_action_timeout(str)` - init_actions_uris 中可执行脚本限定的完成时间
*   `metadata(dict)` - 要添加到所有实例的键值 google 计算引擎元数据条目的字典
*   `image_version(str)` - Dataproc 集群内的软件版本
*   `properties(dict)` - 配置的属性字典（如spark-defaults.conf），见[https://cloud.google.com/dataproc/docs/reference/rest/v1/projects.regions.clusters#SoftwareConfig](https://cloud.google.com/dataproc/docs/reference/rest/v1/projects.regions.clusters#SoftwareConfig)
*   `master_machine_type(str)` - 计算要用于主节点的引擎机器类型
*   `master_disk_size(int)` - 主节点的磁盘大小
*   `worker_machine_type(str)` - 计算要用于工作节点的引擎计算机类型
*   `worker_disk_size(int)` - 工作节点的磁盘大小
*   `num_preemptible_workers(int)` - 可抢占的工作节点数
*   `labels(dict)` - 要添加到集群的标签的字典
*   `zone(str)` - 集群所在的区域。（模板渲染后）
*   `network_uri(str)` - 用于机器通信的网络 uri，不能用 subnetwork_uri 指定
*   `subnetwork_uri(str)` - 无法使用 network_uri 指定要用于机器通信的子网 uri
*   `internal_ip_only(bool)` - 如果为 true，则集群中的所有实例将只具有内部 IP 地址。这只能为启用子网的网络启用
*   `tags(list[str])` - 要添加到所有实例的 GCE 标记
*   `region` - 默认为'global'，可能在未来变得相关。（模板渲染后）
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `service_account(str)` - dataproc 实例的服务帐户。
*   `service_account_scopes(list[str])` - 要包含的服务帐户范围的 URI。
*   `idle_delete_ttl(int)` - 集群在保持空闲状态时保持活动状态的最长持续时间。通过此阈值将导致集群被自动删除。持续时间（秒）。
*   `auto_delete_time(datetime.datetime)` - 自动删除集群的时间。
*   `auto_delete_ttl(int)` - 集群的生命周期，集群将在此持续时间结束时自动删除，持续时间（秒）。（如果设置了 auto_delete_time，则将忽略此参数）


##### DataprocClusterScaleOperator

```py
class airflow.contrib.operators.dataproc_operator.DataprocClusterScaleOperator(cluster_name，project_id，region ='global'，gcp_conn_id ='google_cloud_default'，delegate_to = None，num_workers = 2，num_preemptible_workers = 0，graceful_decommission_timeout = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

在 Google Cloud Dataproc 上进行扩展，向上或向下扩展。operator 将等待，直到重新调整集群。

**示例** ：

```py
 t1 = DataprocClusterScaleOperator（
task_id ='dataproc_scale'，project_id ='my-project'，cluster_name ='cluster-1'，num_workers = 10，num_preemptible_workers = 10，graceful_decommission_timeout ='1h'dag = dag）
```

也可以看看

有关扩展集群的更多详细信息，请参阅以下参考：[https://cloud.google.com/dataproc/docs/concepts/configuring-clusters/scaling-clusters](https://cloud.google.com/dataproc/docs/concepts/configuring-clusters/scaling-clusters)

参数：

*   `cluster_name(str)` - 要扩展的集群的名称。（模板渲染后）
*   `project_id(str)` - 集群运行的 Google 云项目的 ID。（模板渲染后）
*   `region(str)` - 数据通路簇的区域。（模板渲染后）
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `num_workers(int)` - 工人数量
*   `num_preemptible_workers(int)` - 新的可抢占工人数量
*   `graceful_decommission_timeout(str)` - 优雅的 YARN decomissioning 超时。最大值为 1d
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

##### DataprocClusterDeleteOperator

```py
class airflow.contrib.operators.dataproc_operator.DataprocClusterDeleteOperator(cluster_name，project_id，region ='global'，gcp_conn_id ='google_cloud_default'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

删除 Google Cloud Dataproc 上的集群。operator 将等待，直到集群被销毁。

参数：

*   `cluster_name(str)` - 要创建的集群的名称。（模板渲染后）
*   `project_id(str)` - 集群运行的 Google 云项目的 ID。（模板渲染后）
*   `region(str)` - 保留为“全局”，将来可能会变得相关。（模板渲染后）
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


##### DataProcPigOperator

```py
class airflow.contrib.operators.dataproc_operator.DataProcPigOperator(query = None，query_uri = None，variables = None，job_name ='{{task.task_id}}_{{ds_nodash}}'，cluster_name ='cluster-1'，dataproc_pig_properties = None，dataproc_pig_jars = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，region ='global'，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

在 Cloud DataProc 集群上启动 Pig 查询作业。操作的参数将传递给集群。

在 dag 的 default_args 中定义 dataproc_*参数是一种很好的做法，比如集群名称和 UDF。

```py
 default_args = {
    'cluster_name' : 'cluster-1' ,
    'dataproc_pig_jars' : [
        'gs://example/udf/jar/datafu/1.2.0/datafu.jar' ,
        'gs://example/udf/jar/gpig/1.2/gpig.jar'
    ]
}

```

您可以将 pig 脚本作为字符串或文件引用传递。使用变量传递要在集群上解析的 pig 脚本的变量，或者使用要在脚本中解析的参数作为模板参数。

**示例** ：

```py
 t1 = DataProcPigOperator (
        task_id = 'dataproc_pig' ,
        query = 'a_pig_script.pig' ,
        variables = { 'out' : 'gs://example/output/{{ds}}' },
        dag = dag )

```

也可以看看

有关工作提交的更多详细信息，请参阅以下参考：[https://cloud.google.com/dataproc/reference/rest/v1/projects.regions.jobs](https://cloud.google.com/dataproc/reference/rest/v1/projects.regions.jobs)

参数：

*   `query(str)` - 对查询文件的查询或引用(pg 或 pig 扩展）。（模板渲染后)
*   `query_uri(str)` - 云存储上的猪脚本的 uri。
*   `variables(dict)` - 查询的命名参数的映射。（模板渲染后）
*   `job_name(str)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。（模板渲染后）
*   `cluster_name(str)` - DataProc 集群的名称。（模板渲染后）
*   `dataproc_pig_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_pig_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(str)` - 创建数据加载集群的指定区域。


##### DataProcHiveOperator

```py
class airflow.contrib.operators.dataproc_operator.DataProcHiveOperator(query = None，query_uri = None，variables = None，job_name ='{{task.task_id}}_{{ds_nodash}}'，cluster_name ='cluster-1'，dataproc_hive_properties = None，dataproc_hive_jars = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，region ='global'，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

在 Cloud DataProc 集群上启动 Hive 查询作业。

参数：

*   `query(str)` - 查询或对查询文件的引用（q 扩展名）。
*   `query_uri(str)` - 云存储上的 hive 脚本的 uri。
*   `variables(dict)` - 查询的命名参数的映射。
*   `job_name(str)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。
*   `cluster_name(str)` - DataProc 集群的名称。
*   `dataproc_hive_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_hive_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(str)` - 创建数据加载集群的指定区域。


##### DataProcSparkSqlOperator

```py
class airflow.contrib.operators.dataproc_operator.DataProcSparkSqlOperator(query = None，query_uri = None，variables = None，job_name ='{{task.task_id}}_{{ds_nodash}}'，cluster_name ='cluster-1'，dataproc_spark_properties = None，dataproc_spark_jars = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，region ='global'，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

在 Cloud DataProc 集群上启动 Spark SQL 查询作业。

参数：

*   `query(str)` - 查询或对查询文件的引用（q 扩展名）。（模板渲染后）
*   `query_uri(str)` - 云存储上的一个 spark sql 脚本的 uri。
*   `variables(dict)` - 查询的命名参数的映射。（模板渲染后）
*   `job_name(str)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。（模板渲染后）
*   `cluster_name(str)` - DataProc 集群的名称。（模板渲染后）
*   `dataproc_spark_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_spark_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(str)` - 创建数据加载集群的指定区域。


##### DataProcSparkOperator

```py
class airflow.contrib.operators.dataproc_operator.DataProcSparkOperator(main_jar = None，main_class = None，arguments = None，archives = None，files = None，job_name ='{{task.task_id}}_{{ds_nodash}}'，cluster_name ='cluster-1'，dataproc_spark_properties = None，dataproc_spark_jars = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，region ='global'，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

在 Cloud DataProc 集群上启动 Spark 作业。

参数：

*   `main_jar(str)` - 在云存储上配置的作业 jar 的 URI。（使用 this 或 main_class，而不是两者一起）。
*   `main_class(str)` - 作业类的名称。（使用 this 或 main_jar，而不是两者一起）。
*   `arguments(list)` - 作业的参数。（模板渲染后）
*   `archives(list)` - 将在工作目录中解压缩的已归档文件列表。应存储在云存储中。
*   `files(list)` - 要复制到工作目录的文件列表
*   `job_name(str)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。（模板渲染后）
*   `cluster_name(str)` - DataProc 集群的名称。（模板渲染后）
*   `dataproc_spark_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_spark_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(str)` - 创建数据加载集群的指定区域。


##### DataProcHadoopOperator

```py
class airflow.contrib.operators.dataproc_operator.DataProcHadoopOperator(main_jar = None，main_class = None，arguments = None，archives = None，files = None，job_name ='{{task.task_id}}_{{ds_nodash}}'，cluster_name ='cluster-1'，dataproc_hadoop_properties = None，dataproc_hadoop_jars = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，region ='global'，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

在 Cloud DataProc 集群上启动 Hadoop 作业。

参数：

*   `main_jar(str)` - 在云存储上配置的作业 jar 的 URI。（使用 this 或 main_class，而不是两者一起）。
*   `main_class(str)` - 作业类的名称。（使用 this 或 main_jar，而不是两者一起）。
*   `arguments(list)` - 作业的参数。（模板渲染后）
*   `archives(list)` - 将在工作目录中解压缩的已归档文件列表。应存储在云存储中。
*   `files(list)` - 要复制到工作目录的文件列表
*   `job_name(str)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。（模板渲染后）
*   `cluster_name(str)` - DataProc 集群的名称。（模板渲染后）
*   `dataproc_hadoop_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_hadoop_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(str)` - 创建数据加载集群的指定区域。


##### DataProcPySparkOperator

```py
class airflow.contrib.operators.dataproc_operator.DataProcPySparkOperator(main，arguments = None，archives = None，pyfiles = None，files = None，job_name ='{{task.task_id}}_{{ds_nodash}}'，cluster_name =' cluster-1'，dataproc_pyspark_properties = None，dataproc_pyspark_jars = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，region ='global'，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

在 Cloud DataProc 集群上启动 PySpark 作业。

参数：

*   `main(str)` - [必需]用作驱动程序的主 Python 文件的 Hadoop 兼容文件系统（HCFS）URI。必须是.py 文件。
*   `arguments(list)` - 作业的参数。（模板渲染后）
*   `archives(list)` - 将在工作目录中解压缩的已归档文件列表。应存储在云存储中。
*   `files(list)` - 要复制到工作目录的文件列表
*   `pyfiles(list)` - 要传递给 PySpark 框架的 Python 文件列表。支持的文件类型：.py，.egg 和.zip
*   `job_name(str)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。（模板渲染后）
*   `cluster_name(str)` - DataProc 集群的名称。
*   `dataproc_pyspark_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_pyspark_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(str)` - 创建数据加载集群的指定区域。


##### DataprocWorkflowTemplateInstantiateOperator

```py
class airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateInstantiateOperator(template_id，*args，**kwargs)
```

基类： [`airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateBaseOperator`]

在 Google Cloud Dataproc 上实例化 WorkflowTemplate。operator 将等待 WorkflowTemplate 完成执行。

也可以看看

请参阅：[https://cloud.google.com/dataproc/docs/reference/rest/v1beta2/projects.regions.workflowTemplates/instantiate](https://cloud.google.com/dataproc/docs/reference/rest/v1beta2/projects.regions.workflowTemplates/instantiate)

参数：

*   `template_id(str)` - 模板的 id。（模板渲染后）
*   `project_id(str)` - 模板运行所在的 Google 云项目的 ID
*   `region(str)` - 保留为“全局”，将来可能会变得相关
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


##### DataprocWorkflowTemplateInstantiateInlineOperator

```py
class airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateInstantiateInlineOperator(template，*args，**kwargs)
```

基类： [`airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateBaseOperator`]

在 Google Cloud Dataproc 上实例化 WorkflowTemplate 内联。operator 将等待 WorkflowTemplate 完成执行。

也可以看看

请参阅：[https://cloud.google.com/dataproc/docs/reference/rest/v1beta2/projects.regions.workflowTemplates/instantiateInline](https://cloud.google.com/dataproc/docs/reference/rest/v1beta2/projects.regions.workflowTemplates/instantiateInline)

参数：

*   `template(map)` - 模板内容。（模板渲染后）
*   `project_id(str)` - 模板运行所在的 Google 云项目的 ID
*   `region(str)` - 保留为“全局”，将来可能会变得相关
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


### 云数据存储区

#### 数据存储区运营商

*   [DatastoreExportOperator](#DatastoreExportOperator)：将实体从 Google Cloud Datastore 导出到云存储。
*   [DatastoreImportOperator](#DatastoreImportOperator)：将实体从云存储导入 Google Cloud Datastore。

##### DatastoreExportOperator

```py
class airflow.contrib.operators.datastore_export_operator.DatastoreExportOperator(bucket，namespace = None，datastore_conn_id ='google_cloud_default'，cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，entity_filter = None，labels = None，polling_interval_in_seconds = 10，overwrite_existing = False，xcom_push = False，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

将实体从 Google Cloud Datastore 导出到云存储

参数：

*   `bucket(str)` - 要备份数据的云存储桶的名称
*   `namespace(str)` - 指定云存储桶中用于备份数据的可选命名空间路径。如果 GCS 中不存在此命名空间，则将创建该命名空间。
*   `datastore_conn_id(str)` - 要使用的数据存储区连接 ID 的名称
*   `cloud_storage_conn_id(str)` - 强制写入备份的云存储连接 ID 的名称
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `entity_filter(dict)` - 导出中包含项目中哪些数据的说明，请参阅[https://cloud.google.com/datastore/docs/reference/rest/Shared.Types/EntityFilter](https://cloud.google.com/datastore/docs/reference/rest/Shared.Types/EntityFilter)
*   `labels(dict)` - 客户端分配的云存储标签
*   `polling_interval_in_seconds(int)` - 再次轮询执行状态之前等待的秒数
*   `overwrite_existing(bool)` - 如果存储桶+命名空间不为空，则在导出之前将清空它。这样可以覆盖现有备份。
*   `xcom_push(bool)` - 将操作名称推送到 xcom 以供参考


##### DatastoreImportOperator

```py
class airflow.contrib.operators.datastore_import_operator.DatastoreImportOperator(bucket，file，namespace = None，entity_filter = None，labels = None，datastore_conn_id ='google_cloud_default'，delegate_to = None，polling_interval_in_seconds = 10，xcom_push = False，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

将实体从云存储导入 Google Cloud Datastore

参数：

*   `bucket(str)` - 云存储中用于存储数据的容器
*   `file(str)` - 指定云存储桶中备份元数据文件的路径。它应该具有扩展名.overall_export_metadata
*   `namespace(str)` - 指定云存储桶中备份元数据文件的可选命名空间。
*   `entity_filter(dict)` - 导出中包含项目中哪些数据的说明，请参阅[https://cloud.google.com/datastore/docs/reference/rest/Shared.Types/EntityFilter](https://cloud.google.com/datastore/docs/reference/rest/Shared.Types/EntityFilter)
*   `labels(dict)` - 客户端分配的云存储标签
*   `datastore_conn_id(str)` - 要使用的连接 ID 的名称
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `polling_interval_in_seconds(int)` - 再次轮询执行状态之前等待的秒数
*   `xcom_push(bool)` - 将操作名称推送到 xcom 以供参考


#### DatastoreHook

```py
class airflow.contrib.hooks.datastore_hook.DatastoreHook(datastore_conn_id ='google_cloud_datastore_default'，delegate_to = None)
```

基类： [`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`]

与 Google Cloud Datastore 互动。此挂钩使用 Google Cloud Platform 连接。

此对象不是线程安全的。如果要同时发出多个请求，则需要为每个线程创建一个钩子。

```py
allocate_ids(partialKeys)
```

为不完整的密钥分配 ID。请参阅[https://cloud.google.com/datastore/docs/reference/rest/v1/projects/allocateIds](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/allocateIds)

参数：**partialKeys** - 部分键列表

返回：完整密钥列表。

```py
begin_transaction()
```

获取新的事务处理

> 也可以看看
>
> [https://cloud.google.com/datastore/docs/reference/rest/v1/projects/beginTransaction](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/beginTransaction)

返回：交易句柄

```py
commit(body)
```

提交事务，可选地创建，删除或修改某些实体。

也可以看看

[https://cloud.google.com/datastore/docs/reference/rest/v1/projects/commit](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/commit)

参数：**body** - 提交请求的主体

返回：提交请求的响应主体

```py
delete_operation（名称）
```

删除长时间运行的操作

参数：**name** - 操作资源的名称


```py
export_to_storage_bucket(bucket，namespace = None，entity_filter = None，labels = None)
```

将实体从 Cloud Datastore 导出到 Cloud Storage 进行备份

```py
get_conn（version= 'V1'）
```

返回 Google 云端存储服务对象。

```py
GET_OPERATION（name）
```

获取长时间运行的最新状态

参数：**name** - 操作资源的名称


```py
import_from_storage_bucket(bucket，file，namespace = None，entity_filter = None，labels = None)
```

将备份从云存储导入云数据存储

```py
lookup(keys，read_consistency = None，transaction = None)
```

按键查找一些实体

也可以看看

[https://cloud.google.com/datastore/docs/reference/rest/v1/projects/lookup](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/lookup)

参数：

*   **keys** - 要查找的键
*   **read_consistency** - 要使用的读取一致性。默认，强或最终。不能与事务一起使用。
*   **transaction** - 要使用的事务，如果有的话。

返回：查找请求的响应主体。

```py
poll_operation_until_done(name，polling_interval_in_seconds)
```

轮询备份操作状态直到完成

```py
rollback(transaction)
```

回滚事务

也可以看看

[https://cloud.google.com/datastore/docs/reference/rest/v1/projects/rollback](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/rollback)

参数：**transaction** - 要回滚的事务


```py
run_query（体）
```

运行实体查询。

也可以看看

[https://cloud.google.com/datastore/docs/reference/rest/v1/projects/runQuery](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/runQuery)

参数：**body** - 查询请求的主体

返回：批量查询结果。

### 云 ML 引擎

#### 云 ML 引擎运营商

*   [MLEngineBatchPredictionOperator](#MLEngineBatchPredictionOperator)：启动 Cloud ML Engine 批量预测作业。
*   [MLEngineModelOperator](#MLEngineModelOperator)：管理 Cloud ML Engine 模型。
*   [MLEngineTrainingOperator](#MLEngineTrainingOperator)：启动 Cloud ML Engine 培训工作。
*   [MLEngineVersionOperator](#MLEngineVersionOperator)：管理 Cloud ML Engine 模型版本。

##### MLEngineBatchPredictionOperator

```py
class airflow.contrib.operators.mlengine_operator.MLEngineBatchPredictionOperator(project_id，job_id，region，data_format，input_paths，output_path，model_name = None，version_name = None，uri = None，max_worker_count = None，runtime_version = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

启动 Google Cloud ML Engine 预测作业。

注意：对于模型原点，用户应该考虑以下三个选项中的一个：1。仅填充“uri”字段，该字段应该是指向 tensorflow savedModel 目录的 GCS 位置。2.仅填充'model_name'字段，该字段引用现有模型，并将使用模型的默认版本。3.填充“model_name”和“version_name”字段，这些字段指特定模型的特定版本。

在选项 2 和 3 中，模型和版本名称都应包含最小标识符。例如，打电话

```py
 MLEngineBatchPredictionOperator (
    ... ,
    model_name = 'my_model' ,
    version_name = 'my_version' ,
    ... )

```

如果所需的型号版本是“projects / my_project / models / my_model / versions / my_version”。

有关参数的更多文档，请参阅[https://cloud.google.com/ml-engine/reference/rest/v1/projects.jobs](https://cloud.google.com/ml-engine/reference/rest/v1/projects.jobs)。

参数：

*   `project_id(str)` - 提交预测作业的 Google Cloud 项目名称。（模板渲染后）
*   `job_id(str)` - Google Cloud ML Engine 上预测作业的唯一 ID。（模板渲染后）
*   `data_format(str)` - 输入数据的格式。如果未提供或者不是[“TEXT”，“TF_RECORD”，“TF_RECORD_GZIP”]之一，它将默认为“DATA_FORMAT_UNSPECIFIED”。
*   `input_paths(list[str])` - 批量预测的输入数据的 GCS 路径列表。接受通配符 operator [*](28)，但仅限于结尾处。（模板渲染后）
*   `output_path(str)` - 写入预测结果的 GCS 路径。（模板渲染后）
*   `region(str)` - 用于运行预测作业的 Google Compute Engine 区域。（模板化）
*   `model_name(str)` - 用于预测的 Google Cloud ML Engine 模型。如果未提供 version_name，则将使用此模型的默认版本。如果提供了 version_name，则不应为 None。如果提供 uri，则应为 None。（模板渲染后）
*   `version_name(str)` - 用于预测的 Google Cloud ML Engine 模型版本。如果提供 uri，则应为 None。（模板渲染后）
*   `uri(str)` - 用于预测的已保存模型的 GCS 路径。如果提供了 model_name，则应为 None。它应该是指向张量流 SavedModel 的 GCS 路径。（模板渲染后）
*   `max_worker_count(int)` - 用于并行处理的最大 worker 数。如果未指定，则默认为 10。
*   `runtime_version(str)` - 用于批量预测的 Google Cloud ML Engine 运行时版本。
*   `gcp_conn_id(str)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用 doamin 范围的委派。


```py
Raises:
`ValueError` ：如果无法确定唯一的模型/版本来源。
```

##### MLEngineModelOperator

```py
class airflow.contrib.operators.mlengine_operator.MLEngineModelOperator(project_id，model，operation ='create'，gcp_conn_id ='google_cloud_default'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

管理 Google Cloud ML Engine 模型的运营商。

参数：

*   `project_id(str)` - MLEngine 模型所属的 Google Cloud 项目名称。（模板渲染后）
*   `model(dict)` - 

包含有关模型信息的字典。如果`操作`是`create`，则`model`参数应包含有关此模型的所有信息，例如`name`。

    如果`操作`是`get`，则`model`参数应包含`模型`的`名称`。

*   `操作` - 

执行的操作。可用的操作是：

    *   `create`：创建`model`参数提供的新模型。
    *   `get`：获取在模型中指定名称的特定`模型`。
*   `gcp_conn_id(str)` - 获取连接信息时使用的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


##### MLEngineTrainingOperator

```py
class airflow.contrib.operators.mlengine_operator.MLEngineTrainingOperator(project_id，job_id，package_uris，training_python_module，training_args，region，scale_tier = None，runtime_version = None，python_version = None，job_dir = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，mode ='PRODUCTION'，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

启动 MLEngine 培训工作的operator 。

参数：

*   `project_id(str)` - 应在其中运行 MLEngine 培训作业的 Google Cloud 项目名称（模板化）。
*   `job_id(str)` - 提交的 Google MLEngine 培训作业的唯一模板化 ID。（模板渲染后）
*   `package_uris(str)` - MLEngine 培训作业的包位置列表，其中应包括主要培训计划+任何其他依赖项。（模板渲染后）
*   `training_python_module(str)` - 安装'package_uris'软件包后，在 MLEngine 培训作业中运行的 Python 模块名称。（模板渲染后）
*   `training_args(str)` - 传递给 MLEngine 训练程序的模板化命令行参数列表。（模板渲染后）
*   `region(str)` - 用于运行 MLEngine 培训作业的 Google Compute Engine 区域（模板化）。
*   `scale_tier(str)` - MLEngine 培训作业的资源层。（模板渲染后）
*   `runtime_version(str)` - 用于培训的 Google Cloud ML 运行时版本。（模板渲染后）
*   `python_version(str)` - 训练中使用的 Python 版本。（模板渲染后）
*   `job_dir(str)` - 用于存储培训输出和培训所需的其他数据的 Google 云端存储路径。（模板渲染后）
*   `gcp_conn_id(str)` - 获取连接信息时使用的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `mode(str)` - 可以是'DRY_RUN'/'CLOUD'之一。在“DRY_RUN”模式下，不会启动真正的培训作业，但会打印出 MLEngine 培训作业请求。在“CLOUD”模式下，将发出真正的 MLEngine 培训作业创建请求。


##### MLEngineVersionOperator

```py
class airflow.contrib.operators.mlengine_operator.MLEngineVersionOperator(project_id，model_name，version_name = None，version = None，operation ='create'，gcp_conn_id ='google_cloud_default'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

管理 Google Cloud ML Engine 版本的运营商。

参数：

*   `project_id(str)` - MLEngine 模型所属的 Google Cloud 项目名称。
*   `model_name(str)` - 版本所属的 Google Cloud ML Engine 模型的名称。（模板渲染后）
*   `version_name(str)` - 用于正在操作的版本的名称。如果没有人及`版本`的说法是没有或不具备的值`名称`键，那么这将是有效载荷中用于填充`名称`键。（模板渲染后）
*   `version(dict)` - 包含版本信息的字典。如果`操作`是`create`，则`version`应包含有关此版本的所有信息，例如 name 和 deploymentUrl。如果`操作`是`get`或`delete`，则`version`参数应包含`版本`的`名称`。如果是 None，则唯一可能的`操作`是`list`。（模板渲染后）
*   `operation(str)` -

    执行的操作。可用的操作是：

    *   `create`：在`model_name`指定的`模型中`创建新版本，在这种情况下，`version`参数应包含创建该版本的所有信息（例如`name`，`deploymentUrl`）。
    *   `get`：获取`model_name`指定的`模型中`特定版本的完整信息。应在`version`参数中指定版本的名称。
    *   `list`：列出`model_name`指定的`模型的`所有可用版本。
    *   `delete`：从`model_name`指定的`模型中`删除`version`参数中指定的`版本`。应在`version`参数中指定版本的名称。
*   `gcp_conn_id(str)` - 获取连接信息时使用的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


#### Cloud ML Engine Hook

##### MLEngineHook

```py
class airflow.contrib.hooks.gcp_mlengine_hook.MLEngineHook(gcp_conn_id ='google_cloud_default'，delegate_to = None)
```

基类： [`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`]

```py
create_job(project_id，job，use_existing_job_fn = None)
```

启动 MLEngine 作业并等待它达到终端状态。

参数：

*   `project_id(str)` - 将在其中启动 MLEngine 作业的 Google Cloud 项目 ID。
*   `job(dict)` -

    应该提供给 MLEngine API 的 MLEngine Job 对象，例如：

    ```py
     {
      'jobId' : 'my_job_id' ,
      'trainingInput' : {
        'scaleTier' : 'STANDARD_1' ,
        ...
      }
    }

    ```

*   `use_existing_job_fn(function)` - 如果已存在具有相同 job_id 的 MLEngine 作业，则此方法（如果提供）将决定是否应使用此现有作业，继续等待它完成并返回作业对象。它应该接受 MLEngine 作业对象，并返回一个布尔值，指示是否可以重用现有作业。如果未提供“use_existing_job_fn”，我们默认重用现有的 MLEngine 作业。

返回：如果作业成功到达终端状态（可能是 FAILED 或 CANCELED 状态），则为 MLEngine 作业对象。

返回类型：字典

```py
create_model(project_id，model)
```

创建一个模型。阻止直到完成。

```py
create_version(project_id，model_name，version_spec)
```

在 Google Cloud ML Engine 上创建版本。

如果版本创建成功则返回操作，否则引发错误。

```py
delete_version(project_id，model_name，version_name)
```

删除给定版本的模型。阻止直到完成。

```py
get_conn()
```

返回 Google MLEngine 服务对象。

```py
get_model(project_id，model_name)
```

获取一个模型。阻止直到完成。

```py
list_versions(project_id，model_name)
```

列出模型的所有可用版本。阻止直到完成。

```py
set_default_version(project_id，model_name，version_name)
```

将版本设置为默认值。阻止直到完成。

### 云储存

#### 存储运营商

*   [FileToGoogleCloudStorageOperator](#FileToGoogleCloudStorageOperator)：将文件上传到 Google 云端存储。
*   [GoogleCloudStorageCreateBucketOperator](#GoogleCloudStorageCreateBucketOperator)：创建新的云存储桶。
*   [GoogleCloudStorageListOperator](#GoogleCloudStorageListOperator)：列出存储桶中的所有对象，并在名称中添加字符串前缀和分隔符。
*   [GoogleCloudStorageDownloadOperator](#GoogleCloudStorageDownloadOperator)：从 Google 云端存储中下载文件。
*   [GoogleCloudStorageToBigQueryOperator](#GoogleCloudStorageToBigQueryOperator)：将 Google 云存储中的文件加载到 BigQuery 中。
*   [GoogleCloudStorageToGoogleCloudStorageOperator](#GoogleCloudStorageToGoogleCloudStorageOperator)：将对象从存储桶复制到另一个存储桶，并在需要时重命名。

##### FileToGoogleCloudStorageOperator

```py
class airflow.contrib.operators.file_to_gcs.FileToGoogleCloudStorageOperator(src，dst，bucket，google_cloud_storage_conn_id ='google_cloud_default'，mime_type ='application / octet-stream'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

将文件上传到 Google 云端存储

参数：

*   `src(str)` - 本地文件的路径。（模板渲染后）
*   `dst(str)` - 指定存储桶中的目标路径。（模板渲染后）
*   `bucket(str)` - 要上传的存储桶。（模板渲染后）
*   `google_cloud_storage_conn_id(str)` - 要上传的 Airflow 连接 ID
*   `mime_type(str)` - mime 类型字符串
*   `delegate_to(str)` - 代理的帐户（如果有）


```py
execute(context)
```

将文件上传到 Google 云端存储

##### GoogleCloudStorageCreateBucketOperator

```py
class airflow.contrib.operators.gcs_operator.GoogleCloudStorageCreateBucketOperator(bucket_name，storage_class ='MULTI_REGIONAL'，location ='US'，project_id = None，labels = None，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

创建一个新存储桶。Google 云端存储使用平面命名空间，因此您无法创建名称已在使用中的存储桶。

> 也可以看看
>
> 有关详细信息，请参阅存储桶命名指南：[https://cloud.google.com/storage/docs/bucketnaming.html#requirements](https://cloud.google.com/storage/docs/bucketnaming.html#requirements)

参数：

*   `bucket_name(str)` - 存储桶的名称。（模板渲染后）
*   `storage_class(str)` -

    这定义了存储桶中对象的存储方式，并确定了 SLA 和存储成本（模板化）。价值包括

    *   `MULTI_REGIONAL`
    *   `REGIONAL`
    *   `STANDARD`
    *   `NEARLINE`
    *   `COLDLINE`

    如果在创建存储桶时未指定此值，则默认为 STANDARD。

*   `location(str)` -

    水桶的位置。（模板化）存储桶中对象的对象数据驻留在此区域内的物理存储中。默认为美国。

    也可以看看

    [https://developers.google.com/storage/docs/bucket-locations](https://developers.google.com/storage/docs/bucket-locations)

*   `project_id(str)` - GCP 项目的 ID。（模板渲染后）
*   `labels(dict)` - 用户提供的键/值对标签。
*   `google_cloud_storage_conn_id(str)` - 连接到 Google 云端存储时使用的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


示例:

以下 operator 将在区域中创建`test-bucket`具有`MULTI_REGIONAL`存储类的新存储桶`EU`

```py
 CreateBucket = GoogleCloudStorageCreateBucketOperator (
    task_id = 'CreateNewBucket' ,
    bucket_name = 'test-bucket' ,
    storage_class = 'MULTI_REGIONAL' ,
    location = 'EU' ,
    labels = { 'env' : 'dev' , 'team' : 'airflow' },
    google_cloud_storage_conn_id = 'airflow-service-account'
)

```

##### GoogleCloudStorageDownloadOperator

```py
class airflow.contrib.operators.gcs_download_operator.GoogleCloudStorageDownloadOperator(bucket，object，filename = None，store_to_xcom_key = None，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

从 Google 云端存储下载文件。

参数：

*   `bucket(str)` - 对象所在的 Google 云存储桶。（模板渲染后）
*   `object(str)` - 要在 Google 云存储桶中下载的对象的名称。（模板渲染后）
*   `filename(str)` - 应将文件下载到的本地文件系统（正在执行操作符的位置）上的文件路径。（模板化）如果未传递文件名，则下载的数据将不会存储在本地文件系统中。
*   `store_to_xcom_key(str)` - 如果设置了此参数，operator 将使用此参数中设置的键将下载文件的内容推送到 XCom。如果未设置，则下载的数据不会被推送到 XCom。（模板渲染后）
*   `google_cloud_storage_conn_id(str)` - 连接到 Google 云端存储时使用的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


##### GoogleCloudStorageListOperator

```py
class airflow.contrib.operators.gcslistoperator.GoogleCloudStorageListOperator(bucket，prefix = None，delimiter = None，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

使用名称中的给定字符串前缀和分隔符列出存储桶中的所有对象。

```
此 operator 返回一个 python 列表，其中包含可供其使用的对象的名称
```

`xcom`在下游任务中。

参数：

*   `bucket(str)` - 用于查找对象的 Google 云存储桶。（模板渲染后）
*   `prefix(str)` - 前缀字符串，用于过滤名称以此前缀开头的对象。（模板渲染后）
*   `delimiter(str)` - 要过滤对象的分隔符。（模板化）例如，要列出 GCS 目录中的 CSV 文件，您可以使用 delimiter ='。csv'。
*   `google_cloud_storage_conn_id(str)` - 连接到 Google 云端存储时使用的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

示例：

以下 operator 将列出存储桶中文件`sales/sales-2017`夹中的所有 Avro 文件`data`。

```py
 GCS_Files = GoogleCloudStorageListOperator (
    task_id = 'GCS_Files' ,
    bucket = 'data' ,
    prefix = 'sales/sales-2017/' ,
    delimiter = '.avro' ,
    google_cloud_storage_conn_id = google_cloud_conn_id
)

```

##### GoogleCloudStorageToBigQueryOperator

```py
class airflow.contrib.operators.gcs_to_bq.GoogleCloudStorageToBigQueryOperator(bucket，source_objects，destination_project_dataset_table，schema_fields = None，schema_object = None，source_format ='CSV'，compression ='NONE'，create_disposition ='CREATE_IF_NEEDED'，skip_leading_rows = 0，write_disposition =' WRITE_EMPTY'，field_delimiter ='，'，max_bad_records = 0，quote_character = None，ignore_unknown_values = False，allow_quoted_newlines = False，allow_jagged_rows = False，max_id_key = None，bigquery_conn_id ='bigquery_default'，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，schema_update_options =()，src_fmt_configs = {}，external_table = False，time_partitioning = {}，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

将文件从 Google 云存储加载到 BigQuery 中。

可以用两种方法之一指定用于 BigQuery 表的模式。您可以直接传递架构字段，也可以将运营商指向 Google 云存储对象名称。Google 云存储中的对象必须是包含架构字段的 JSON 文件。

参数：

*   `bucket(str)` - 要加载的桶。（模板渲染后）
*   `source_objects` - 要加载的 Google 云存储 URI 列表（模板化）。如果 source_format 是'DATASTORE_BACKUP'，则列表必须只包含一个 URI。
*   `destination_project_dataset_table(str)` - 用于加载数据的表(\<project>.)\<dataset>.\<table> BigQuery 表。如果未包\<project>，则项目将是连接 json 中定义的项目。（模板渲染后）
*   `schema_fields(list)` - 如果设置，则此处定义的架构字段列表：[https://cloud.google.com/bigquery/docs/reference/v2/jobs#configuration.load](https://cloud.google.com/bigquery/docs/reference/v2/jobs#configuration.load)当 source_format 为'DATASTORE_BACKUP'时，不应设置。
*   `schema_object` - 如果设置，则指向包含表的架构的.json 文件的 GCS 对象路径。（模板渲染后）
*   `schema_object` - 字符串
*   `source_format(str)` - 要导出的文件格式。
*   `compression(str)` - [可选]数据源的压缩类型。可能的值包括 GZIP 和 NONE。默认值为 NONE。Google Cloud Bigtable，Google Cloud Datastore 备份和 Avro 格式会忽略此设置。
*   `create_disposition(str)` - 如果表不存在，则创建处置。
*   `skip_leading_rows(int)` - 从 CSV 加载时要跳过的行数。
*   `write_disposition(str)` - 表已存在时的写处置。
*   `field_delimiter(str)` - 从 CSV 加载时使用的分隔符。
*   `max_bad_records(int)` - BigQuery 在运行作业时可以忽略的最大错误记录数。
*   `quote_character(str)` - 用于引用 CSV 文件中数据部分的值。
*   `ignore_unknown_values(bool)` - [可选]指示 BigQuery 是否应允许表模式中未表示的额外值。如果为 true，则忽略额外值。如果为 false，则将具有额外列的记录视为错误记录，如果错误记录太多，则在作业结果中返回无效错误。
*   `allow_quoted_newlines(bool)` - 是否允许引用的换行符（true）或不允许（false）。
*   `allow_jagged_rows(bool)` - 接受缺少尾随可选列的行。缺失值被视为空值。如果为 false，则缺少尾随列的记录将被视为错误记录，如果错误记录太多，则会在作业结果中返回无效错误。仅适用于 CSV，忽略其他格式。
*   `max_id_key(str)` - 如果设置，则是 BigQuery 表中要加载的列的名称。在加载发生后，Thsi 将用于从 BigQuery 中选择 MAX 值。结果将由 execute()命令返回，该命令又存储在 XCom 中供将来的operator 使用。这对增量加载很有帮助 - 在将来的执行过程中，您可以从最大 ID 中获取。
*   `bigquery_conn_id(str)` - 对特定 BigQuery 的引用。
*   `google_cloud_storage_conn_id(str)` - 对特定 Google 云存储挂钩的引用。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `schema_update_options(list)` - 允许更新目标表的模式作为加载作业的副作用。
*   `src_fmt_configs(dict)` - 配置特定于源格式的可选字段
*   `external_table(bool)` - 用于指定目标表是否应为 BigQuery 外部表的标志。默认值为 False。
*   `time_partitioning(dict)` - 配置可选的时间分区字段，即按 API 规范按字段，类型和到期分区。请注意，“field”在 dataset.table $ partition 的并发中不可用。


##### GoogleCloudStorageToGoogleCloudStorageOperator

```py
class airflow.contrib.operators.gcs_to_gcs.GoogleCloudStorageToGoogleCloudStorageOperator(source_bucket，source_object，destination_bucket = None，destination_object = None，move_object = False，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

将对象从存储桶复制到另一个存储桶，并在需要时重命名。

参数：

*   `source_bucket(str)` - 对象所在的源 Google 云存储桶。（模板渲染后）
*   `source_object(str)` -

    要在 Google 云存储分区中复制的对象的源名称。（模板化）如果在此参数中使用通配符：

    > 您只能在存储桶中使用一个通配符作为对象（文件名）。通配符可以出现在对象名称内或对象名称的末尾。不支持在存储桶名称中附加通配符。

*   `destination_bucket` - 目标 Google 云端存储分区


对象应该在哪里。（模板化）：type destination_bucket：string：param destination_object：对象的目标名称

> 目标 Google 云存储桶。（模板化）如果在 source_object 参数中提供了通配符，则这是将添加到最终目标对象路径的前缀。请注意，将删除通配符之前的源路径部分; 如果需要保留，则应将其附加到 destination_object。例如，使用 prefix `foo/*`和 destination_object'blah `/``，文件`foo/baz`将被复制到`blah/baz`; 保留前缀写入 destination_object，例如`blah/foo`，在这种情况下，复制的文件将被命名`blah/foo/baz`。

参数：**move_object** - 当移动对象为 True 时，移动对象


```py
 复制到新位置。
```

这相当于 mv 命令而不是 cp 命令。

参数：

*   `google_cloud_storage_conn_id(str)` - 连接到 Google 云端存储时使用的连接 ID。
*   `delegate_to(str)` - 代理的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


```py
 Examples:
```

下面的操作将命名一个文件复制`sales/sales-2017/january.avro`在`data`桶的文件和名为斗`copied_sales/2017/january-backup.avro` in the ``data_backup`

```py
 copy_single_file = GoogleCloudStorageToGoogleCloudStorageOperator (
    task_id = 'copy_single_file' ,
    source_bucket = 'data' ,
    source_object = 'sales/sales-2017/january.avro' ,
    destination_bucket = 'data_backup' ,
    destination_object = 'copied_sales/2017/january-backup.avro' ,
    google_cloud_storage_conn_id = google_cloud_conn_id
)

```

以下 operator 会将文件`sales/sales-2017`夹中的所有 Avro 文件（即名称以该前缀开头）复制到存储`data`桶中的`copied_sales/2017`文件夹中`data_backup`。

```py
 copy_files = GoogleCloudStorageToGoogleCloudStorageOperator (
    task_id = 'copy_files' ,
    source_bucket = 'data' ,
    source_object = 'sales/sales-2017/*.avro' ,
    destination_bucket = 'data_backup' ,
    destination_object = 'copied_sales/2017/' ,
    google_cloud_storage_conn_id = google_cloud_conn_id
)

```

以下 operator 会将文件`sales/sales-2017`夹中的所有 Avro 文件（即名称以该前缀开头）移动到`data`存储桶中的同一文件夹`data_backup`，删除过程中的原始文件。

```py
 move_files = GoogleCloudStorageToGoogleCloudStorageOperator (
    task_id = 'move_files' ,
    source_bucket = 'data' ,
    source_object = 'sales/sales-2017/*.avro' ,
    destination_bucket = 'data_backup' ,
    move_object = True ,
    google_cloud_storage_conn_id = google_cloud_conn_id
)

```

#### GoogleCloudStorageHook

```py
class airflow.contrib.hooks.gcs_hook.GoogleCloudStorageHook(google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None)
```

基类： [`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`]

与 Google 云端存储互动。此挂钩使用 Google Cloud Platform 连接。

```py
copy(source_bucket，source_object，destination_bucket = None，destination_object = None)
```

将对象从存储桶复制到另一个存储桶，并在需要时重命名。

destination_bucket 或 destination_object 可以省略，在这种情况下使用源桶/对象，但不能同时使用两者。

参数：

*   `source_bucket(str)` - 要从中复制的对象的存储桶。
*   `source_object(str)` - 要复制的对象。
*   `destination_bucket(str)` - 要复制到的对象的目标。可以省略; 然后使用相同的桶。
*   `destination_object` - 给定对象的（重命名）路径。可以省略; 然后使用相同的名称。


```py
create_bucket(bucket_name，storage_class ='MULTI_REGIONAL'，location ='US'，project_id = None，labels = None)
```

创建一个新存储桶。Google 云端存储使用平面命名空间，因此您无法创建名称已在使用中的存储桶。

也可以看看

有关详细信息，请参阅存储桶命名指南：[https://cloud.google.com/storage/docs/bucketnaming.html#requirements](https://cloud.google.com/storage/docs/bucketnaming.html#requirements)

参数：

*   `bucket_name(str)` - 存储桶的名称。
*   `storage_class(str)` -

    这定义了存储桶中对象的存储方式，并确定了 SLA 和存储成本。价值包括

    *   `MULTI_REGIONAL`
    *   `REGIONAL`
    *   `STANDARD`
    *   `NEARLINE`
    *   `COLDLINE` 。

    如果在创建存储桶时未指定此值，则默认为 STANDARD。

*   `location(str)` -

    桶的位置。存储桶中对象的对象数据驻留在此区域内的物理存储中。默认为美国。

    也可以看看

    [https://developers.google.com/storage/docs/bucket-locations](https://developers.google.com/storage/docs/bucket-locations)

*   `project_id(str)` - GCP 项目的 ID。
*   `labels(dict)` - 用户提供的键/值对标签。

返回：如果成功，则返回`id`桶的内容。

```py
delete(bucket，object，generation=None）
```

如果未对存储桶启用版本控制，或者使用了生成参数，则删除对象。

参数：

*   `bucket(str)` - 对象所在的存储桶的名称
*   `object(str)` - 要删除的对象的名称
*   `generation(str)` - 如果存在，则永久删除该代的对象

返回：如果成功则为真

```py
下载（bucket，object，filename = None）
```

从 Google 云端存储中获取文件。

参数：

*   `bucket(str)` - 要获取的存储桶。
*   `object(str)` - 要获取的对象。
*   `filename(str)` - 如果设置，则应写入文件的本地文件路径。


```py
exists(budket, object)
```

检查 Google 云端存储中是否存在文件。

参数：

*   `bucket(str)` - 对象所在的 Google 云存储桶。
*   `object(str)` - 要在 Google 云存储分区中检查的对象的名称。


```py
get_conn()
```

返回 Google 云端存储服务对象。

```py
get_crc32c(bucket，object)
```

获取 Google Cloud Storage 中对象的 CRC32c 校验和。

参数：

*   `bucket(str)` - 对象所在的 Google 云存储桶。
*   `object(str)` - 要在 Google 云存储分区中检查的对象的名称。


```py
get_md5hash(bucket，object)
```

获取 Google 云端存储中对象的 MD5 哈希值。

参数：

*   `bucket(str)` - 对象所在的 Google 云存储桶。
*   `object(str)` - 要在 Google 云存储分区中检查的对象的名称。


```py
get_size(bucket，object)
```

获取 Google 云端存储中文件的大小。

参数：

*   `bucket(str)` - 对象所在的 Google 云存储桶。
*   `object(str)` - 要在 Google 云存储分区中检查的对象的名称。


```py
is_updated_after(bucket，object，ts)
```

检查 Google Cloud Storage 中是否更新了对象。

参数：

*   `bucket(str)` - 对象所在的 Google 云存储桶。
*   `object(str)` - 要在 Google 云存储分区中检查的对象的名称。
*   `ts(datetime)` - 要检查的时间戳。


```py
list(bucket，versions = None，maxResults = None，prefix = None，delimiter = None)
```

使用名称中的给定字符串前缀列出存储桶中的所有对象

参数：

*   `bucket(str)` - 存储桶名称
*   `versions(bool)` - 如果为 true，则列出对象的所有版本
*   `maxResults(int)` - 在单个响应页面中返回的最大项目数
*   `prefix(str)` - 前缀字符串，用于过滤名称以此前缀开头的对象
*   `delimiter(str)` - 根据分隔符过滤对象（例如'.csv'）

返回：与过滤条件匹配的对象名称流

```py
rewrite(source_bucket，source_object，destination_bucket，destination_object = None)
```

具有与复制相同的功能，除了可以处理超过 5 TB 的文件，以及在位置和/或存储类之间复制时。

destination_object 可以省略，在这种情况下使用 source_object。

参数：

*   `source_bucket(str)` - 要从中复制的对象的存储桶。
*   `source_object(str)` - 要复制的对象。
*   `destination_bucket(str)` - 要复制到的对象的目标。
*   `destination_object` - 给定对象的（重命名）路径。可以省略; 然后使用相同的名称。


```py
upload(bucket，object，filename，mime_type ='application / octet-stream')
```

将本地文件上传到 Google 云端存储。

参数：

*   `bucket(str)` - 要上传的存储桶。
*   `object(str)` - 上传本地文件时要设置的对象名称。
*   `filename(str)` - 要上传的文件的本地文件路径。
*   `mime_type(str)` - 上传文件时要设置的 MIME 类型。


### Google Kubernetes 引擎

#### Google Kubernetes 引擎集群 Operators

*   [GKEClusterDeleteOperator](#GKEClusterDeleteOperator)：在 Google Cloud Platform 中创建 Kubernetes 集群
*   [GKEPodOperator](#GKEPodOperator)：删除 Google Cloud Platform 中的 Kubernetes 集群

##### GKEClusterCreateOperator

```py
class airflow.contrib.operators.gcp_container_operator.GKEClusterCreateOperator(project_id，location，body = {}，gcp_conn_id ='google_cloud_default'，api_version ='v2'，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

##### GKEClusterDeleteOperator

```py
class airflow.contrib.operators.gcp_container_operator.GKEClusterDeleteOperator(project_id，name，location，gcp_conn_id ='google_cloud_default'，api_version ='v2'，*args，**kwargs)
```

基类： `airflow.models.BaseOperator`

##### GKEPodOperator

#### Google Kubernetes Engine Hook

```py
class airflow.contrib.hooks.gcp_container_hook.GKEClusterHook(project_id，location)
```

基类： `airflow.hooks.base_hook.BaseHook`

```py
create_cluster(cluster，retry = <object object>，timeout = <object object>)
```

创建一个集群，由指定数量和类型的 Google Compute Engine 实例组成。

参数：

*   `cluster(dict 或 google.cloud.container_v1.types.Cluster)` - 集群 protobuf 或 dict。如果提供了 dict，它必须与 protobuf 消息的格式相同 google.cloud.container_v1.types.Cluster
*   `重试(google.api_core.retry.Retry)` - 用于重试请求的重试对象（google.api_core.retry.Retry）。如果指定 None，则不会重试请求。
*   `timeout(float)` - 等待请求完成的时间（以秒为单位）。请注意，如果指定了重试，则超时适用于每次单独尝试。

返回：新集群或现有集群的完整 URL


raise

ParseError：在尝试转换 dict 时出现 JSON 解析问题 AirflowException：cluster 不是 dict 类型也不是 Cluster proto 类型

```py
delete_cluster(name，retry = <object object>，timeout = <object object>)
```

删除集群，包括 Kubernetes 端点和所有工作节点。在集群创建期间配置的防火墙和路由也将被删除。集群可能正在使用的其他 Google Compute Engine 资源（例如，负载均衡器资源）如果在初始创建时不存在，则不会被删除。

参数：

*   `name(str)` - 要删除的集群的名称
*   `retry(google.api_core.retry.Retry)` - 重试用于确定何时/是否重试请求的对象。如果指定 None，则不会重试请求。
*   `timeout(float)` - 等待请求完成的时间（以秒为单位）。请注意，如果指定了重试，则超时适用于每次单独尝试。

返回：如果成功则删除操作的完整 URL，否则为 None

```py
get_cluster(name，retry = <object object>，timeout = <object object>)
```

获取指定集群的详细信息：param name：要检索的集群的名称：type name：str：param retry：用于重试请求的重试对象。如果指定了 None，

> 请求不会被重试。

参数：`timeout(float)` - 等待请求完成的时间（以秒为单位）。请注意，如果指定了重试，则超时适用于每次单独尝试。

返回：一个 google.cloud.container_v1.types.Cluster 实例

```py
get_operation(operation_name)
```

从 Google Cloud 获取操作：param operation_name：要获取的操作的名称：type operation_name：str：return：来自 Google Cloud 的新的更新操作

```py
wait_for_operation（operation）
```

给定操作，持续从 Google Cloud 获取状态，直到完成或发生错误：param 操作：等待的操作：键入操作：google.cloud.container_V1.gapic.enums.Operator：return：a new，updated 从 Google Cloud 获取的操作
