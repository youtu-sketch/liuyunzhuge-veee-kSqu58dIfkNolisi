[合集 \- Ideal库 \- Common库(6\)](https://github.com)[1\.开源 \- Ideal库 \- 常用时间转换扩展方法（一）11\-07](https://github.com/hugogoos/p/18531206)[2\.开源 \- Ideal库 \- 常用时间转换扩展方法（二）11\-09](https://github.com/hugogoos/p/18535467)[3\.开源 \- Ideal库 \- 获取特殊时间扩展方法（三）11\-11](https://github.com/hugogoos/p/18538819):[westworld加速](https://tianchuang88.com)[4\.开源 \- Ideal库 \-获取特殊时间扩展方法（四）11\-12](https://github.com/hugogoos/p/18539591)[5\.开源 \- Ideal库 \- 常用枚举扩展方法（一）11\-13](https://github.com/hugogoos/p/18542907)6\.开源 \- Ideal库 \- 常用枚举扩展方法（二）11\-14收起
书接上回，今天继续和大家享一些关于枚举操作相关的常用扩展方法。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241113234605566-243897916.png)


今天主要分享通过枚举值转换成枚举、枚举名称以及枚举描述相关实现。


我们首先修改一下上一篇定义用来测试的正常枚举，新增一个枚举项，代码如下：



```
//正常枚举
internal enum StatusEnum
{
    [Description("正常")]
    Normal = 0,
    [Description("待机")]
    Standby = 1,
    [Description("离线")]
    Offline = 2,
    Online = 3,
    Fault = 4,
}

```

# ***01***、根据枚举值转换成枚举


该方法接收枚举值作为参数，并转为对应枚举，转换失败则返回空。


枚举类Enum中自带了两种转换方法，其一上篇文章使用过即Parse，这个方法可以接收string或Type类型作为参数，其二为ToObject方法，接收参数为整数类型。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241113234615033-1922719206.png)


因为枚举值本身就是整数类型，因此我们选择ToObject方法作为最终实现，这样就避免使用Parse方法时还需要把整数类型参数进行转换。


同时我们通过上图可以发现枚举值可能的类型有uint、ulong、ushort、sbyte、long、int、byte、short八种情况。


因此下面我们以int类型作为示例，进行说明，但同时考虑到后面通用性、扩展性，我们再封装一个公共的泛型实现可以做到支持上面八种类型。因此本方法会调用一个内部私有方法，具体如下：



```
//根据枚举值转换成枚举，转换失败则返回空
public static T? ToEnumByValue(this int value) where T : struct, Enum
{
    //调用根据枚举值转换成枚举方法
    return ToEnumByValue<int, T>(value);
}

```

而内部私有方法即通过泛型实现对多种类型支持，我们先看代码实现再详细讲解，具体代码如下：



```
//根据枚举值转换成枚举，转换失败则返回空
private static TEnum? ToEnumByValue(this TSource value)
    where TSource : struct
    where TEnum : struct, Enum
{
    //检查整数值是否是有效的枚举值并且是否是有效位标志枚举组合项
    if (!Enum.IsDefined(typeof(TEnum), value) && !IsValidFlagsMask(value))
    {
        //非法数据则返回空
        return default;
    }
    //有效枚举值则进行转换
    return (TEnum)Enum.ToObject(typeof(TEnum), value);
}

```

该方法首先验证参数合法性，验证通过直接使用ToObject方法进行转换。


参数验证首先通过Enum.IsDefined方法校验参数是否是有效的枚举项，这是因为无论是ToObject方法还是Parse方法对于整数类型参数都是可以转换成功的，无论这个参数是否是枚举中的项，因此我们需要首先排查掉非枚举中的项。


而该方法中IsValidFlagsMask方法主要是针对位标志枚举组合情况，位标志枚举特性导致即使我们枚举项中没有定义相关项，但是可以通过组合得到而且是合法的，因此我们需要对位标志枚举单独处理，具体实现代码如下：



```
//存储枚举是否为位标志枚举
private static readonly ConcurrentDictionarybool> _flags = new();
//存储枚举对应掩码值
private static readonly ConcurrentDictionarylong> _flagsMasks = new();
private static bool IsValidFlagsMask<TSource, TEnum>(TSource source)
    where TSource : struct
    where TEnum : struct, Enum
{
    var type = typeof(TEnum);
    //判断是否为位标志枚举,如果有缓存直接获取，否则计算后存入缓存再返回
    var isFlags = _flags.GetOrAdd(type, (key) =>
    {
        //检查枚举类型是否有Flags特性
        return Attribute.IsDefined(key, typeof(FlagsAttribute));
    });
    //如果不是位标志枚举则返回false
    if (!isFlags)
    {
        return false;
    }
    //获取枚举掩码，如果有缓存直接获取，否则计算后存入缓存再返回
    var mask = _flagsMasks.GetOrAdd(type, (key) =>
    {
        //初始化存储掩码变量
        var mask = 0L;
        //获取枚举所有值
        var values = Enum.GetValues(key);
        //遍历所有枚举值，通过位或运算合并所有枚举值
        foreach (Enum enumValue in values)
        {
            //将枚举值转为long类型
            var valueLong = Convert.ToInt64(enumValue);
            // 过滤掉负数或无效的值，规范的位标志枚举应该都为非负数
            if (valueLong >= 0)
            {
                //合并枚举值至mask
                mask |= valueLong;
            }
        }
        //返回包含所有枚举值的枚举掩码
        return mask;
    });
    var value = Convert.ToInt64(source);
    //使用待验证值value和枚举掩码取反做与运算
    //结果等于0表示value为有效枚举值
    return (value & ~mask) == 0;
}

```

