---
title: C#版本的Hibernate-Validator参数自动验证组件
date: 2020-05-12 20:00:00
tags:
	- C#
	- .Net
	- 参数验证
	- Hibernate-Validator
	- AOP
categories:
	- 快速开发框架
	- .Net
	- AOP
---

### 一、背景

某天，我在刷掘金社区的时候，看到了一篇文章，讲如何进行参数自动验证，点进去看，是作者在介绍Hibernate-Validator这个参数验证框架。

然后我发现Java的这个框架的使用实在太`优雅`了，仅需要在对象的属性上加上各种注解（注解可以简单理解为C#中的特性，即Attribute），就可以实现参数自动校验。

于是乎，我就有了实现一个C#版本的参数验证组件的想法，梳理了一下大体思路

1. 我们需要一个AOP框架，用来自动在函数执行前后加入我们的参数验证代码，这里我用的是MrAdvice。
2. 我们需要定义验证的Attribute，用于让用户告知程序需要如何验证参数以及验证失败后的错误信息，这里我部分用了和Hibernate-Validator相同的类型，如：`NullAttribute` 用于验证参数必须为null等
3. 我们需要实现各个验证Attribute的验证逻辑，在这里程序知道要验证的参数的`参数类型`以及`参数的值`就可以判断此参数是否验证通过了。
4. 在AOP切面中，依次解析当前Model中每个属性，拿到此属性的`验证Attribute`、`参数类型`、`参数的值`，并调用验证方法获取验证结果
5. 将结果返回或者抛出`参数验证失败`异常

思路理清晰了，方案可行，开干。

### 二、代码实现

1. **新建一个类库，然后添加MrAdvice的NuGet包，项目结构如下**

   ```
   ├── AOP                       		  
   │   └── ValidateParamAttribute	// 用于解析Model的属性，并调用具体的验证逻辑
   ├── Constraints                 // 用于存放所有的验证Attribute以及每个Attribute自己的验证逻辑
   │   ├── BaseAttribute			// 基础特性，所有自定义特性都需要继承此特性，并实现验证方法
   │   ├── AssertFalseAttribute	// 验证参数必须为false
   │   ├── AssertTrueAttribute     // 验证参数必须为true
   │   ├── DigitsAttribute			// 验证参数的整数位个数和小数位个数必须符合用户的要求
   │   ├── EmailAttribute          // 验证参数必须是邮箱格式（也可以自定义验证邮箱的正则表达式）
   │   ├── FutureAttribute      	// 验证参数的值必须大于客户指定的时间
   │   ├── InnerValidAttribute     // 告知系统集合类型的参数需要进一步校验集合中每一个元素的参数
   │   ├── MaxAttribute     		// 验证参数的值必须大于用户指定的值
   │   ├── MinAttribute     		// 验证参数的值必须小于用户指定的值
   │   ├── NotBlankAttribute     	// 验证参数去掉首尾空格后不能为空（只适用于字符串）
   │   ├── NotEmptyAttribute     	// 验证参数不能为空集合
   │   ├── NotNullAttribute     	// 验证参数不能为null
   │   ├── NullAttribute     		// 验证参数必须为null
   │   ├── PastAttribute     		// 验证参数的值必须小于客户指定的时间
   │   ├── PatternAttribute     	// 验证参数必须符合指定的正则表达式
   │   ├── PhoneAttribute     		// 验证参数必须是手机号
   │   ├── RangeAttribute     		// 验证参数的值在用户指定范围内
   │   └── SizeAttribute 			// 验证集合的大小在用户指定范围内
   ├── Common						// 定义了一些公共方法，如解析参数，判断类型等
   ├── ConstraintViolationException	// 异常类，校验失败后会抛出此异常，这里我做了点个性化的东西，就是如果函数返回类型是Result类型的话，就把异常信息放到Result对象的Message字段中。
   ```

