# 设计模式之装饰器-AOP

**发布日期:** 2020-10-08  
**原文链接:** https://www.cnblogs.com/morec/p/13781122.html  
**标签:** 无

---

HelloWorld简单例子如下：此例子好好体会下继承 is a和组合 has a的异同。

```csharp
using System;
using System.Runtime.InteropServices;

namespace TestEnviroment
{
 class Program
 {
 static void Main(string[] args)
 {
 BaseClass ins = new Ins();
 ins = new Before(ins);
 ins = new After(ins);
 ins.Do();
 Console.ReadLine();
 }
 }
 public abstract class BaseClass
 {
 public abstract void Do();
 }
 public class Ins : BaseClass
 {
 public override void Do()
 {
 Console.Write("World"); //底层行为
 }
 }
 public abstract class BaseDecorator: BaseClass
 {
 public BaseClass baseClass;
 public BaseDecorator(BaseClass baseClass)
 {
 this.baseClass = baseClass;
 }
 }
 public class After : BaseDecorator
 {
 public After(BaseClass baseClass) : base(baseClass) { }
 public override void Do()
 {
 base.baseClass.Do();
 Console.Write(" Jack"); //扩展区
 }
 }
 public class Before: BaseDecorator
 {
 public Before(BaseClass baseClass) : base(baseClass) { }
 public override void Do()
 {
 Console.Write("Hello "); //扩展区
 base.baseClass.Do();
 }
 }
}

```

下面是综合例子：

```csharp
using System;
using System.Drawing;
using System.Runtime.InteropServices;

namespace TestEnviroment
{
 public interface IText
 {
 string Content { get; }
 }
 /// <summary>
 /// 状态接口
 /// </summary>
 public interface IState
 {
 bool Equals(IState newState);
 }
 public interface IDecorator : IText
 {
 IState State { get; set; }
 void Refresh<T>(IState newState) where T : IDecorator;
 }

 public abstract class DecoratorBase : IDecorator
 {
 protected IText target;
 public DecoratorBase(IText target)
 {
 this.target = target;
 }
 public abstract string Content { get; }
 protected IState state;
 public IState State { get => this.state; set => this.state = value; }

 public virtual void Refresh<T>(IState newState) where T : IDecorator
 {
```

 if (this.GetType() == typeof(T))
```css
 {
 if (newState == null) state = null;
```

 if (State != null && !State.Equals(newState))
```css
 {
 State = newState;
 }
 return;
 }
```

 if (target != null)
```css
 {
 ((IDecorator)target).Refresh<T>(newState);
 }
 }
 }

 public class BoldState : IState
 {
 public bool IsBold;
 public bool Equals(IState newState)
 {
 if (newState == null) return false;
 return ((BoldState)newState).IsBold == IsBold;
 }
 }

 public class ColorState : IState
 {
 public Color Color = Color.Black;
 public bool Equals(IState newState)
 {
 if (newState == null) return false;
 return ((ColorState)newState).Color == Color;
 }
 }

 /// <summary>
 /// 具体装饰类
 /// </summary>
 public class BoldDecorator : DecoratorBase
 {
 public BoldDecorator(IText target) : base(target)
 {
 base.state = new BoldState();
 }

 public override string Content
 {
```

 get
```css
 {
```

 if (((BoldState)State).IsBold)
```css
 return $"<b>{target.Content}</b>";
```

 else
```
 return target.Content;
 }
 }
 }

 /// <summary>
 /// 具体装饰类
 /// </summary>
 public class ColorDecorator : DecoratorBase
 {
 public ColorDecorator(IText target) : base(target)
 {
 base.state = new ColorState();
 }
 public override string Content
 {
```

 get
```css
 {
 string colorName = ((ColorState)State).Color.Name;
 return $"<{colorName}>{target.Content}</{colorName}>";
 }
 }
 }
 /// <summary>
 /// 具体装饰类
 /// </summary>
 public class BlockAllDecorator : DecoratorBase
 {
 public BlockAllDecorator(IText target) : base(target) { }
 public override string Content => string.Empty;
 }
 public class TextObject : IText
 {
 public string Content => "hello";
 }

 class Program
 {
 static void Main(string[] args)
 {
 IText text = new TextObject();
 text = new BoldDecorator(text);
 text = new ColorDecorator(text);

 ColorState newColorState = new ColorState();
 newColorState.Color = Color.Red;
 IDecorator root = (IDecorator)text;
 root.Refresh<ColorDecorator>(newColorState);
 Console.WriteLine($"color text.Content={text.Content}");
 BoldState newBoldState = new BoldState();
 newBoldState.IsBold = true;
 root.Refresh<BoldDecorator>(newBoldState);
 Console.WriteLine($"bold text.Content={text.Content}");

 text = new BlockAllDecorator(text);
 Console.WriteLine($"text.Content={text.Content}");
 Console.ReadLine();
 }
 }
}

```

---

*本文档由博客园下载器自动生成*
