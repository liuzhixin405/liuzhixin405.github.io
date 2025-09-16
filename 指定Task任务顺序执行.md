# 指定Task任务顺序执行

**发布日期:** 2022-03-13  
**原文链接:** https://www.cnblogs.com/morec/p/16000480.html  
**标签:** 无

---

经常听到说线程池这个东西，凭印象写了个这么简单的例子。

```csharp
CusTRun方法要不要await，取决于要不要作为后台任务。
任务可指定数量，线程参数可共享全，顺序可控，可继续改进。

```
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

namespace ConsoleApp1
{
 internal class Program
 {
 static AsyncLocal<int> asyncObj = new AsyncLocal<int>();
 static async Task Main(string[] args)
 {
 await CusTRun(100, async () =>
 {
 //TODO:业务代码
 await Task.Delay(1000);
 Console.WriteLine(asyncObj.Value);
 Console.WriteLine("ManagedThreadId=" + Thread.CurrentThread.ManagedThreadId);
 }, default);
 Console.WriteLine("Hello World!");
 Console.ReadKey();
 }

 static async Task CusTRun(int count, Func<Task> func, CancellationToken cancellationToken)
 {
 ConcurrentDictionary<int, Task> taskDic = new ConcurrentDictionary<int, Task>();
 for (int i = 0; i < count; i++)
 {
 asyncObj.Value = i;
 if (taskDic.Values.Count(t => t.Status != TaskStatus.RanToCompletion) >= 5)
 {
 taskDic = (ConcurrentDictionary<int, Task>)(taskDic.OrderBy(x => x.Key));
 Task.WaitAny(taskDic.Values.ToArray());
 taskDic.Values.Where(t => t.Status != TaskStatus.RanToCompletion).ToList();
 }
 else
 {
 taskDic.TryAdd(i, Task.Run(func, cancellationToken));
 }
 await Task.Delay(1000);
 }
 }
 }
}

---

*本文档由博客园下载器自动生成*
