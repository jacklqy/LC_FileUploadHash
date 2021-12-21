# LC_FileUploadHash
文件hash、上传、实现文件上传重复验证。这里为大家介绍对文件上传进行哈希校验---同一文件只允许上传一次到服务器，其他的上传只要指向文件地址即可。

首先需要设计一张用于文件管理的业务表(对应数据库实体表)：
```
public class SysFile
{
    public int Id { get; set; }
 
    /// <summary>
    /// 文件名称
    /// </summary>
    [StringLength(100)]
    public string FileName { get; set; }
 
    /// <summary>
    /// 文件存放路径
    /// </summary>
    [StringLength(200)]
    public string FilePath { get; set; }
 
    /// <summary>
    /// 存放目录
    /// </summary>
    [StringLength(100)]
    public string FileDirectory { get; set; }
 
    /// <summary>
    /// 文件大小
    /// </summary>
    public int FileSize { get; set; }
 
    /// <summary>
    /// 哈希值 SHa256
    /// </summary>
    [StringLength(100)]
    public string Hash { get; set; }
 
    /// <summary>
    /// 关联表
    /// </summary>
    public string LinkName { get; set; }
 
    /// <summary>
    /// 关联ID
    /// </summary>
    public int LinkID { get; set; }
}
```
UploadFileDto：
```
public class UploadFileDto
{
    public string FileHash { get; set; }

    public string FileName { get; set; }
}
```
实际项目中可以根据该表写一个领域服务，方便接口调用。

由于需要验证文件是否存在，所以需要提供查询和上传接口：
```
public async Task<List<UploadFileDto>> Check(List<UploadFileDto> input)
{
    var existFiles = await _fileRepository.GetAllListAsync(_ => input.Select(f => f.FileHash).Contains(_.Hash));
    foreach (var file in input)
    {
        var existFile = existFiles.FirstOrDefault(_ => _.Hash == file.FileHash);
        if(existFile!=null)
        {
            input.Remove(file);
        }
    }
    return input;
}

public async Task<bool> Upload(IFormFileCollection files)
{
    if (!files.Any())
        return false;

    var filePath = Path.Combine(_env.ContentRootPath, "wwwroot/Upload");
    if (!Directory.Exists(filePath)) Directory.CreateDirectory(filePath);

    foreach (var file in files)
    {
        var fileName = $"{Guid.NewGuid()}@{file.FileName}";
        var tempFilePath = Path.Combine(filePath, fileName);

        using (var fileStream = new FileStream(Path.Combine(filePath, fileName), FileMode.Create))
        {
            await file.CopyToAsync(fileStream);
        }
    }

    return true;
}
```
后台需要以下步骤：

1、对前端计算的文件哈希值进行查询，返回数据库中不存在的文件信息

2、将已存在的文件进行路径指向，此时这些文件就不需要再次上传，只要在数据库中加一条路径指向就可以了

3、对服务器不存在的文件进行逐一上传

由于文件的hash算法是在前端实现，所有后台处理的方式有多种，大家可以根据自己的业务和需求进行调整。

前端（angular）的实现，前端的实现是采用开源的js包，所以任何框架均可实现

首先安装js-sha256包：npm install js-sha256

在需要上传的模块中进行引用：  import { sha256 } from 'js-sha256';

这里使用的是primeng中的上传组件，在选择文件后进行哈希计算：
```
onSelect(event, form): void {
    if (form.files.length == 1) {
        this.uploadDto.pop();
    }
    for (const file of event.files) {
        let self = this;
        let fr = new FileReader();
        var upload = new UploadFileDto();
        upload.fileName = file.name;
        fr.readAsArrayBuffer(file);
        fr.onload = function (data: any) {
            let fi = data.target.result;
            var hash = sha256(fi);
            upload.fileHash = hash;
            self.uploadDto.push(upload);
        }
    }
}
```
点击上传：
```
myUploader(event, form): void {
  if (event.files.length == 0) {
      return;
  }

  this.uploading = true;
  this._filesUploadService.check(this.uploadDto)
      .subscribe(result => {
          this.uploadDto = result;
      });

  let input = new FormData();
  for (const file of event.files) {
      var uploadFile = this.uploadDto.find(_ => _.fileName == file.name);
      if (uploadFile) {
          input.append('files', file);
      }
  }
  this._httpClient
      .post(this.uploadUrl, input)
      .subscribe(result => {
          if (result) {
              for (const file of event.files) {
                  this.uploadedFiles.push(file);
              }
              form.clear();
              this.uploading = false;
              this.message.success('上传成功！');
          }
          else {
              this.uploading = false;
              this.message.error('上传失败！');
          }
      },
          error=>{
              this.message.error('上传失败！');
      });
}
```
其他开发人员可能对angular语法有点难懂，其实核心只有三步：

1、选择文件并对文件进行计算

2、上传计算的文件信息并获取返回信息

3、对返回的文件信息进行包装，将文件流一并传入接口


总结：以上只是对文件校验上传的简单实现，如有不足之处，还请多多赐教。