2. **在Constraints文件夹中，我们创建了一个名叫BaseAttribute的虚类，并根据验证类型抽象出了一些公共参数和行为。比如验证失败后的返回信息，分组名称以及验证方法。具体的定义如下**

   ```c#
   	   /// <summary>
       /// 基础特性，所有的自定义特性都要继承此特性
       /// </summary>
       public abstract class BaseAttribute : Attribute
       {
           /// <summary>
           /// 返回的错误信息
           /// </summary>
           internal string Message { get; set; }
   
           /// <summary>
           /// 分组，用于解决同一个参数的校验方式在不同业务中使用不同规则的问题
           /// </summary>
           internal string Group { get; set; }
   
           /// <summary>
           /// 构造函数
           /// </summary>
           /// <param name="msg">返回的错误信息</param>
           /// <param name="group">分组，用于解决同一个参数的校验方式在不同业务中使用不同规则的问题</param>
           protected BaseAttribute(string msg, string group = "")
           {
               this.Message = msg;
               this.Group = group;
           }
   
           /// <summary>
           /// 验证参数是否符合要求
           /// </summary>
           /// <param name="value">参数值</param>
           /// <param name="prop">参数类型</param>
           /// <returns>符合要求=true</returns>
           public abstract bool Validate(object value, PropertyInfo prop);
       }
   ```

   这里说明一下分组（Group）参数的意义，有时候我们会遇到这种情况，同一个model的参数在不同的函数中，需要有不同的验证方式。

   比如说有一个User对象，里面有一个Id属性，在添加用户的函数中，需要验证id属性必须为0，而在编辑用户函数中，需要验证id属性必须大于0。

   这个时候就可以在id属性上的两个验证Attribute中，分别给group参数赋值为"addUser"和"editUser"，

   然后

   在添加用户函数的校验Attribute上面，将group参数设置为"addUser"，

   在编辑用户函数的校验Attribute上面，将group参数设置为"editUser"，

   验证的时候添加用户就只会使用Id属性上group为"addUser"的验证Attribute，编辑用户就只会使用id属性上group为"editUser"的验证Attribute。这样同一个属性就可以在不同的函数中使用不同的验证Attribute了。

3. **实现验证Attribute，这里我用MaxAttribute来举例说明如何实现一个验证**

   MaxAttribute的用途是验证参数的值必须大于用户指定的值，这里就需要传入用户想要指定的值是多少，最大值是否可以包含这个值，以及我们需要判断一下当前的参数是否是数字类型。

   用户指定的数据我们用两个私有变量来保存，通过构造函数的方式传入

   判断是否数字类型的方法如下

   ```c#
           /// <summary>
           /// 判断是否是数字类型
           /// </summary>
           /// <param name="type">类型</param>
           /// <returns>判断结果，数字类型=true</returns>
           internal static bool IsNumericType(this Type type)
           {
               switch (Type.GetTypeCode(type))
               {
                   case TypeCode.Byte:
                   case TypeCode.SByte:
                   case TypeCode.UInt16:
                   case TypeCode.UInt32:
                   case TypeCode.UInt64:
                   case TypeCode.Int16:
                   case TypeCode.Int32:
                   case TypeCode.Int64:
                   case TypeCode.Decimal:
                   case TypeCode.Double:
                   case TypeCode.Single:
                       return true;
                   default:
                       return false;
               }
           }
   ```

   具体的实现代码如下

   ```c#
   	   /// <summary>
       /// 验证元素值小于等于指定的value值
       /// </summary>
       [Serializable]
       [AttributeUsage(AttributeTargets.Property, AllowMultiple = true)]
       public class MaxAttribute : BaseAttribute
       {
           /// <summary>
           /// 要比较的值
           /// </summary>
           private decimal _value;
   
           /// <summary>
           /// 是否包含传入的值
           /// </summary>
           private bool _isInclude;
   
           /// <summary>
           /// 构造函数
           /// </summary>
           /// <param name="value">要比较的值</param>
           /// <param name="msg">返回的错误信息</param>
           /// <param name="isInclude">是否包含传入的值</param>
           /// <param name="group">分组，用于解决同一个参数的校验方式在不同业务中使用不同规则</param>
           public MaxAttribute(double value, string msg, bool isInclude = false, string @group = "") : base(msg, @group)
           {
               _value = Convert.ToDecimal(value);
               _isInclude = isInclude;
           }
   
           /// <summary>
           /// 验证参数是否符合要求
           /// </summary>
           /// <param name="value">参数值</param>
           /// <param name="prop">参数类型</param>
           /// <returns>符合要求=true</returns>
           public override bool Validate(object value, PropertyInfo prop)
           {
               if (!prop.PropertyType.IsNumericType())
               {
                   return false;
               }
               decimal dec = Convert.ToDecimal(value);
               return _isInclude ? dec <= _value : dec < _value;
           }
       }
   ```

   

