# aspnetcore两种上传图片(文件)的方式

**发布日期:** 2022-11-16  
**原文链接:** https://www.cnblogs.com/morec/p/16897168.html  
**标签:** 无

---

aspnetcore上传图片也就是上传文件有两种方式，一种是通过form-data，一种是binary。

先介绍第一种form-data:

该方式需要显示指定一个IFormFile类型，该组件会动态通过打开一个windows窗口选择文件 及图片。

![](./images/aspnetcore两种上传图片(文件)的方式/image_1.png)

postman演示如上，代码如下：

 [HttpPost]
 [AllowAnonymous]
 public IActionResult UploadFileByForm(IFormFile formFile)
 {
 var file = formFile;
 if (file == null)
 return JsonContent(new { status = "error" }.ToJson());

 string path = $"/Upload/{Guid.NewGuid().ToString("N")}/{file.FileName}";
 string physicPath = GetAbsolutePath($"~{path}");
 string dir = Path.GetDirectoryName(physicPath);
 if (!Directory.Exists(dir))
 Directory.CreateDirectory(dir);
 using (FileStream fs = new FileStream(physicPath, FileMode.Create))
 {
 file.CopyTo(fs);
 }

 string url = $"{_configuration["WebRootUrl"]}{path}";
 var res = new
 {
 name = file.FileName,
 status = "done",
 thumbUrl = url,
 url = url
 };

 return JsonContent(res.ToJson());
 }

第二种通过binary

![](./images/aspnetcore两种上传图片(文件)的方式/image_2.png)

同样是打开windows窗口选择文件，不过这里不需要参数显式指定IFormFile

看代码就明白了,这里通过请求上下文的body来读取字节流，如果有提示不能同步读取流的话，需要指定 services.Configure<KestrelServerOptions>(x => x.AllowSynchronousIO = true)；现在都不用iis，所以不介绍了。

 /// <summary>
 /// 上传图片 binary模式
 /// </summary>
 /// <returns></returns>
 [HttpPost]
 [AllowAnonymous]
 public IActionResult UploadImage()
 {
 var request = HttpContext.Request;
 var stream = new BinaryReader(request.Body);
 var body = stream.ReadBytes(1024*1024*5);
 return File(body, @"image/jpeg");

 }

aspnetcore7提供了如下的方式，通过指定本地文件绝对地址来读取字节流，应该算是第二种方式吧。

代码如下：参考：[ASP.NET Core 7.0 的新增功能 | Microsoft Learn](https://learn.microsoft.com/zh-cn/aspnet/core/release-notes/aspnetcore-7.0?view=aspnetcore-7.0) 需要引入包SixLabors.ImageSharp、SixLabors.ImageSharp.Formats.Jpeg

using SixLabors.ImageSharp;
using SixLabors.ImageSharp.Formats.Jpeg;
using SixLabors.ImageSharp.Processing;

var builder = WebApplication.CreateBuilder(args);

var app = builder.Build();

app.MapGet("/process-image/{strImage}", (string strImage, HttpContext http, CancellationToken token) =>
{
 http.Response.Headers.CacheControl = $"public,max-age={TimeSpan.FromHours(24).TotalSeconds}";
 return Results.Stream(stream => ResizeImageAsync(strImage, stream, token), "image/jpeg");
});

async Task ResizeImageAsync(string strImage, Stream stream, CancellationToken token)
{
 var strPath = $"wwwroot/img/{strImage}";
 using var image = await Image.LoadAsync(strPath, token);
 int width = image.Width / 2;
 int height = image.Height / 2;
 image.Mutate(x =>x.Resize(width, height));
 await image.SaveAsync(stream, JpegFormat.Instance, cancellationToken: token);
}

总结:上传文件或图片没有什么难度，只不过注意输出输入的格式就行了。如果需要多文件的话IFormFileCollection就行了， 这里不做介绍，可以看看微软的官方文档。

---

*本文档由博客园下载器自动生成*
