# The instance of entity type 'Model' cannot be tracked because another instance with the same key value for {'Id'} is already being tracked.

**发布日期:** 2022-11-22  
**原文链接:** https://www.cnblogs.com/morec/p/16916119.html  
**标签:** 无

---

The instance of entity type 'Model' cannot be tracked because another instance with the same key value for {'Id'} is already being tracked. When attaching existing entities, ensure that only one entity instance with a given key value is attached. Consider using 'DbContextOptionsBuilder.EnableSensitiveDataLogging' to see the conflicting key values.

新增数据到数据库后再在同一个方法或请求更新某些字段报的错误，大概的意思就是=>

无法跟踪实体类型“Model”的实例，因为已经在跟踪具有相同键值{'Id'}的另一个实例。在附加现有实体时，请确保只附加一个具有给定键值的实体实例。考虑使用' dbcontexttoptionsbuilder。EnableSensitiveDataLogging'查看冲突的键值。

我的做法是既然同一个链接，那么我就重新弄一个连接出来，因为我这efcore组件封装了，取context有点麻烦。所以选择别的办法。

源代码如下,报错的地方就是@2这里的更新引发错误：

 [HttpPost]
 public async Task SaveData(PushMessageUpdateRequest data)
 {
 if (data == null)
 throw new BusException($"{nameof(PushMessageUpdateRequest)} is null", ErrorCodeDefine.ParameterIsNull);
 if (data.Id.IsNullOrEmpty())
 {
 InitEntity(data);
 foreach (var content in data.Contents)
 {
 InitEntity(content);
 }
 await _pushMessageBusiness.AddDataAsync(data);
 var res = await _soptBusiness.FirstOrDefault(x => x.BusinessCode == "tttt").GetCustomerPushInfo();
 var groups = res.Where(x => x.Value != null).GroupBy(x => x.Value).ToDictionary(x => x.Key, y => y.Select(x => x.Key).ToList());
 List<bool> pushedFlag = new List<bool> { };
 foreach (var item in groups)
 {
 var pushContent = data.Contents.Where(x => x.Lang == item.Key.ToLower()).FirstOrDefault();
 if (pushContent != null && pushContent.Content.IsNotNullOrEmpty())
 {
 if (data.PushTime == null)
 {
 var result = await _pushClient.SendPush(new Klickl.Push.Sdk.Model.SendPushRequest
 {
 Type = Klickl.Push.Sdk.Model.PushType.Google,
 Ids = item.Value,
 NoticeContent = pushContent?.Content,
 Title = pushContent?.Title,
 IsApnsProduction = false
 });
 pushedFlag.Add(result);
 }
 else
 {
 JobHelper.SetDelayJob(async () =>
 {
 var result = await _pushClient.SendPush(new Klickl.Push.Sdk.Model.SendPushRequest
 {
 Type = Klickl.Push.Sdk.Model.PushType.Google,
 Ids = item.Value,
 NoticeContent = pushContent?.Content,
 Title = pushContent?.Title,
 IsApnsProduction = false
 });
 pushedFlag.Add(result);
 }, data.PushTime.Value.Subtract(DateTime.Now));
 }

 }
 }

 data.SuccessCount= pushedFlag.Where(x=>x==true).Count(); //@1
 data.PushCount = groups.Count;
 await _pushMessageBusiness.UpdateDataAsync(data); //@2
 }
 else
 {
 await _pushMessageBusiness.UpdateDataAsync(data);
 }
 }

首先想的是_pushMessageBusiness这个business重新构造一个，像这样@3新增的business，@4就是为了报错加上去的。测试发现还是报一样的错误。

 private readonly IPushMessageBusiness _pushMessageBusiness; //@3
 private readonly PushClient _pushClient;
 private readonly IEnumerable<ISpotBusiness> _soptBusiness;
 private readonly IAutoPushMessageBusiness _autoPushMessageBusiness;
 private record SaveDataChangeEventArgs(string Id, int SucessCount, int PushCount); 
 private EventHandler<SaveDataChangeEventArgs> SaveDataChangeEventEventHandler;
 
 private readonly IPushMessageBusiness _updataPushMessageBusiness; @4
 private IServiceScopeFactory _scopeFactory;

