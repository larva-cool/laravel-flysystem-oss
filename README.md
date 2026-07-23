# Laravel Flysystem OSS

<p align="center">
    <a href="https://packagist.org/packages/larva/laravel-flysystem-oss"><img src="https://poser.pugx.org/larva/laravel-flysystem-oss/v/stable" alt="Stable Version"></a>
    <a href="https://packagist.org/packages/larva/laravel-flysystem-oss"><img src="https://poser.pugx.org/larva/laravel-flysystem-oss/downloads" alt="Total Downloads"></a>
    <a href="https://packagist.org/packages/larva/laravel-flysystem-oss"><img src="https://poser.pugx.org/larva/laravel-flysystem-oss/license" alt="License"></a>
</p>

适用于 Laravel 的阿里云 OSS（Object Storage Service）Flysystem 适配器，完整支持 Flysystem v3 的所有文件操作接口，并提供临时 URL 签名、自定义访问域名等扩展能力。

## 环境要求

| 依赖 | 版本 |
| --- | --- |
| PHP | ^8.2 |
| Laravel Framework | ^12.0 \|\| ^13.0 |
| League Flysystem | ^3.0 |
| larva/flysystem-oss | ^1.0 |

## 安装

```bash
composer require larva/laravel-flysystem-oss -vv
```

该包通过 Laravel 的包发现机制自动注册服务提供者 `Larva\Flysystem\Aliyun\ObjectStorageServiceProvider`，无需手动注册。

## 配置

修改配置文件 `config/filesystems.php`，在 `disks` 中添加一个 `oss` 磁盘配置：

```php
'oss' => [
    'driver'        => 'oss',
    'access_id'     => env('OSS_ACCESS_ID', 'your access id'),
    'access_key'    => env('OSS_ACCESS_KEY', 'your access key'),
    'bucket'        => env('OSS_BUCKET', 'your bucket'),
    'endpoint'      => env('OSS_ENDPOINT', 'your endpoint'),  // 不要使用 CNAME，请使用 OSS 的 endpoint 地址
    'url'           => env('OSS_URL', 'cdn url'),             // 自定义访问域名，可以是 CDN 或绑定的域名，如 https://www.bbb.com，末尾不要斜杠
    'root'          => env('OSS_ROOT', ''),                   // 文件路径前缀，若所有内容存放在子目录中则填写，否则留空
    'visibility'    => 'public',                               // 默认可见性，可选：public / private
    'security_token' => null,                                  // STS 临时安全令牌，用于临时授权场景
    'proxy'         => null,                                   // HTTP 代理地址
    'timeout'       => 3600,                                   // 请求超时时间（秒）
    'ssl'           => true,                                   // 是否使用 HTTPS
],
```

如需将其设为默认存储驱动，修改 `default` 配置：

```php
'default' => 'oss',
```

### 环境变量示例

在 `.env` 文件中添加对应的环境变量：

```env
OSS_ACCESS_ID=your-access-id
OSS_ACCESS_KEY=your-access-key
OSS_BUCKET=your-bucket
OSS_ENDPOINT=oss-cn-hangzhou.aliyuncs.com
OSS_URL=https://cdn.example.com
OSS_ROOT=
```

## 使用