该方法首先是判断当前枚举是否是位标志枚举即枚举是否带有Flags特性，可以通过Attribute.IsDefined实现，考虑到性能问题，因此我们把枚举是否为位标志枚举存入缓存中，当下次使用时就不必再次判断了。


如果当前枚举不是位标志枚举则之间返回false。


如果是位标志枚举则进入关键点了，如何判断一个值是否为一组值或一组值任意组合里面的一个？


这里用到了位掩码技术，通过按位或对所有枚举项进行标记，达到合并所有枚举项的目的，同时还包括可能的组合情况。


这里存储掩码的变量定义为long类型，因为我们需要兼容上文提到的八种整数类型。同时一个符合规范的位标志枚举设计枚举值是不会出现负数的因此也需要过滤掉。


同时考虑到性能问题，也需要把每个枚举对于的枚举掩码记录到缓存中方便下次使用。


拿到枚举掩码后我们需要对其进行取反，即表示所有符合要求的值，此值再与待验证参数做按位与操作，如果不为0表示待验证才是为无效枚举值，否则为有效枚举值。


关于位操作我们后面找机会再单独详解讲解其中原理和奥秘。


讲解完整个实现过程我们还需要对该方法进行详细的单元测试，具体分为以下几种情况：


(1\) 正常枚举值，成功转换成枚举；


(2\) 不存在的枚举值，但是可以通过枚举项按位或合并得到，返回空；


(3\) 不存在的枚举值，也不可以通过枚举项按位或合并得到，返回空；


(4\) 正常位标志枚举值，成功转换成枚举；


(5\) 不存在的枚举值，但是可以通过枚举项按位或合并得到，成功转换成枚举；


(6\) 不存在的枚举值，也不可以通过枚举项按位或合并得到，返回空；


具体实现代码如下：



```
[Fact]
public void ToEnumByValue()
{
    //正常枚举值，成功转换成枚举
    var status = 1.ToEnumByValue();
    Assert.Equal(StatusEnum.Standby, status);
    //不存在的枚举值，但是可以通过枚举项按位或合并得到，返回空
    var isStatusNull = 5.ToEnumByValue();
    Assert.Null(isStatusNull);
    //不存在的枚举值，也不可以通过枚举项按位或合并得到，返回空
    var isStatusNull1 = 8.ToEnumByValue();
    Assert.Null(isStatusNull1);
    //正常位标志枚举值，成功转换成枚举
    var flags = 3.ToEnumByValue();
    Assert.Equal(TypeFlagsEnum.HttpAndUdp, flags);
    //不存在的枚举值，但是可以通过枚举项按位或合并得到，成功转换成枚举
    var flagsGroup = 5.ToEnumByValue();
    Assert.Equal(TypeFlagsEnum.Http | TypeFlagsEnum.Tcp, flagsGroup);
    //不存在的枚举值，也不可以通过枚举项按位或合并得到，返回空
    var isFlagsNull = 8.ToEnumByValue();
    Assert.Null(isFlagsNull);
}

```

# ***02***、根据枚举值转换成枚举或默认值


该方法是对上一个方法的补充，用于处理转换不成功时，则返回一个指定默认枚举，具体代码如下：



```
//根据枚举值转换成枚举，转换失败则返回默认枚举
public static T? ToEnumOrDefaultByValue(this int value, T defaultValue) 
    where T : struct, Enum
{
    //调用根据枚举值转换成枚举方法
    var result = value.ToEnumByValue();
    if (result.HasValue)
    {
        //返回枚举
        return result.Value;
    }
    //转换失败则返回默认枚举
    return defaultValue;
}

```

然后我们进行一个简单单元测试，代码如下：



```
[Fact]
public void ToEnumOrDefaultByValue()
{
    //正常枚举值，成功转换成枚举
    var status = 1.ToEnumOrDefaultByValue(StatusEnum.Offline);
    Assert.Equal(StatusEnum.Standby, status);
    //不存在的枚举值，返回指定默认枚举
    var statusDefault = 5.ToEnumOrDefaultByValue(StatusEnum.Offline);
    Assert.Equal(StatusEnum.Offline, statusDefault);
}

```

# ***03***、根据枚举值转换成枚举名称


