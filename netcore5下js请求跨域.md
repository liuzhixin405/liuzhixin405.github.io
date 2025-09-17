# netcore5下js请求跨域

**发布日期:** 2021-01-25  
**原文链接:** https://www.cnblogs.com/morec/p/14325240.html  
**标签:** 无

---

后端代码如下:

```csharp
using Microsoft.AspNetCore.Mvc;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

// For more information on enabling Web API for empty projects, visit https://go.microsoft.com/fwlink/?LinkID=397860

namespace WebApplication.Controllers
{
```

 [Route("api/[controller]")]
 [ApiController]
```csharp
 public class ValuesController : ControllerBase
 {
 // GET: api/<ValuesController>
```

 [HttpGet]
```csharp
 public IEnumerable<string> Get()
 {
 return new string[] { "value1", "value2" };
 }

 // GET api/<ValuesController>/5
 [HttpGet("{id}")]
 public string Get(int id)
 {
 return "value";
 }

 // POST api/<ValuesController>
```

 [HttpPost]
```csharp
 public void Post([FromForm] string value) <span style="color: rgba(255, 0, 0, 1)">//FromBody须改成FromFrom要不然跨域还是会报错
 {
 }

 </span>// PUT api/<ValuesController>/5
 [HttpPut("{id}")]
 public void Put(int id, [FromBody] string value)
 {
 }

 // DELETE api/<ValuesController>/5
 [HttpDelete("{id}")]
 public void Delete(int id)
 {
 }
 }
}

using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.HttpsPolicy;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace WebApplication
{
 public class Startup
 {
 readonly string MyAllowSpecificOrigins = "_myAllowSpecificOrigins";

 public Startup(IConfiguration configuration)
 {
 Configuration = configuration;
 }

 public IConfiguration Configuration { get; }

 // This method gets called by the runtime. Use this method to add services to the container.
 public void ConfigureServices(IServiceCollection services)
 {
```

 services.AddCors(options =>
```css
 {
 <span style="color: rgba(255, 0, 0, 1)"> options.AddPolicy(name: MyAllowSpecificOrigins,
```

 builder </span>=>
```css
 {
 builder.WithMethods("get", "post").AllowAnyOrigin();//允许任何来源的主机访问
<span style="color: rgba(255, 0, 0, 1)"> 
 });
 });

 services.AddControllersWithViews();
 }

 </span>// This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
 public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
 {
```

 if (env.IsDevelopment())
```css
 {
 app.UseDeveloperExceptionPage();
 }
```

 else
```css
 {
 app.UseExceptionHandler("/Home/Error");
 // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
 app.UseHsts();
 }
 app.UseHttpsRedirection();
 app.UseStaticFiles();

 app.UseRouting();

 app.UseAuthorization();
 <span style="color: rgba(255, 0, 0, 1)"> app.UseCors(MyAllowSpecificOrigins);
```

 app.UseEndpoints(endpoints </span>=>
```css
 {
```

 endpoints.MapControllerRoute(
 name: "default",
```css
 pattern: "{controller=Home}/{action=Index}/{id?}");
 });
 }
 }
}

```

前端就是一个测试demo，代码如下：

```html
<HTML>
<HEAD></HEAD>
<script charset="utf-8">
function createCORSRequest(method,url){
var xhr = new XMLHttpRequest();
if("withCredentials" in xhr){
xhr.open(method,url,true);
}else if(typeof XDomainRequest !="undifined"){
xhr = new XDomainRequest();
xhr.open(method,url);
}else
{
xhr=null;
}
return xhr;
}

//get请求

<!-- var request = createCORSRequest("get","https://localhost:44351/api/values"); -->
<!-- if(request){ -->

<!-- request.onload=function(){ -->
<!-- alert(request.responseText); -->
<!-- }; -->
<!-- request.send(); -->
<!-- } -->
//post请求
<!-- var request = createCORSRequest("post","https://localhost:44351/api/values"); -->
<!-- if(request){ -->

 <!-- //request.withCredentials = "true"; //<span style="color: rgba(255, 0, 0, 1)">客户端设置跨域即可，前端不能设置，如果设置反而报错 --></span>
<!-- //request.setRequestHeader('Access-Control-Allow-Origin','*'); -->
<!-- //request.setRequestHeader('Content-Type','application/json'); -->
<!-- //request.setRequestHeader('Origin','localhost:44351'); -->
<!-- request.onload=function(){ -->
<!-- alert(request.responseText); -->
<!-- }; -->
<!-- request.send("hello world!"); -->
<!-- } -->
</script>
<BODY>

<div id="i">
```

test
```xml
</div>
</BODY>
</HTML>

```

---

*本文档由博客园下载器自动生成*