本包完全兼容 Laravel 文件系统 API，使用方式与本地存储一致，参见 [Laravel Filesystem 文档](https://laravel.com/docs/filesystem)。

### 基础文件操作

```php
use Illuminate\Support\Facades\Storage;

$disk = Storage::disk('oss');

// 写入文件
$disk->put('path/to/file.txt', 'file contents');

// 以流的方式写入
$disk->writeStream('path/to/file.txt', fopen('local-file.txt', 'r'));

// 读取文件
$contents = $disk->get('path/to/file.txt');

// 以流的方式读取
$stream = $disk->readStream('path/to/file.txt');

// 判断文件是否存在
$exists = $disk->exists('path/to/file.txt');

// 删除文件
$disk->delete('path/to/file.txt');

// 复制 / 移动文件
$disk->copy('source/path.txt', 'destination/path.txt');
$disk->move('source/path.txt', 'destination/path.txt');
```

### 目录操作

```php
// 创建目录
$disk->makeDirectory('path/to/directory');

// 删除目录（递归删除目录下所有文件）
$disk->deleteDirectory('path/to/directory');

// 列出目录内容
foreach ($disk->listContents('path/to/directory') as $item) {
    $item->path();      // 路径
    $item->type();      // 类型：file / dir
    $item->lastModified();
    $item->fileSize();
}
```

### 文件元信息

```php
$mimeType     = $disk->mimeType('path/to/file.txt');
$lastModified = $disk->lastModified('path/to/file.txt');
$fileSize     = $disk->fileSize('path/to/file.txt');
```

### 可见性（ACL）管理

可见性映射关系：

| Flysystem 可见性 | OSS ACL |
| --- | --- |
| `public` | `public-read` |
| `private` | `private` |

```php
use Illuminate\Contracts\Filesystem\Visibility;

// 设置可见性
$disk->setVisibility('path/to/file.txt', Visibility::PRIVATE);

// 获取可见性
$visibility = $disk->getVisibility('path/to/file.txt');
```

### 获取文件 URL

当 `url` 配置项被设置时，所有生成的 URL 均以该自定义域名为前缀：

```php
$url = $disk->url('path/to/file.txt');
// 返回: https://cdn.example.com/path/to/file.txt
```

当 `url` 未设置时：
- **public** 可见性的文件返回 OSS 默认的访问地址（协议根据 `ssl` 配置自动判断）
- **private** 可见性的文件自动返回 5 分钟有效的临时签名 URL

### 临时签名 URL

可为任意文件生成指定有效期的临时访问 URL，适用于私有文件的临时授权访问：

```php
use Carbon\Carbon;

// 生成 10 分钟有效的临时 URL
$tempUrl = $disk->temporaryUrl('path/to/private-file.txt', Carbon::now()->addMinutes(10));
```

### 获取 OSS 客户端

如需调用阿里云 OSS SDK 的高级功能，可直接获取底层的 `OssClient` 实例：

```php
$client = $disk->getClient();

// 示例：分片上传大文件
$client->multiuploadFile('your-bucket', 'large-file.zip', '/path/to/local/large-file.zip');
```

### 上传时的扩展选项

通过 `visibility` 配置或上传参数，可在写入时指定 OSS 特有的选项（如服务端加密、存储类型、标签等）：

```php
use Illuminate\Contracts\Filesystem\Visibility;
use League\Flysystem\Visibility as FlysystemVisibility;

// 设置存储类型为低频访问
$disk->put('file.txt', 'contents', [
    'x-oss-storage-class' => 'IA',
]);

// 设置服务端加密
$disk->put('secret.txt', 'contents', [
    'x-oss-server-side-encryption' => 'AES256',
]);

// 设置对象标签
$disk->put('file.txt', 'contents', [
    'x-oss-tagging' => 'key1=value1&key2=value2',
]);
```

支持的 OSS 特有选项：

| 选项 | 说明 |
| --- | --- |
| `Cache-Control` | HTTP 缓存控制头 |
| `Content-Disposition` | 内容处置头 |
| `Content-Encoding` | 内容编码 |
| `Content-MD5` | 内容 MD5 校验值 |
| `Content-Length` | 内容长度 |
| `x-oss-forbid-overwrite` | 是否禁止覆盖同名对象 |
| `x-oss-server-side-encryption` | 服务端加密方式 |
| `x-oss-server-side-data-encryption` | 服务端数据加密方式 |
| `x-oss-server-side-encryption-key-id` | KMS 加密密钥 ID |
| `x-oss-object-acl` | 对象 ACL |
| `x-oss-storage-class` | 存储类型（Standard / IA / Archive 等） |
| `x-oss-tagging` | 对象标签 |

## 贡献

欢迎提交 Issue 和 Pull Request。提交代码前请确保通过代码风格检查：

```bash
# 检查代码风格（不修复）
composer check-style

# 自动修复代码风格
composer fix-style
```

## License

[MIT License](LICENSE) (c) Larva Tech