因为_pushMessageBusiness里面有idbaccessor，及数据库的底层链接，所以上面注入idbaccessor是没问题，可以把bug解决掉，但是这样取原始远不如获取business安全和稳定。

![](./images/The instance of entity type 'Model' cannot be tracked because another instance with the same/image_1.png)

解决办法如下：

 data.SuccessCount= pushedFlag.Where(x=>x==true).Count();
 data.PushCount = groups.Count;
 await _pushMessageBusiness.UpdateDataAsync(data);

上面代码最好独立出来，我通过事件来更新。

 if (groups.Count > 0 && SaveDataChangeEventEventHandler != null)
 SaveDataChangeEventEventHandler(this, new SaveDataChangeEventArgs(data.Id, pushedFlag.Where(x => x == true).Count(), groups.Count));

事件放到构造函数，主要是新注入了一个scope工厂，这个也是关键代码，生成了的business和原来是不一样的。代码如下：

 public PushMessageController(IPushMessageBusiness pushMessageBusiness,
 IPushClientFactory factory,
 IEnumerable<ISpotBusiness> soptBusiness, IAutoPushMessageBusiness autoPushMessageBusiness,
 IServiceScopeFactory scopeFactory)
 {
 _pushMessageBusiness = pushMessageBusiness;
 _pushClient = factory.Create("ttttt");
 _soptBusiness = soptBusiness;
 _autoPushMessageBusiness = autoPushMessageBusiness;
 _scopeFactory = scopeFactory;
 SaveDataChangeEventEventHandler += (obj, args) =>
 {
 AsyncHelper.RunSync(async () =>
 {
 using(var scope = _scopeFactory.CreateAsyncScope())
 {
 var _bus = scope.ServiceProvider.GetRequiredService<IPushMessageBusiness>();
 var entity = await _bus.GetEntityAsync(x => x.Id.Equals(args.Id));
 if (entity == null) { return; }
 entity.SuccessCount = args.SucessCount;
 entity.PushCount = args.PushCount;
 await _bus.UpdateAsync(entity);
 }
 
 });

 };

 }
 
 private readonly IPushMessageBusiness _pushMessageBusiness;
 private readonly PushClient _pushClient;
 private readonly IEnumerable<ISpotBusiness> _soptBusiness;
 private readonly IAutoPushMessageBusiness _autoPushMessageBusiness;
 private record SaveDataChangeEventArgs(string Id, int SucessCount, int PushCount); 
 private EventHandler<SaveDataChangeEventArgs> SaveDataChangeEventEventHandler;
 
 private IServiceScopeFactory _scopeFactory;

 public static class AsyncHelper
 {
 private static readonly TaskFactory _myTaskFactory =
 new TaskFactory(CancellationToken.None, TaskCreationOptions.None, TaskContinuationOptions.None, TaskScheduler.Default);

 /// <summary>
 /// 同步执行
 /// </summary>
 /// <param name="func">任务</param>
 public static void RunSync(Func<Task> func)
 {
 _myTaskFactory.StartNew(func).Unwrap().ConfigureAwait(false).GetAwaiter().GetResult();
 }

 /// <summary>
 /// 同步执行
 /// </summary>
 /// <typeparam name="TResult">返回类型</typeparam>
 /// <param name="func">任务</param>
 /// <returns></returns>
 public static TResult RunSync<TResult>(Func<Task<TResult>> func)
 {
 return _myTaskFactory.StartNew(func).Unwrap().ConfigureAwait(false).GetAwaiter().GetResult();
 }
 }

---

*本文档由博客园下载器自动生成*
