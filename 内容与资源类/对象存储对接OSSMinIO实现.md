# 对象存储对接OSSMinIO实现

## 1. 功能定义

对象存储对接要解决的问题是：

- 文件不再存本地磁盘，而是存到统一的对象存储服务里

这通常是文件系统从“学习版”走向“正式版”的关键一步。

常见对象存储包括：

- `阿里云 OSS`
- `腾讯云 COS`
- `AWS S3`
- `MinIO`

其中：

- `MinIO` 很适合本地或私有化部署
- `OSS` 更适合云上正式项目

## 2. 为什么要接对象存储

因为本地文件存储虽然简单，但很快会遇到这些问题：

- 多机部署文件不同步
- 容器重启文件可能丢失
- 扩容困难
- 带宽和访问压力都在应用服务器上

而对象存储能更好地解决这些问题：

- 存储独立
- 扩展方便
- 访问能力更稳定
- 能更容易挂 CDN

## 3. 核心设计

接对象存储时，最重要的设计思想是：

1. 上层业务不要感知具体存储介质
2. 本地存储和对象存储都实现同一个接口
3. 变化只收敛在“文件写入方式”这一层

也就是说，真正应该抽象的是：

```java
public interface FileStorageService {
    FileUploadVO upload(MultipartFile file, String bizType);
}
```

## 4. MinIO 和 OSS 的共性

无论是 `MinIO` 还是 `OSS`，上传流程本质都差不多：

1. 校验文件
2. 生成对象名
3. 上传到指定 bucket
4. 获取访问路径
5. 保存文件元数据

真正差异主要在 SDK 调用方式。

## 5. MinIO 核心实现

## 5.1 配置示例

```json
{
  "endpoint": "http://127.0.0.1:9000",
  "accessKey": "minioadmin",
  "secretKey": "minioadmin",
  "bucketName": "upload"
}
```

## 5.2 MinIO 上传核心代码

```java
@Service
public class MinioFileStorageServiceImpl implements FileStorageService {

    private final MinioClient minioClient;
    private final FileRecordMapper fileRecordMapper;

    @Value("${file.minio.bucket-name}")
    private String bucketName;

    @Value("${file.minio.public-url-prefix}")
    private String publicUrlPrefix;

    public MinioFileStorageServiceImpl(
            MinioClient minioClient,
            FileRecordMapper fileRecordMapper) {
        this.minioClient = minioClient;
        this.fileRecordMapper = fileRecordMapper;
    }

    @Override
    public FileUploadVO upload(MultipartFile file, String bizType) {
        String originName = file.getOriginalFilename();
        String suffix = getSuffix(originName);
        String objectName = LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE)
                + "/"
                + UUID.randomUUID().toString().replace("-", "")
                + "."
                + suffix;

        try (InputStream inputStream = file.getInputStream()) {
            minioClient.putObject(
                    PutObjectArgs.builder()
                            .bucket(bucketName)
                            .object(objectName)
                            .stream(inputStream, file.getSize(), -1)
                            .contentType(file.getContentType())
                            .build()
            );
        } catch (Exception ex) {
            throw new BizException("上传到MinIO失败");
        }

        String url = publicUrlPrefix + "/" + objectName;

        FileRecord record = new FileRecord();
        record.setOriginName(originName);
        record.setStorageName(objectName);
        record.setContentType(file.getContentType());
        record.setSize(file.getSize());
        record.setStoragePath(url);
        record.setBizType(bizType);
        record.setUploadUserId(UserContext.get().getUserId());
        fileRecordMapper.insert(record);

        FileUploadVO vo = new FileUploadVO();
        vo.setFileId(record.getId());
        vo.setFileName(originName);
        vo.setUrl(url);
        vo.setSize(file.getSize());
        return vo;
    }

    private String getSuffix(String fileName) {
        if (!StringUtils.hasText(fileName) || !fileName.contains(".")) {
            throw new BizException("文件名不合法");
        }
        return fileName.substring(fileName.lastIndexOf('.') + 1);
    }
}
```

## 6. OSS 核心实现思路

如果换成 `OSS`，整体流程几乎不变，变化的主要是 SDK 调用：

```java
ossClient.putObject(bucketName, objectName, inputStream);
```

也就是说，上层业务不应该因为你从本地切换到 OSS 而发生很大变化。  
变化应该只在 `FileStorageService` 的实现类里。

## 7. 为什么对象名也要自己生成

对象存储虽然不会像本地文件夹那样直观重名覆盖，但对象名仍然建议由系统统一生成。  
原因和本地文件名类似：

1. 避免冲突
2. 保持目录规范
3. 避免原始文件名带来安全和兼容问题

## 8. 下载和预览怎么做

接入对象存储后，下载和预览常见有两种方式：

1. 直接返回公开 URL
2. 生成临时签名 URL

如果文件是公开资源，例如商品图、文章封面，通常可以直接访问。  
如果文件是私有资源，例如合同、内部附件，通常更推荐签名 URL。

## 9. 常见坑点

### 9.1 上层业务写死本地路径逻辑

一旦换成对象存储，改造成本会很大。

### 9.2 上传成功了，但文件表没记录

后续业务无法引用和管理。

### 9.3 对私有文件使用永久公开地址

会带来权限和泄露风险。

## 10. 总结

对象存储对接的工程重点，不是 SDK 怎么调，而是把文件存储抽象成统一接口，让“本地存储”和“对象存储”都只是底层实现差异。

一旦这层抽象做对了，系统就能很自然地从学习阶段的本地文件，演进到正式项目里的 `MinIO/OSS` 架构。
