# C#面对抽象编程第一讲

**发布日期:** 2021-12-19  
**原文链接:** https://www.cnblogs.com/morec/p/15708366.html  
**标签:** 无

---

闲话不多说，面向对象编程是高级语言的一个特点，但是把它概括成面向抽象更容易直击灵魂，经过了菜鸟大家都要面对的是不要写这么菜的代码了。

上例子，这应该是大家都很熟悉耳熟能详的代码， so easy。

```

```
 1 using System;
 2 using System.Diagnostics;
 3 
 4 namespace ConsoleApp1
 5 {
 6 internal class Program
 7 {
 8 static void Main(string[] args)
 9 {
10 Demo demo = new Demo();
11 demo.PrintData();
12 }
13 }
14 internal class Demo
15 {
16 private const int Max = 10;
17 private int[] Generate()
18 {
19 Random rnd = new Random();
20 int[] data = new int[Max];
21 for (int i = 0; i < Max; i++)
22 {
23 data[i] = rnd.Next() % 1023;
24 }
25 return data;
26 }
27 public void PrintData()
28 {
29 string result = string.Join(",", Array.ConvertAll<int, string>(Generate(), n => Convert.ToString(n)));
30 Trace.WriteLine(result);
31 Console.WriteLine(result);
32 }
33 }
34 }
```

```
我们看看它的脆弱性在哪里？ 

•随机数发生器可能变成从数据库提取的一批商品数量或从多个下游企业发来的报文中筛选出来的RFID过检（通过检查）集装箱件数。 

•用户可能还需要把信息写入数据库、写入文件，或者觉得有些Viewer显示没什么用处，只要一种Output窗口就可以了。

归纳一下，这种混合方式的程序相对脆弱，因为会导致变化的因素比较多，按照我们之前设计模式的经验，这时候应该抽象对象，这里我们先把V和M抽象出来，然后在C中组合它们：

```

```
using System;
using System.Collections.Generic;
using System.Diagnostics;

namespace ConsoleApp2
{
 internal class Program
 {
 static void Main(string[] args)
 {
 Controller controller = new Controller();
 controller.Model = new Randomizer();
 controller += new TraceView();
 controller += new ConsoleView();
 controller.Process();
 }
 }
 public class Controller
 {
 private IList<IView> views = new List<IView>();

 private IModel model;
 public virtual IModel Model { get => model; set => model = value; }

 public void Process()
 {
 if (views.Count == 0) return;
 string result = string.Join(",", Array.ConvertAll<int, string>(model.Data, n => Convert.ToString(n)));
 foreach (var view in views)
 {
 view.Print(result);
 }
 }
 public static Controller operator +(Controller controller, IView view)
 {
 if (view == null) throw new ArgumentNullException("view");
 controller.views.Add(view);
 return controller;
 }
 public static Controller operator -(Controller controller, IView view)
 {
 if (view == null) throw new ArgumentNullException("view");
 controller.views.Remove(view);
 return controller;
 }
 }

 class Randomizer : IModel
 {
 public int[] Data
 {
 get
 {
 Random random = new Random();
 int[] result = new int[10];
 for (int i = 0; i < result.Length; i++)
 {
 result[i] = random.Next() % 1023;
 }
 return result;
 }
 }
 }
 class ConsoleView : IView
 {
 public void Print(string data)
 {
 Console.WriteLine(data);
 }
 }
 
 class TraceView : IView
 {
 public void Print(string data)
 {
 Trace.WriteLine(data);
 }
 }

 public interface IView
 {
 void Print(string data);
 }

 public interface IModel
 {
 int[] Data { get; }
 }
}
```

```

按照上面的介绍，主动方式MVC需要一个观察者对M保持关注。这里我们简单采用.NET的事件机制来充当这个角色，而本应从V发起的重新访问M获得新数据的过程也简化为事件参数，在触发事件的同时一并提供，代码如下所示：

```

```
using System;
using System.Diagnostics;

namespace ConsoleApp3
{
 internal class Program
 {
 static void Main(string[] args)
 {
 Controller controller = new Controller();
 IModel model = new Model();
　　　　　　　controller.Model = model;
 controller += new TraceView();
 controller += new ConsoleView();
 // 后续Model修改时，不经过Controller，而是经由观察者完成View的变化 model[1] = 2000; // 第一次修改（修改的内容按照之前的随机数计算不会出现） model[3] = -100; // 第二次修改（修改的内容按照之前的随机数计算不会出现）

 }
 }

 internal class Controller
 {
 private IModel model;
 public virtual IModel Model { get => model; set => model = value; }
 public static Controller operator +(Controller controller, IView view)
 {
 if (view == null) throw new ArgumentNullException(nameof(view));
 controller.Model.DataChanged += view.Handler;
 return controller;
 }
 public static Controller operator -(Controller controller, IView view)
 {
 if (view == null) throw new ArgumentNullException(nameof(view));
 controller.Model.DataChanged -= view.Handler;
 return controller;
 }
 }

 internal class Model : IModel
 {
 public event EventHandler<ModelEventArgs> DataChanged;
 private int[] data;
 public int this[int index]
 {
 get => data[index];
 set
 {
 this.data[index] = value;
 DataChanged?.Invoke(this, new ModelEventArgs(data));
 }
 }
 public Model()
 {
 Random rnd = new Random();
 data = new int[10];
 for (int i = 0; i < data.Length; i++)
 {
 data[i] = rnd.Next() % 1023;
 }
 }
 }
 internal abstract class ViewBase : IView
 {
 public abstract void Print(string data);
 public virtual void OnDataChanged(object sender, ModelEventArgs args)
 {
 Print(args.Context);
 }
 public virtual EventHandler<ModelEventArgs> Handler
 {
 get => this.OnDataChanged;
 }
 }
 internal class TraceView : ViewBase
 {
 public override void Print(string data)
 {
 Trace.WriteLine(data);
 }

 }
 internal class ConsoleView : ViewBase
 {
 public override void Print(string data)
 {
 Console.WriteLine(data);
 }
 }
 internal class ModelEventArgs : EventArgs
 {
 private string content;
 public string Context { get => this.content; }

 public ModelEventArgs(int[] data)
 {
 content = string.Join(",", Array.ConvertAll<int, string>(data, n => Convert.ToString(n)));
 }
 }
 internal interface IModel
 {
 event EventHandler<ModelEventArgs> DataChanged;
 int this[int index] { get; set; }
 }
 internal interface IView
 {
 EventHandler<ModelEventArgs> Handler { get; }
 void Print(string data);
 }

}
```

```
从上面的示例不难看出，相对被动方式的MVC而言，采用.NET事件方式实现主动方式有下述优势： 

•结构更加松散耦合，M/V之间可以没有直接的依赖关系，组装过程可以由C完成，M/V之间的观察者仅经由.NET标准事件和委托进行交互。 

 •不用设计独立的观察者对象。 

•由于C不需参与M数据变更后实际的交互过程，因此C也无需设计用来保存V的容器。 

•如果EventArgs设计合理的话，可以更自由地与其他产品或第三方对象体系进行集成。

注： 摘录于 王翔 《设计模式 基于c#的工程化实现及扩展》 摘自其中一节，2008出版的版本。新的版本改动很大，没有那么喜欢了。推荐有兴趣的自行剁手，一顿饭的钱。这本书的质量本人觉得非常高，虽然2008的年出版，但是放到现在再结合现代的技术发展更加印证了这本书是经得起时间考验的。

---

*本文档由博客园下载器自动生成*