4. **上面的Attribute都是作用于属性上，用于告知系统要使用哪种验证Attribute。**

   **接下来，就是此组件的核心了，此特性作用于函数上，用于解析哪些参数需要验证，并获取验证结果** 

   首先，我们需要让用户告诉系统需要校验的参数名和当前校验的分组名称，这些信息通过构造函数传入

   参数名用于判断哪些参数需要校验，如果有多个参数则需要将多个参数的名称用 `-` 符号隔开传入

   分组名称用于区分不同的校验Attribute，传了此值后，组件将会只解析那些组名和此值相等的校验Attribute。

   核心代码：

   ```c#
     	 /// <summary>
       /// 参数验证
       /// </summary>
       [Serializable]
       [AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
       public class ValidateParamAttribute : Attribute, IMethodAdvice
       {
           /// <summary>
           /// 验证类型
           /// </summary>
           private ValidType _type { get; set; }
   
           /// <summary>
           /// 要验证的参数，不传默认为验证当前方法入参的所有参数
           /// 格式：参数名称1-参数名称2(用-将参数名称隔开)
           /// 例：Regist(RegistModel1 reg1,RegistModel2 reg2)，若需要同时校验reg1，reg2两个参数的值，则传入：reg1-reg2;
           /// </summary>
           private string _paramNames { get; }
   
           /// <summary>
           /// 组名，仅仅校验参数的特性中和当前组名一致的特性
           /// </summary>
           private string _group { get; }
   
           /// <summary>
           /// 构造函数
           /// </summary>
           /// <param name="type">验证类型</param>
           /// <param name="paramName">要验证的参数，不传默认为验证当前方法入参的所有参数</param>
           public ValidateParamAttribute(string paramName = "", string group = "", ValidType type = ValidType.Fast)
           {
               _type = type;
               _paramNames = paramName;
               _group = group;
           }
   
           /// <summary>
           /// Implements advice logic.
           /// Usually, advice must invoke context.Proceed()
           /// </summary>
           /// <param name="context">The method advice context.</param>
           public void Advise(MethodAdviceContext context)
           {
               try
               {
                   var allType = GetChkParamType(context);
                   if (allType != null && allType.Any())
                   {
                       foreach (var paramInfo in allType)
                       {
                           var chkRes = ChkAllProp(paramInfo.Key.GetProperties().ToList(), paramInfo.Value);
                           if (!string.IsNullOrWhiteSpace(chkRes))
                           {
                               // 参数不符合要求
                               if (context.HasReturnValue)
                               {
                                   // 如果有返回值，进一步判断返回值类型是否指定类型，若是指定类型，则将错误信息放到指定位置。
                                   var methodInfo = context.TargetMethod as MethodInfo;
                                   if (methodInfo == null)
                                   {
                                       throw new ConstraintViolationException("参数校验失败补充返回值，未找到目标方法");
                                   }
                                   context.ReturnValue = context.IsTargetMethodAsync
                                       ? AsyncDefaultForType(methodInfo.ReturnType, chkRes)
                                       : DefaultForType(methodInfo.ReturnType, chkRes);
                                   return;
                               }
                               throw new ConstraintViolationException(chkRes);
                           }
                       }
                   }
   
                   context.Proceed();
               }
               catch (Exception e)
               {
                   Console.WriteLine(e);
                   throw;
               }
           }
       }
   ```

   由于篇幅原因，解析参数等繁琐代码就不放上来了，大家感兴趣的话可以去我的GitHub中查看源码，GitHub地址在文章末尾。

