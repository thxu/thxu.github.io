---
title: NetFramework自定义配置节点
date: 2020-06-08 21:00:00
tags:
	- 个人笔记
	- NetFramework
	- 自定义配置节点
categories:
	- 个人笔记
	- Ubuntu
---

在netframework项目中，我们经常会使用配置文件，比如把数据库连接字符串放到ConnectionString节点中，把其他配置信息放到appSetting节点中等。

但是有时候我们也会需要一些复杂的配置信息，比如说我们需要配置一个坐标点，我们希望能够把X轴数据和Y轴数据放到一起，这样方便阅读和梳理程序逻辑。

或者我们可能需要配置一个集合，比如redis的链接字符串，我们希望将多个redis链接字符串放到同一个节点中统一管理

这时候我们就可以通过自定义配置节点，来解决以上需求

#### 1、自定义单项配置文件节点

首先我们来看看要实现的效果

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <configSections>
    <section name="PointSetting" type="NetFrameworkConfigDemo.PointSection,NetFrameworkConfigDemo"/>
  </configSections>
    
  <PointSetting X="1" Y="2"></PointSetting>
    
  <startup>
    <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.5" />
  </startup>
</configuration>
```

在上面的配置文件中，我们定义了一个名为`PointSetting`的配置节点，接下来我们就来看看改如何读取这个节点的配置内容

1. 先定义一个类，并继承ConfigurationSection，通过这个类我们可以非常轻松的实现节点属性的读取

   ```c#
   public class PointSection : ConfigurationSection
       {
           [ConfigurationProperty("X", IsRequired = true)]
           public int X
           {
               get => Convert.ToInt32(base["X"]);
               set => base["X"] = value;
           }
   
           [ConfigurationProperty("Y", IsRequired = true)]
           public int Y
           {
               get => Convert.ToInt32(base["Y"]);
               set => base["Y"] = value;
           }
       }
   ```

2. 在App.Config文件中定义该Section

   ```xml
   <section name="PointSetting" type="NetFrameworkConfigDemo.PointSection,NetFrameworkConfigDemo"/>
   ```

   其中的type即为刚刚定义的`PointSection`

3. 读取配置文件

   ```C#
   var tmp = ConfigurationManager.GetSection("PointSetting") as PointSection;
   var x = tmp.X;
   var y = tmp.Y;
   ```

   好了，现在已经可以正常读取自定义的`PointSetting`配置节点的数据了

#### 2、自定义集合类型的配置文件节点

我们先来看看最终配置文件中要实现的效果

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <configSections>
    <section name="RedisSetting" type="NetFrameworkConfigDemo.RedisSettingSection,NetFrameworkConfigDemo"/>
  </configSections>
    
  <RedisSetting>
    <RedisConnections>
      <ConnStr DbNumber="1" DbAlias="User" ConnectionStr="10.0.0.21:6379"></ConnStr>
      <ConnStr DbNumber="2" DbAlias="Order" ConnectionStr="10.0.0.21:6379"></ConnStr>
    </RedisConnections>
  </RedisSetting>
    
  <startup>
    <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.5" />
  </startup>
</configuration>
```

在上面的配置文件中，我们定义了一个名为`RedisSetting`的配置节点，在这个节点中，我们可以管理多个redis链接字符串，接下来我们就来实现读取这个配置节点的内容。

1. 首先，还是定义一个类，并继承ConfigurationSection

   ```c#
   public class RedisSettingSection : ConfigurationSection
       {
           [ConfigurationProperty("RedisConnections", IsRequired = true)]
           [ConfigurationCollection(typeof(ConnStrSection), CollectionType = ConfigurationElementCollectionType.AddRemoveClearMap, AddItemName = "ConnStr")]
           public RedisConnectionsContainer RedisConnections
           {
               get => base["RedisConnections"] as RedisConnectionsContainer;
               set => base["RedisConnections"] = value;
           }
       }
   ```

   在上面这个类中，我们只定义了一个属性，这个属性对应的就是上面xml文件中的`RedisConnections`节点，这个属性是一个集合类型，并通过`ConfigurationCollection`特性告知系统其集合类型中每一项的具体类型，此处我们的集合中是`ConnStrSection`类型。接下来，我们就来定义`ConnStrSection`类型和`RedisConnectionsContainer`

2. `RedisConnectionsContainer`的定义

   ```c#
   public class RedisConnectionsContainer : ConfigurationElementCollection
       {
           /// <summary>在派生的类中重写时，创建一个新的 <see cref="T:System.Configuration.ConfigurationElement" />。</summary>
           /// <returns>一个新创建的 <see cref="T:System.Configuration.ConfigurationElement" />。</returns>
           protected override ConfigurationElement CreateNewElement()
           {
               return new ConnStrSection();
           }
   
           /// <summary>在派生类中重写时获取指定配置元素的元素键。</summary>
           /// <param name="element">要为其返回键的 <see cref="T:System.Configuration.ConfigurationElement" />。</param>
           /// <returns>一个 <see cref="T:System.Object" />，用作指定 <see cref="T:System.Configuration.ConfigurationElement" /> 的键。</returns>
           protected override object GetElementKey(ConfigurationElement element)
           {
               return (element as ConnStrSection).DbNumber;
           }
       }
   ```

   这个类的主要目的是告知系统改如何创建集合中的每一项，以及获取其唯一标识

3. `ConnStrSection`的定义

   ```c#
   public class ConnStrSection : ConfigurationElement
       {
           [ConfigurationProperty("DbNumber", IsRequired = true)]
           public int DbNumber
           {
               get => Convert.ToInt32(base["DbNumber"]);
               set => base["DbNumber"] = value;
           }
   
           [ConfigurationProperty("DbAlias", IsRequired = true)]
           public string DbAlias { get => base["DbAlias"].ToString(); set => base["DbAlias"] = value; }
   
           [ConfigurationProperty("ConnectionStr", IsRequired = true)]
           public string ConnectionStr { get => base["ConnectionStr"].ToString(); set => base["ConnectionStr"] = value; }
   
           /// <summary>返回表示当前对象的字符串。</summary>
           /// <returns>表示当前对象的字符串。</returns>
           public override string ToString()
           {
               return $"db={DbNumber}{Environment.NewLine}" +
                      $"dbAlias={DbAlias}{Environment.NewLine}" +
                      $"connStr={ConnectionStr}";
           }
       }
   ```

   这个类用来定义集合中每一项的属性

4. 然后我们就可以来读取此配置节点的具体内容了。

   ```c#
               var tmp = ConfigurationManager.GetSection("RedisSetting");
               var section1 = tmp as RedisSettingSection;
               foreach (ConnStrSection conn in section1.RedisConnections)
               {
                   Console.WriteLine($"{conn}");
               }
   ```



#### 补充一点小知识

* ConfigurationSection：对应整个自定义节点Xml的最外层节点
* ConfigurationElement：Section节点下的一个子节点，不包含集合
* ConfigurationElementCollection：对应Element集合