该方法接收枚举值作为参数，并转为对应枚举名称，转换失败则返回空。


实现则是通过根据枚举值转换成枚举方法获得枚举，然后通过枚举获取枚举名称，具体代码如下：



```
//根据枚举值转换成枚举名称，转换失败则返回空
public static string? ToEnumNameByValue(this int value) where T : struct, Enum
{
    //调用根据枚举值转换成枚举方法
    var result = value.ToEnumByValue();
    if (result.HasValue)
    {
        //返回枚举名称
        return result.Value.ToString();
    }
    //转换失败则返回空
    return default;
}

```

我们进行如下单元测试：



```
[Fact]
public void ToEnumNameByValue()
{
    //正常枚举值，成功转换成枚举名称
    var status = 1.ToEnumNameByValue();
    Assert.Equal("Standby", status);
    //不存在的枚举值，返回空
    var isStatusNull = 10.ToEnumNameByValue();
    Assert.Null(isStatusNull);
    //正常位标志枚举值，成功转换成枚举名称
    var flags = 3.ToEnumNameByValue();
    Assert.Equal("HttpAndUdp", flags);
    //不存在的位标志枚举值，返回空
    var isFlagsNull = 20.ToEnumNameByValue();
    Assert.Null(isFlagsNull);
}

```

# ***04***、根据枚举值转换成枚举名称默认值


该方法是对上一个方法的补充，用于处理转换不成功时，则返回一个指定默认枚举名称，具体代码如下：



```
//根据枚举值转换成枚举名称，转换失败则返回默认枚举名称
public static string? ToEnumNameOrDefaultByValue(this int value, string defaultValue) 
    where T : struct, Enum
{
    //调用根据枚举值转换成枚举名称方法
    var result = value.ToEnumNameByValue();
    if (!string.IsNullOrWhiteSpace(result))
    {
        //返回枚举名称
        return result;
    }
    //转换失败则返回默认枚举名称
    return defaultValue;
}

```

进行简单的单元测试，具体代码如下：



```
[Fact]
public void ToEnumNameOrDefaultByValue()
{
    //正常枚举值，成功转换成枚举名称
    var status = 1.ToEnumNameOrDefaultByValue("离线");
    Assert.Equal("Standby", status);
    //不存在的枚举名值，返回指定默认枚举名称
    var statusDefault = 12.ToEnumNameOrDefaultByValue("离线");
    Assert.Equal("离线", statusDefault);
}

```

# ***05***、根据枚举值转换成枚举描述


该方法接收枚举值作为参数，并转为对应枚举名称，转换失败则返回空。


实现则是通过根据枚举值转换成枚举方法获得枚举，然后通过枚举获取枚举描述，具体代码如下：



```
//根据枚举值转换成枚举描述，转换失败则返回空
public static string? ToEnumDescByValue(this int value) where T : struct, Enum
{
    //调用根据枚举值转换成枚举方法
    var result = value.ToEnumByValue();
    if (result.HasValue)
    {
        //返回枚举描述
        return result.Value.ToEnumDesc();
    }
    //转换失败则返回空
    return default;
}

```

单元测试如下：



```
[Fact]
public void ToEnumDescByValue()
{
    //正常位标志枚举值，成功转换成枚举描述
    var flags = 3.ToEnumDescByValue();
    Assert.Equal("Http协议,Udp协议", flags);
    //正常的位标志枚举值，组合项不存在，成功转换成枚举描述
    var flagsGroup1 = 5.ToEnumDescByValue();
    Assert.Equal("Http协议,Tcp协议", flagsGroup1);
}

```

# ***06***、根据枚举值转换成枚举描述默认值


该方法是对上一个方法的补充，用于处理转换不成功时，则返回一个指定默认枚举描述，具体代码如下：



```
//根据枚举值转换成枚举描述，转换失败则返回默认枚举描述
public static string? ToEnumDescOrDefaultByValue(this int value, string defaultValue) 
    where T : struct, Enum
{
    //调用根据枚举值转换成枚举描述方法
    var result = value.ToEnumDescByValue();
    if (!string.IsNullOrWhiteSpace(result))
    {
        //返回枚举描述
        return result;
    }
    //转换失败则返回默认枚举描述
    return defaultValue;
}

```

单元测试代码如下：



```
[Fact]
public void ToEnumDescOrDefaultByValue()
{
    //正常枚举值，成功转换成枚举描述
    var status = 1.ToEnumDescOrDefaultByValue("测试");
    Assert.Equal("待机", status);
    //不存在的枚举值，返回指定默认枚举描述
    var statusDefault = 11.ToEnumDescOrDefaultByValue("测试");
    Assert.Equal("测试", statusDefault);
}

```

稍晚些时候我会把库上传至Nuget，大家可以直接使用Ideal.Core.Common。


***注***：测试方法代码以及示例源码都已经上传至代码库，有兴趣的可以看看。[https://gitee.com/hugogoos/Ideal](https://github.com)