5. **和`Hibernate-Validator`的一点不同之处**

   * ***增加了对集合类型中每一个元素的校验，这需要在属性上加上`InnerValidAttribute`特性***

     有时候我们会遇到这种情况，就是我们不仅仅需要判断某个集合不能为空集合，还要进一步对集合内的每一个元素都判断是否符合验证。

     比如有一个Teacher类，里面有一个属性是Student对象的集合，这时候我们不仅仅要判断Student对象集合不能为空，而且集合中每一个Student对象的ID都必须大于0，Name不能为空等等。

     这时候我们在Student对象集合上面加上`InnerValidAttribute`特性后，组件就会深入集合的每一项进行参数验证了。

   * ***增加了自动填写错误信息到指定类型中***

     在我们团队的代码中，大部分的函数的返回值都是公共的`Result`对象类型，这个对象中定义了`ErrCode`、`ErrMsg`等信息。

     我们不希望每次参数校验失败后都抛出异常，最好是把错误信息自动填充到`ErrMsg`字段中，所以就加上了此功能，组件在判断条件未通过的时候，会先判断是否有返回值且返回值是否是`Result`类型，如果是`Result`类型的话，就把错误信息写入到`ErrMsg`字段中然后返回。

     大家也可以添加自己的类型，决定如何将错误信息写入到自己的类型中。

   * ***增加了异步方法的参数验证***

     这个把我坑得有点惨，异步类型的返回值的类型必须和函数签名一模一样，不能用`object`代替，还好最后找到一个办法来反射出了具体的类型。大家感兴趣的话可以去GitHub上看源码。

   

### 三、如何使用

这里我还是用**PhoneAttribute**来举例说明。

1. 先定义一个Model

   ```c#
       public class PhoneModel
       {
           [Phone("手机号格式不正确")]
           public string PhoneField { get; set; }
       }
   ```

2. 编写具体的逻辑类

   ```c#
           [ValidateParam("model")]
           public int PhoneTest(PhoneModel model)
           {
               return 100;
           }
   
           [ValidateParam("model")]
           public Result PhoneTest1(PhoneModel model)
           {
               return new Result() { IsSucceed = true, Message = "参数验证通过，执行成功" };
           }
   
           [ValidateParam("model")]
           public async Task<Result> PhoneTest2(PhoneModel model)
           {
               return await Task.Factory.StartNew((() =>
                   {
                       return new Result() { IsSucceed = true, Message = "参数验证通过，执行成功" };
                   }));
           }
   ```

3. 调用此函数，查看效果

   ```c#
   PhoneModel model = new PhoneModel
   {
       PhoneField = "13555555554"
   };
   res = new TestLogic().PhoneTest(model);
   
   model.PhoneField = "13555555554111222";
   res = new TestLogic().PhoneTest(model);
   
   var res1 = new TestLogic().PhoneTest1(model);
   
   res1 = new TestLogic().PhoneTest2(model).Result;
   ```

4. 观察结果，可以发现，

   第一个函数执行成功，返回了100，

   第二个函数抛出了`手机号格式不正确`的异常

   第三个函数和第四个函数都返回了`Result`对象，其中`Message`字段的值为`手机号格式不正确`，符合我们的预期。

5. 更多使用方式请查看源码中的单元测试。

### 四、项目地址

https://github.com/thxu/Th.Validator

 欢迎大家来吐槽和提意见。





