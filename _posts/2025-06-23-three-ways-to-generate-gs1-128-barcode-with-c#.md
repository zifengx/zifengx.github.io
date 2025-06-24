---
title: C#生成GS1-128条码的三个方法
date: 2025-06-23 22:30:00 -0400
description: 前面发了一篇介绍 GS1-128 的文章，有很多小伙伴私信来问如何生成 GS1-128，今天写一下用C#生成 GS1-128 的两个方法, 顺便也提一下最近遇到的新条码 GS1 DataMatrix 的生成方法。
categories: [知识, 条形码]
tags: [条形码, barcode, gs1-128, gs1-datamatrix, c#, 物流] # TAG names should always be lowercase
---

前面发了一篇介绍 GS1-128 的文章，有很多小伙伴私信来问如何生成 GS1-128，今天写一下用C#生成 GS1-128 的三个方法，顺便也提一下最近遇到的新条码 GS1-DataMatrix 的生成方法。

## 方法1

最简单快速，使用第三方库 [ZXing.Net](https://github.com/micjahn/ZXing.Net)， ZXing.Net 是 java ZXing（Zebra Crossing）条码处理库的 .NET 移植版，支持多种一维和二维条码的读取（识别）和生成（编码）功能。支持Code 128，但不直接支持 GS1-128 的 AI/FNC1 规范。可以手动插入 FNC1 字符（ASCII 29）。

- GS1-128 使用字符 0x00F1 作为 GS
- GS1-DataMatrix 使用字符 0x29 作为 GS

### 示例代码

#### GS1-128

```c#
var writer = new BarcodeWriter { Format = BarcodeFormat.CODE_128};

var barcode = writer.Write($"{(char)0x00F1}01234567890{(char)0x00F1}45678");
```

#### GS1-DataMatrix

```c#
var writer = new BarcodeWriter { Format = BarcodeFormat.DATA_MATRIX };

var barcode = writer.Write($"{(char)29}01234567890{(char)29}45678");
```

## 方法2

还有另一个库[BarcodeLib](https://github.com/barnhill/barcodelib)也可以使用同样的方法，我没用过这个方法，感兴趣的朋友可以自行验证。

- 字符 `\u00EF` 作为 FNC1， 字符 `\u001D` 作为 GS

### 示例代码

```c#
Barcode barcode = new Barcode();
barcode.IncludeLabel = true;
string gs1Data = "\u00EF" + "01095011015300021725063010ABC123"; // \u00EF = FNC1
Image img = barcode.Encode(TYPE.CODE128, gs1Data, Color.Black, Color.White, 300, 100);
img.Save("gs1-128.png", System.Drawing.Imaging.ImageFormat.Png);
```

## 方法3

手写代码，不是很推荐，但是对于想了解条码生成原理，FNC1 分隔符，AI-应用标识符的小伙伴值得一看。转自[GS1-128条形码](https://blog.csdn.net/starfd/article/details/7190128)

### 示例代码

#### GS1-128

```c#
/// <summary>
/// GS1-128(UCC/EAN128)条形码，遵循标准GB/T 16986-2009
/// </summary>
public class GS1_128 : absCode128
{
    private List<string> _aiList = new List<string>();//rawData分割后符合商品应用标识规范的字符串集合
    /// <summary>
    /// GS1-128(UCC/EAN128)条形码，非定长标识符后面必须跟空格，定长标识符带不带无所谓
    /// </summary>
    /// <param name="rawData">包含ASCII码表32~126，其中32对应的空格sp用来通知生成FNC1分割符</param>
    public GS1_128(string rawData)
        : base(rawData)
    {
    }
    /// <summary>
    /// 对应的ASCII码范围为32~126,其中32对应的空格sp用来通知生成FNC1分割符
    /// </summary>
    /// <returns></returns>
    protected override bool RawDataCheck()
    {
        this._presentationData = string.Empty;
        string[] tempArray = this._rawData.Split((char)32);//以空格为分隔符将字符串进行分割
        foreach (string ts in tempArray)
        {
            int ptr = 0;
            do
            {
                string tempStr;
                ApplicationIdentifier ai = AI.GetAI(ts.Substring(ptr));
                int residuelength = ts.Length - ptr;//剩余字符串长度
                if (ai == null || residuelength < ai.MinLength || (!ai.IsFixedLength && residuelength > ai.MaxLength))
                {//第三个判定条件：因为不定长，而且经过空格分割，所以此时如果出现剩余字符串长度超出标识符最大长度规定，则认为错误
                    return false;
                }
                else
                {
                    int length = Math.Min(ai.MaxLength, residuelength);
                    tempStr = ts.Substring(ptr, length);
                    ptr += length;
                }
                if (!AI.IsRight(ai, tempStr))
                {
                    return false;
                }
                //展示数据加上括号
                this._presentationData += string.Format("({0}){1}", tempStr.Substring(0, ai.AILength), tempStr.Substring(ai.AILength));

                #region 修改为遵循预定义长度的AI后面才不加上FNC1,而不是实际定长的就不加上FNC1
                //if (!ai.IsFixedLength)
                //{
                //    tempStr += (char)32;//为不定长AI加上空格，以便生成条形码时确认需要在此部分后面加入分隔符FNC1
                //}
                #endregion

                this._aiList.Add(tempStr);
            }
            while (ptr < ts.Length);
        }

        return true;
    }

    protected override string GetEncodedData()
    {
        StringBuilder tempBuilder = new StringBuilder();
        CharacterSet nowCharacterSet;
        //校验字符
        int checkNum = Code128.GetStartIndex(this._aiList[0], out nowCharacterSet);
        tempBuilder.Append(Code128.BSList[checkNum]);//加上起始符
        tempBuilder.Append(Code128.BSList[Code128.FNC1]);//加上第一个FNC1表示当前是GS1-128
        checkNum += Code128.FNC1;
        int nowWeight = 2;//当前权值

        for (int i = 0; i < this._aiList.Count; i++)
        {
            string tempStr = this._aiList[i];
            int nowIndex = 0;
            #region 修改为遵循预定义长度的AI后面才不加上FNC1,而不是实际定长的就不加上FNC1
            //bool isEndWithSP = tempStr[tempStr.Length - 1] == (char)32;
            //if (isEndWithSP)
            //{
            //    tempStr = tempStr.Substring(0, tempStr.Length - 1);
            //}
            #endregion
            Code128.GetEncodedData(tempStr, tempBuilder, ref nowCharacterSet, ref nowIndex, ref nowWeight, ref checkNum);
            #region 修改为遵循预定义长度的AI后面才不加上FNC1,而不是实际定长的就不加上FNC1
            //if (isEndWithSP && i != this._aiList.Count - 1)
            //{
            //    //非定长标识符后面加上FNC1，此时FNC1作为分隔符存在
            //    Code128.EncodingCommon(tempBuilder, Code128.FNC1, ref nowWeight, ref checkNum);
            //}
            #endregion
            if (!AI.IsPredefinedAI(tempStr) && i != this._aiList.Count - 1)
            {
                //非预定长标识符后面加上FNC1，此时FNC1作为分隔符存在
                Code128.EncodingCommon(tempBuilder, Code128.FNC1, ref nowWeight, ref checkNum);
            }
        }

        checkNum %= 103;
        tempBuilder.Append(Code128.BSList[checkNum]);//加上校验符
        tempBuilder.Append(Code128.BSList[Code128.Stop]);//加上结束符
        return tempBuilder.ToString();
    }
}
```

#### AI相关

```c#
/// <summary>
/// 商品应用标识符标准 GBT 16986-2009 
/// </summary>
internal static class AI
{
    /// <summary>
    /// GBT 16986-2009 定义的应用标识符集合
    /// </summary>
    public static readonly List<ApplicationIdentifier> AIList = new List<ApplicationIdentifier>()
    {
        //20系列货运包装箱代码
        new ApplicationIdentifier("00", new List<DataFormat>(){ new DataFormat(new byte[]{18})}),
        //01全球贸易项目代码
        new ApplicationIdentifier("01", new List<DataFormat>(){ new DataFormat(new byte[]{14})}),
        //02物流单元内贸易项目的GTIN
        new ApplicationIdentifier("02", new List<DataFormat>(){ new DataFormat(new byte[]{14})}),
        //10批号
        new ApplicationIdentifier("10", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,20})}),
        //11生产日期YYMMDD
        new ApplicationIdentifier("11", new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //12付款截止日期YYMMDD
        new ApplicationIdentifier("12", new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //13包装日期YYMMDD
        new ApplicationIdentifier("13", new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //15保质期YYMMDD
        new ApplicationIdentifier("15", new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //17有效期YYMMDD
        new ApplicationIdentifier("17", new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //20产品变体
        new ApplicationIdentifier("20", new List<DataFormat>(){ new DataFormat(new byte[]{2})}),
        //21系列号
        new ApplicationIdentifier("21", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,20})}),
        //22医疗卫生行业产品二级数据
        new ApplicationIdentifier("22", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,29})}),
        //240附加产品标识
        new ApplicationIdentifier("240", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //241客户方代码
        new ApplicationIdentifier("241", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //242定制产品代码
        new ApplicationIdentifier("242", new List<DataFormat>(){ new DataFormat(new byte[]{1,6})}),
        //250二级系列号
        new ApplicationIdentifier("250", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //251源实体参考代码
        new ApplicationIdentifier("251", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //253全球文件/单证类型代码
        new ApplicationIdentifier("253", new List<DataFormat>(){ new DataFormat(new byte[]{13}),new DataFormat(new byte[]{1,17})}),
        //254GLN扩展部分代码
        new ApplicationIdentifier("254", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,20})}),
        //30可变数量
        new ApplicationIdentifier("30", new List<DataFormat>(){ new DataFormat(new byte[]{1,8})}),
        //31nn 贸易与物流量度
        new ApplicationIdentifier("31", 4, new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //32nn 贸易与物流量度
        new ApplicationIdentifier("32", 4, new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //33nn 贸易与物流量度
        new ApplicationIdentifier("33", 4, new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //34nn 贸易与物流量度
        new ApplicationIdentifier("34", 4, new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //35nn 贸易与物流量度
        new ApplicationIdentifier("35", 4, new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //36nn 贸易与物流量度
        new ApplicationIdentifier("36", 4, new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //37物流单元内贸易项目数量
        new ApplicationIdentifier("37", new List<DataFormat>(){ new DataFormat(new byte[]{1,8})}),
        //390n单一货币区内应付款金额
        new ApplicationIdentifier("390", 4, new List<DataFormat>(){ new DataFormat(new byte[]{1,15})}),
        //391n具有ISO货币代码的应付款金额
        new ApplicationIdentifier("391", 4, new List<DataFormat>(){ new DataFormat(new byte[]{3}),new DataFormat(new byte[]{1,15})}),
        //392n单一货币区内变量贸易项目应付款金额
        new ApplicationIdentifier("392", 4, new List<DataFormat>(){ new DataFormat(new byte[]{1,15})}),
        //393n具有ISO货币代码的变量贸易项目应付款金额
        new ApplicationIdentifier("393", 4, new List<DataFormat>(){ new DataFormat(new byte[]{3}),new DataFormat(new byte[]{1,15})}),
        //400客户订购单代码
        new ApplicationIdentifier("400", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //401货物托运代码
        new ApplicationIdentifier("401", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //402 装运标识代码
        new ApplicationIdentifier("402", new List<DataFormat>(){ new DataFormat(new byte[]{17})}),
        //403路径代码
        new ApplicationIdentifier("403", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //410 交货地全球位置码
        new ApplicationIdentifier("410", new List<DataFormat>(){ new DataFormat(new byte[]{13})}),
        //411 受票方全球位置码
        new ApplicationIdentifier("411", new List<DataFormat>(){ new DataFormat(new byte[]{13})}),
        //412 最终目的地全球位置码
        new ApplicationIdentifier("412", new List<DataFormat>(){ new DataFormat(new byte[]{13})}),
        //413 交货地全球位置码
        new ApplicationIdentifier("413", new List<DataFormat>(){ new DataFormat(new byte[]{13})}),
        //414 标志物理位置的全球位置码
        new ApplicationIdentifier("414", new List<DataFormat>(){ new DataFormat(new byte[]{13})}),
        //415 开票方全球位置码
        new ApplicationIdentifier("415", new List<DataFormat>(){ new DataFormat(new byte[]{13})}),
        //420 同一邮政行政区域内交货地邮政编码
        new ApplicationIdentifier("420", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //421 具有3位ISO国家(或地区)代码的交货地邮政编码
        new ApplicationIdentifier("421", new List<DataFormat>(){ new DataFormat(new byte[]{3}),new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,9})}),
        //422 贸易项目原产国(或地区)
        new ApplicationIdentifier("422", new List<DataFormat>(){ new DataFormat(new byte[]{3})}),
        //423 贸易项目初始加工国(或地区)
        new ApplicationIdentifier("423", new List<DataFormat>(){ new DataFormat(new byte[]{3}),new DataFormat(new byte[]{1,12})}),
        //424 贸易项目加工国(或地区)
        new ApplicationIdentifier("424", new List<DataFormat>(){ new DataFormat(new byte[]{3})}),
        //425 贸易项目拆分国(或地区)
        new ApplicationIdentifier("425", new List<DataFormat>(){ new DataFormat(new byte[]{3})}),
        //426 全程加工贸易项目的国家(或地区)
        new ApplicationIdentifier("426", new List<DataFormat>(){ new DataFormat(new byte[]{3})}),
        //7001 北约物资代码
        new ApplicationIdentifier("7001", new List<DataFormat>(){ new DataFormat(new byte[]{13})}),
        //7002 UN/ECE胴体肉与分割品分类
        new ApplicationIdentifier("7002", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //7003 产品的有效日期和时间
        new ApplicationIdentifier("7003", new List<DataFormat>(){ new DataFormat(new byte[]{8}),new DataFormat(new byte[]{1,2})}),
        //703s 具有3位ISO国家(或地区)代码的加工者核准号码
        new ApplicationIdentifier("703", new List<DataFormat>(){ new DataFormat(new byte[]{3}),new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,27})}),
        //8001 卷装产品
        new ApplicationIdentifier("8001", new List<DataFormat>(){ new DataFormat(new byte[]{14})}),
        //8002 蜂窝移动电话标识符
        new ApplicationIdentifier("8002", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,20})}),
        //8003 全球可回收资产标识符
        new ApplicationIdentifier("8003", new List<DataFormat>(){ new DataFormat(new byte[]{14}),new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,16})}),
        //8004 全球单个资产标识符
        new ApplicationIdentifier("8004", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //8005 单价
        new ApplicationIdentifier("8005", new List<DataFormat>(){ new DataFormat(new byte[]{6})}),
        //8006 贸易项目组件的标识符
        new ApplicationIdentifier("8006", new List<DataFormat>(){ new DataFormat(new byte[]{14}),new DataFormat(new byte[]{2}),new DataFormat(new byte[]{2})}),
        //8007 国际银行账号代码
        new ApplicationIdentifier("8007", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //8008 产品生产的日期和时间
        new ApplicationIdentifier("8008", new List<DataFormat>(){ new DataFormat(new byte[]{8}),new DataFormat(new byte[]{1,4})}),
        //8018 全球服务关系代码
        new ApplicationIdentifier("8018", new List<DataFormat>(){ new DataFormat(new byte[]{18})}),
        //8020 付款单代码
        new ApplicationIdentifier("8020", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,25})}),
        //8100 GS1-128优惠券扩展代码-NSC+Offer Code
        new ApplicationIdentifier("8100", new List<DataFormat>(){ new DataFormat(new byte[]{1}),new DataFormat(new byte[]{5})}),
        //8101 GS1-128优惠券扩展代码-NSC+Offer Code + end of offer code
        new ApplicationIdentifier("8101", new List<DataFormat>(){ new DataFormat(new byte[]{1}),new DataFormat(new byte[]{5}),new DataFormat(new byte[]{4})}),
        //8102 GS1-128优惠券扩展代码-NSC
        new ApplicationIdentifier("8102", new List<DataFormat>(){ new DataFormat(new byte[]{1}),new DataFormat(new byte[]{1})}),
        //90 贸易伙伴之间相互约定的信息
        new ApplicationIdentifier("90", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //91 公司内部信息
        new ApplicationIdentifier("91", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //92 公司内部信息
        new ApplicationIdentifier("92", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //93 公司内部信息
        new ApplicationIdentifier("93", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //94 公司内部信息
        new ApplicationIdentifier("94", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //95 公司内部信息
        new ApplicationIdentifier("95", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //96 公司内部信息
        new ApplicationIdentifier("96", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //97 公司内部信息
        new ApplicationIdentifier("97", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //98 公司内部信息
        new ApplicationIdentifier("98", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})}),
        //99 公司内部信息
        new ApplicationIdentifier("99", new List<DataFormat>(){ new DataFormat(AICharacter.a| AICharacter.n, new byte[]{1,30})})
    };
    /// <summary>
    /// 需要遵循的预定义长度的应用标识符及其对应的总长度
    /// </summary>
    public static readonly List<KeyValuePair<string, byte>> PredefinedAILength = new List<KeyValuePair<string, byte>>()
    {
        new KeyValuePair<string,byte>("00",20),new KeyValuePair<string,byte>("01",16),new KeyValuePair<string,byte>("02",16),new KeyValuePair<string,byte>("03",16),
        new KeyValuePair<string,byte>("04",18),new KeyValuePair<string,byte>("11",8),new KeyValuePair<string,byte>("12",8),new KeyValuePair<string,byte>("13",8),
        new KeyValuePair<string,byte>("14",8),new KeyValuePair<string,byte>("15",8),new KeyValuePair<string,byte>("16",8),new KeyValuePair<string,byte>("17",8),
        new KeyValuePair<string,byte>("18",8),new KeyValuePair<string,byte>("19",8),new KeyValuePair<string,byte>("20",4),new KeyValuePair<string,byte>("31",10),
        new KeyValuePair<string,byte>("32",10),new KeyValuePair<string,byte>("33",10),new KeyValuePair<string,byte>("34",10),new KeyValuePair<string,byte>("35",10),
        new KeyValuePair<string,byte>("36",10),new KeyValuePair<string,byte>("41",16)

    };


    /// <summary>
    /// 获取字符串对应的 第一个 商品应用标识
    /// </summary>
    /// <param name="rawData"></param>
    /// <returns></returns>
    internal static ApplicationIdentifier GetAI(string rawData)
    {
        return AIList.Where(ai => rawData.StartsWith(ai.AI)).FirstOrDefault();
    }
    /// <summary>
    /// 判断指定字符串是否是预定义长度应用标识符
    /// </summary>
    /// <param name="data"></param>
    /// <returns></returns>
    internal static bool IsPredefinedAI(string data)
    {
        KeyValuePair<string, byte> temp = PredefinedAILength.Where(ai => data.StartsWith(ai.Key) && data.Length == ai.Value).FirstOrDefault();
        return !string.IsNullOrEmpty(temp.Key);
    }
    /// <summary>
    /// 判断指定字符串是否是符合指定应用标识规范
    /// </summary>
    /// <param name="ai"></param>
    /// <param name="aiStr"></param>
    /// <returns></returns>
    internal static bool IsRight(ApplicationIdentifier ai, string aiStr)
    {
        //标识符部分，字符串必须以相同的AI开头
        if (!aiStr.StartsWith(ai.AI) || aiStr.Length > ai.MaxLength || aiStr.Length < ai.MinLength)
        {
            return false;
        }
        //如果AILength与ai对应的AI长度不一致时，还需检验后续几个字符是否是数字
        for (int i = ai.AI.Length; i < ai.AILength; i++)
        {
            if (!char.IsDigit(aiStr[i]))
            {
                return false;
            }
        }

        int ptr = ai.AILength;
        for (int i = 0; i < ai.DataWithoutAI.Count; i++)
        {
            DataFormat df = ai.DataWithoutAI[i];
            for (int j = 0; j < df.Length[df.Length.Length - 1]; j++)
            {
                if ((df.Character == AICharacter.n && !char.IsDigit(aiStr[ptr])) || (byte)aiStr[ptr] < 33 || (byte)aiStr[ptr] > 126)
                {
                    return false;
                }

                ptr++;
                if (ptr >= aiStr.Length)
                {
                    break;
                }
            }
        }

        return true;
    }
}
/// <summary>
/// 商品标识符的数据类型范围
/// </summary>
[Flags]
internal enum AICharacter
{
    /// <summary>
    /// 数字
    /// </summary>
    n = 1,
    /// <summary>
    /// 字母
    /// </summary>
    a = 2
}
/// <summary>
/// 商品应用标识
/// </summary>
internal sealed class ApplicationIdentifier
{
    private bool _isFixedLength;
    private string _ai;
    private byte _aiLength;
    private List<DataFormat> _dataWithoutAI;
    private byte _minLength;
    private byte _maxLength;

    public ApplicationIdentifier(string ai, List<DataFormat> dataWithoutAI)
        : this(ai, (byte)ai.Length, dataWithoutAI)
    {
    }

    public ApplicationIdentifier(string ai, byte aiLength, List<DataFormat> dataWithoutAI)
    {
        this._ai = ai;
        this._aiLength = aiLength;
        this._dataWithoutAI = dataWithoutAI;

        this._minLength = aiLength;
        this._maxLength = aiLength;

        for (int i = 0; i < this._dataWithoutAI.Count; i++)
        {
            byte[] temp = this._dataWithoutAI[i].Length;
            this._minLength += temp[0];
            this._maxLength += temp[temp.Length - 1];
        }
        this._isFixedLength = this._minLength == this._maxLength;
    }
    /// <summary>
    /// 商品应用标识符
    /// </summary>
    public string AI
    {
        get { return this._ai; }
    }
    /// <summary>
    /// 标识符长度
    /// </summary>
    public byte AILength
    {
        get { return this._aiLength; }
    }
    /// <summary>
    /// 排除标识符后字符串的数据格式
    /// </summary>
    public List<DataFormat> DataWithoutAI
    {
        get { return this._dataWithoutAI; }
    }
    /// <summary>
    /// 是否定长
    /// </summary>
    public bool IsFixedLength
    {
        get { return this._isFixedLength; }
    }
    /// <summary>
    /// 获取该商品应用标识允许的最小长度(包含AI)
    /// </summary>
    /// <returns></returns>
    public byte MinLength
    {
        get { return this._minLength; }
    }
    /// <summary>
    /// 获取该商品应用标识允许的最大长度(包含AI)
    /// </summary>
    /// <returns></returns>
    public byte MaxLength
    {
        get { return this._maxLength; }
    }
}
/// <summary>
/// 数据格式
/// </summary>
internal sealed class DataFormat
{
    private AICharacter _character;
    private byte[] _length;
    /// <summary>
    /// 默认数据格式为数字
    /// </summary>
    /// <param name="length"></param>
    public DataFormat(byte[] length)
        : this(AICharacter.n, length)
    {
    }
    public DataFormat(AICharacter character, byte[] length)
    {
        this._character = character;
        this._length = length;
    }

    /// <summary>
    /// 数据类型
    /// </summary>
    public AICharacter Character
    { get { return this._character; } }
    /// <summary>
    /// 数据长度，数组长度为1时表示定长
    /// </summary>
    public byte[] Length
    { get { return this._length; } }
}
```

#### Code128

```c#
/// <summary>
/// Code128基础相关类
/// </summary>
internal static class Code128
{
    /*
         *  128  尺寸要求
         *  最小模块宽度 x  最大1.016mm，最小0.250mm 一个系统中的x应为一恒定值  标准是1mm,放大系数0.25~1.2
         *  左右侧空白区最小宽度为 10x
         *  条高通常为32mm，实际可以根据具体要求
         *  
         * 最大物理长度不应超过 165mm，可编码的最大数据字符数为48，其中包括应用标识符和作为分隔符使用的FNC1字符，但不包括辅助字符和校验符
         * 
         * AI中FNC1同样作为分隔符使用
         * 
         * ASCII
         * 0~31 StartA  专有
         * 96~127 StartB 专有
     * 
     * EAN128不使用空格（ASCII码32）
    */

    /// <summary>
    /// Code128条空排列集合，1代表条b，0代表空s，Index对应符号字符值S
    /// </summary>
    internal static readonly List<string> BSList = new List<string>()
    {
            "212222" , "222122" , "222221" , "121223" , "121322" , "131222" , "122213" , "122312" , "132212" , "221213" ,
            "221312" , "231212" , "112232" , "122132" , "122231" , "113222" , "123122" , "123221" , "223211" , "221132" ,
            "221231" , "213212" , "223112" , "312131" , "311222" , "321122" , "321221" , "312212" , "322112" , "322211" ,
            "212123" , "212321" , "232121" , "111323" , "131123" , "131321" , "112313" , "132113" , "132311" , "211313" ,
            "231113" , "231311" , "112133" , "112331" , "132131" , "113123" , "113321" , "133121" , "313121" , "211331" ,
            "231131" , "213113" , "213311" , "213131" , "311123" , "311321" , "331121" , "312113" , "312311" , "332111" ,
            "314111" , "221411" , "431111" , "111224" , "111422" , "121124" , "121421" , "141122" , "141221" , "112214" ,
            "112412" , "122114" , "122411" , "142112" , "142211" , "241211" , "221114" , "413111" , "241112" , "134111" ,
            "111242" , "121142" , "121241" , "114212" , "124112" , "124211" , "411212" , "421112" , "421211" , "212141" ,
            "214121" , "412121" , "111143" , "111341" , "131141" , "114113" , "114311" , "411113" , "411311" , "113141" ,
            "114131" , "311141" , "411131" , "211412" , "211214" , "211232" , "2331112"
    };

    #region 条空排列集合
    //{
        //    "11011001100" , "11001101100" , "11001100110" , "10010011000" , "10010001100" ,
        //    "10001001100" , "10011001000" , "10011000100" , "10001100100" , "11001001000" ,
        //    "11001000100" , "11000100100" , "10110011100" , "10011011100" , "10011001110" ,
        //    "10111001100" , "10011101100" , "10011100110" , "11001110010" , "11001011100" ,
        //    "11001001110" , "11011100100" , "11001110100" , "11101101110" , "11101001100" ,
        //    "11100101100" , "11100100110" , "11101100100" , "11100110100" , "11100110010" ,
        //    "11011011000" , "11011000110" , "11000110110" , "10100011000" , "10001011000" ,
        //    "10001000110" , "10110001000" , "10001101000" , "10001100010" , "11010001000" ,
        //    "11000101000" , "11000100010" , "10110111000" , "10110001110" , "10001101110" ,
        //    "10111011000" , "10111000110" , "10001110110" , "11101110110" , "11010001110" ,
        //    "11000101110" , "11011101000" , "11011100010" , "11011101110" , "11101011000" ,
        //    "11101000110" , "11100010110" , "11101101000" , "11101100010" , "11100011010" ,
        //    "11101111010" , "11001000010" , "11110001010" , "10100110000" , "10100001100" ,
        //    "10010110000" , "10010000110" , "10000101100" , "10000100110" , "10110010000" ,
        //    "10110000100" , "10011010000" , "10011000010" , "10000110100" , "10000110010" ,
        //    "11000010010" , "11001010000" , "11110111010" , "11000010100" , "10001111010" ,
        //    "10100111100" , "10010111100" , "10010011110" , "10111100100" , "10011110100" ,
        //    "10011110010" , "11110100100" , "11110010100" , "11110010010" , "11011011110" ,
        //    "11011110110" , "11110110110" , "10101111000" , "10100011110" , "10001011110" ,
        //    "10111101000" , "10111100010" , "11110101000" , "11110100010" , "10111011110" ,
        //    "10111101110" , "11101011110" , "11110101110" , "11010000100" , "11010010000" ,
        //    "11010011100" , "1100011101011"
    //};
    #endregion

    internal const byte FNC3_AB = 96, FNC2_AB = 97, SHIFT_AB = 98, CODEC_AB = 99, CODEB_AC = 100, CODEA_BC = 101;

    internal const byte FNC4_A = 101, FNC4_B = 100;

    internal const byte FNC1 = 102, StartA = 103, StartB = 104, StartC = 105;
    internal const byte Stop = 106;



    /// <summary>
    /// 获取字符在字符集A中对应的符号字符值S
    /// </summary>
    /// <param name="c"></param>
    /// <returns></returns>
    internal static byte GetSIndexFromA(char c)
    {
        byte sIndex = (byte)c;
        //字符集A中 符号字符值S 若ASCII<32,则 S=ASCII+64 ,若95>=ASCII>=32,则S=ASCII-32
        if (sIndex < 32)
        {
            sIndex += 64;
        }
        else if (sIndex < 96)
        {
            sIndex -= 32;
        }
        else
        {
            throw new NotImplementedException();
        }
        return sIndex;
    }
    /// <summary>
    /// 获取字符在字符集B中对应的符号字符值S
    /// </summary>
    /// <param name="c"></param>
    /// <returns></returns>
    internal static byte GetSIndexFromB(char c)
    {
        byte sIndex = (byte)c;
        if (sIndex > 31 && sIndex < 128)
        {
            sIndex -= 32;//字符集B中ASCII码 减去32后就等于符号字符值
        }
        else
        {
            throw new NotImplementedException();
        }
        return sIndex;
    }
    internal static byte GetSIndex(CharacterSet characterSet, char c)
    {
        switch (characterSet)
        {
            case CharacterSet.A:
                return GetSIndexFromA(c);
            case CharacterSet.B:
                return GetSIndexFromB(c);
            default:
                throw new NotImplementedException();
        }
    }
    /// <summary>
    /// 判断指定字符是否仅属于指定字符集
    /// </summary>
    /// <param name="characterSet"></param>
    /// <param name="c"></param>
    /// <returns></returns>
    internal static bool CharOnlyBelongsTo(CharacterSet characterSet, char c)
    {
        switch (characterSet)
        {
            case CharacterSet.A:
                return (byte)c < 32;
            case CharacterSet.B:
                return (byte)c > 95 && (byte)c < 128;
            default:
                throw new NotImplementedException();
        }
    }
    /// <summary>
    /// 判断指定字符是否不属于指定字符集
    /// </summary>
    /// <param name="characterSet"></param>
    /// <param name="c"></param>
    /// <returns></returns>
    internal static bool CharNotBelongsTo(CharacterSet characterSet, char c)
    {
        switch (characterSet)
        {
            case CharacterSet.A:
                return (byte)c > 95;
            case CharacterSet.B:
                return (byte)c < 32 && (byte)c > 127;
            default:
                throw new NotImplementedException();
        }
    }
    /// <summary>
    /// 当编码转换时，获取相应的切换符对应的符号字符值
    /// </summary>
    /// <param name="newCharacterSet"></param>
    /// <returns></returns>
    internal static byte GetCodeXIndex(CharacterSet newCharacterSet)
    {
        switch (newCharacterSet)
        {
            case CharacterSet.A:
                return CODEA_BC;
            case CharacterSet.B:
                return CODEB_AC;
            default:
                return CODEC_AB;
        }
    }
    /// <summary>
    /// 获取转换后的字符集
    /// </summary>
    /// <param name="characterSet"></param>
    /// <returns></returns>
    internal static CharacterSet GetShiftCharacterSet(CharacterSet characterSet)
    {
        switch (characterSet)
        {
            case CharacterSet.A:
                return CharacterSet.B;
            case CharacterSet.B:
                return CharacterSet.A;
            default:
                throw new NotImplementedException();
        }
    }
    /// <summary>
    /// 获取指定字符串应该采用的起始符对应的符号字符值
    /// </summary>
    /// <param name="data"></param>
    /// <returns></returns>
    internal static byte GetStartIndex(string data, out CharacterSet startCharacterSet)
    {
        startCharacterSet = GetCharacterSet(data, 0);
        switch (startCharacterSet)
        {
            case CharacterSet.A:
                return StartA;
            case CharacterSet.B:
                return StartB;
            default:
                return StartC;
        }
    }
    /// <summary>
    /// 获取应采用的字符集
    /// </summary>
    /// <param name="data"></param>
    /// <param name="startIndex">判断开始位置</param>
    /// <returns></returns>
    internal static CharacterSet GetCharacterSet(string data, int startIndex)
    {
        CharacterSet returnSet = CharacterSet.B;
        if (Regex.IsMatch(data.Substring(startIndex), @"^\d{4,}"))
        {
            returnSet = CharacterSet.C;
        }
        else
        {
            byte byteC = GetProprietaryChar(data, startIndex);
            returnSet = byteC < 32 ? CharacterSet.A : CharacterSet.B;
        }
        return returnSet;
    }
    /// <summary>
    /// 从指定位置开始，返回第一个大于95(并且小于128)或小于32的字符对应的值
    /// </summary>
    /// <param name="data"></param>
    /// <param name="startIndex"></param>
    /// <returns>如果没有任何字符匹配，则返回255</returns>
    internal static byte GetProprietaryChar(string data, int startIndex)
    {
        byte returnByte = byte.MaxValue;
        for (int i = startIndex; i < data.Length; i++)
        {
            byte byteC = (byte)data[i];
            if (byteC < 32 || byteC > 95 && byteC < 128)
            {
                returnByte = byteC;
                break;
            }
        }
        return returnByte;
    }
    /// <summary>
    /// 获取字符串从指定位置开始连续出现数字的个数
    /// </summary>
    /// <param name="data"></param>
    /// <param name="startIndex"></param>
    /// <returns></returns>
    internal static int GetDigitLength(string data, int startIndex)
    {
        int digitLength = data.Length - startIndex;//默认设定从起始位置开始至最后都是数字
        for (int i = startIndex; i < data.Length; i++)
        {
            if (!char.IsDigit(data[i]))
            {
                digitLength = i - startIndex;
                break;
            }
        }
        return digitLength;
    }

    /// <summary>
    /// 通用方法
    /// </summary>
    /// <param name="tempBuilder"></param>
    /// <param name="sIndex"></param>
    /// <param name="nowWeight"></param>
    /// <param name="checkNum"></param>
    internal static void EncodingCommon(StringBuilder tempBuilder, byte sIndex, ref int nowWeight, ref int checkNum)
    {
        tempBuilder.Append(BSList[sIndex]);
        checkNum += nowWeight * sIndex;
        nowWeight++;
    }
    /// <summary>
    /// 获取原始数据对应的编码后数据(不包括起始符、特殊符(EAN128时)、检验符、终止符)
    /// </summary>
    /// <param name="rawData">编码对应的原始数据</param>
    /// <param name="tempBuilder">编码数据容器</param>
    /// <param name="nowCharacterSet">当前字符集</param>
    /// <param name="i">字符串索引</param>
    /// <param name="nowWeight">当前权值</param>
    /// <param name="checkNum">当前检验值总和</param>
    internal static void GetEncodedData(string rawData, StringBuilder tempBuilder, ref CharacterSet nowCharacterSet, ref int i, ref int nowWeight, ref int checkNum)
    {//因为可能存在字符集C，所以i与nowWeight可能存在不一致关系，所以要分别定义
        byte sIndex;
        switch (nowCharacterSet)
        {
            case CharacterSet.A:
            case CharacterSet.B:
                for (; i < rawData.Length; i++)
                {
                    if (char.IsDigit(rawData[i]))
                    {
                        //数字
                        int digitLength = GetDigitLength(rawData, i);
                        if (digitLength >= 4)
                        {
                            //转入CodeC
                            if (digitLength % 2 != 0)
                            {//奇数位数字，在第一个数字之后插入CodeC字符
                                sIndex = GetSIndex(nowCharacterSet, (rawData[i]));
                                EncodingCommon(tempBuilder, sIndex, ref nowWeight, ref checkNum);
                                i++;
                            }
                            nowCharacterSet = CharacterSet.C;
                            sIndex = GetCodeXIndex(nowCharacterSet);//插入CodeC切换字符
                            EncodingCommon(tempBuilder, sIndex, ref nowWeight, ref checkNum);
                            GetEncodedData(rawData, tempBuilder, ref nowCharacterSet, ref i, ref nowWeight, ref checkNum);
                            return;
                        }
                        else
                        {
                            //如果小于4位数字，则直接内部循环结束
                            for (int j = 0; j < digitLength; j++)
                            {
                                sIndex = GetSIndex(nowCharacterSet, (rawData[i]));
                                EncodingCommon(tempBuilder, sIndex, ref nowWeight, ref checkNum);
                                i++;
                            }
                            i--;//因为上面循环结束后继续外部循环会导致i多加了1，所以要减去1
                            continue;
                        }
                    }
                    else if (CharNotBelongsTo(nowCharacterSet, rawData[i]))
                    {//当前字符不属于目前的字符集
                        byte tempByte = GetProprietaryChar(rawData, i + 1);//获取当前字符后第一个属于A,或B的字符集
                        CharacterSet tempCharacterSet = GetShiftCharacterSet(nowCharacterSet);
                        if (tempByte != byte.MaxValue && CharOnlyBelongsTo(nowCharacterSet, (char)tempByte))
                        {
                            //加入转换符
                            sIndex = SHIFT_AB;
                            EncodingCommon(tempBuilder, sIndex, ref nowWeight, ref checkNum);

                            sIndex = GetSIndex(tempCharacterSet, rawData[i]);
                            EncodingCommon(tempBuilder, sIndex, ref nowWeight, ref checkNum);
                            continue;
                        }
                        else
                        {
                            //加入切换符
                            nowCharacterSet = tempCharacterSet;
                            sIndex = GetCodeXIndex(nowCharacterSet);
                            EncodingCommon(tempBuilder, sIndex, ref nowWeight, ref checkNum);
                            GetEncodedData(rawData, tempBuilder, ref nowCharacterSet, ref i, ref nowWeight, ref checkNum);
                            return;
                        }
                    }
                    else
                    {
                        sIndex = GetSIndex(nowCharacterSet, rawData[i]);
                        EncodingCommon(tempBuilder, sIndex, ref nowWeight, ref checkNum);
                    }
                }
                break;
            default:
                for (; i < rawData.Length; i += 2)
                {
                    if (i != rawData.Length - 1 && char.IsDigit(rawData, i) && char.IsDigit(rawData, i + 1))
                    {
                        sIndex = byte.Parse(rawData.Substring(i, 2));
                        EncodingCommon(tempBuilder, sIndex, ref nowWeight, ref checkNum);
                    }
                    else
                    {
                        nowCharacterSet = GetCharacterSet(rawData, i);
                        //插入转换字符
                        sIndex = GetCodeXIndex(nowCharacterSet);
                        EncodingCommon(tempBuilder, sIndex, ref nowWeight, ref checkNum);
                        GetEncodedData(rawData, tempBuilder, ref nowCharacterSet, ref i, ref nowWeight, ref checkNum);
                        return;
                    }
                }
                break;
        }
    }
}

/// <summary>
/// Code128字符集
/// </summary>
internal enum CharacterSet
{
    A,
    B,
    C
}
```

#### absCode128

```c#
/// <summary>
/// 条形码接口
/// </summary>
public interface IBarCode
{
    string RawData { get; }

    /// <summary>
    /// 条形码对应的数据
    /// </summary>
    string EncodedData { get; }

    /// <summary>
    /// 当前条形码标准
    /// </summary>
    string BarCodeType { get; }

    /// <summary>
    /// 得到条形码对应的图片
    /// </summary>
    /// <returns></returns>
    Image GetBarCodeImage();
}

/// <summary>
/// Code128抽象类
/// </summary>
public abstract class absCode128 : IBarCode
{
    protected string _encodedData;//编码数据
    protected string _rawData;//原始数据
    protected string _presentationData = null;//在条形码下面显示给人看的数据，如果为空，则取原始数据
    protected bool _dataDisplay = true;//是否显示字体

    protected byte _barCellWidth = 1;//模块单位宽度，单位Pix 默认1

    protected bool _showBlank = true;//是否显示左右空白
    protected byte _horizontalMulriple = 10;//水平左右空白对应模块的倍数
    protected byte _verticalMulriple = 8;//垂直上下空白对应模块的倍数


    protected byte _barHeight = 32;//条码高度，单位Pix 默认32

    protected Color _backColor = Color.White;//条码背景色
    protected Color _barColor = Color.Black;//条码颜色

    protected byte _fontPadding;//字体与条形码的空间间隔
    protected float _emSize;//字体大小
    protected FontFamily _fontFamily;//字体样式
    protected FontStyle _fontStyle;//字体样式
    protected StringAlignment _textAlignment;//字体布局位置
    protected Color _fontColor;//字体颜色
    protected bool _fontPositionOnBottom;//字体位置是否是底部，如果不是，则在顶部

    /// <summary>
    /// 当前条形码种类
    /// </summary>
    public string BarCodeType
    {
        get
        {
            return System.Reflection.MethodBase.GetCurrentMethod().DeclaringType.Name;
        }
    }
    /// <summary>
    /// 原始数据
    /// </summary>
    public string RawData
    {
        get { return this._rawData; }
    }
    /// <summary>
    /// 条码展示数据
    /// </summary>
    public string PresentationData
    {
        get { return string.IsNullOrEmpty(this._presentationData) ? this._rawData : this._presentationData; }
    }
    /// <summary>
    /// 条形码对应的编码数据
    /// </summary>
    public string EncodedData
    {
        get { return this._encodedData; }
    }
    /// <summary>
    /// 是否在条形码上显示展示数据
    /// </summary>
    public bool DataDisplay
    {
        get { return this._dataDisplay; }
        set { this._dataDisplay = value; }
    }
    /// <summary>
    /// 条码高度，必须至少是条码宽度的0.15倍或6.35mm，两者取大者
    /// 默认按照实际为32,单位mm
    /// </summary>
    public byte BarHeight
    {
        get { return this._barHeight; }
        set
        {
            this._barHeight = value;
        }
    }
    /// <summary>
    /// 模块宽度  单位：pix
    /// 默认宽度 1pix
    /// </summary>
    public byte BarCellWidth
    {
        get { return this._barCellWidth; }
        set
        {
            if (value == 0)
            {
                this._barCellWidth = 1;
            }
            else
            {
                this._barCellWidth = value;
            }
        }
    }
    /// <summary>
    /// 是否显示左右空白，默认标准显示
    /// </summary>
    public bool ShowBlank
    {
        get { return this._showBlank; }
        set
        {
            this._showBlank = value;
        }
    }
    /// <summary>
    /// 左右空白对应模块宽度的倍数,国际标准最小为10，如果低于10，则取10
    /// </summary>
    public byte HorizontalMulriple
    {
        get { return this._horizontalMulriple; }
        set
        {
            if (value < 10)
            {
                this._horizontalMulriple = 10;
            }
            else
            {
                this._horizontalMulriple = value;
            }
        }
    }
    /// <summary>
    /// 水平空白pix
    /// </summary>
    public int HorizontalMargin
    {
        get
        {
            if (this.ShowBlank)
            {
                return this._barCellWidth * this._horizontalMulriple;
            }
            else
            {
                return 0;
            }
        }
    }
    /// <summary>
    /// 垂直上下空白对应模块的倍数
    /// </summary>
    public byte VerticalMulriple
    {
        get { return this._verticalMulriple; }
        set
        {
            this._verticalMulriple = value;
        }
    }
    /// <summary>
    /// 垂直空白
    /// </summary>
    public int VerticalMargin
    {
        get
        {
            if (this.ShowBlank)
            {
                return this._barCellWidth * this._verticalMulriple;
            }
            else
            {
                return 0;
            }
        }
    }
    /// <summary>
    /// 字体与条形码的空间间隔,单位Pix
    /// </summary>
    public byte FontPadding
    {
        get { return this._fontPadding; }
        set
        {
            this._fontPadding = value;
        }
    }
    /// <summary>
    /// 字体大小
    /// </summary>
    public float FontSize
    {
        get { return this._emSize; }
        set { this._emSize = value; }
    }
    /// <summary>
    /// 字体样式
    /// </summary>
    public FontFamily FontFamily
    {
        get { return this._fontFamily; }
        set { this._fontFamily = value; }
    }
    /// <summary>
    /// 字体样式
    /// </summary>
    public FontStyle FontStyle
    {
        get { return this._fontStyle; }
        set { this._fontStyle = value; }
    }
    /// <summary>
    /// 字体布局位置
    /// </summary>
    public StringAlignment TextAlignment
    {
        get { return this._textAlignment; }
        set { this._textAlignment = value; }
    }
    /// <summary>
    /// 字体颜色
    /// </summary>
    public Color FontColor
    {
        get { return this._fontColor; }
        set { this._fontColor = value; }
    }

    public absCode128(string rawData)
    {
        this._rawData = rawData;
        if (string.IsNullOrEmpty(this._rawData))
        {
            throw new Exception("空字符串无法生成条形码");
        }
        this._rawData = this._rawData.Trim();
        if (!this.RawDataCheck())
        {
            throw new Exception(rawData + " 不符合 " + this.BarCodeType + " 标准");
        }
        this._encodedData = this.GetEncodedData();

        //是否加入检验可编码最大字符数超出标准48，貌似只在EAN128中有规定必须不超出48
        this.FontInit();

    }
    /// <summary>
    /// 字体初始化
    /// </summary>
    private void FontInit()
    {
        this._fontPadding = 4;
        this._emSize = 12;
        this._fontFamily = new FontFamily("Times New Roman");
        this._fontStyle = FontStyle.Regular;
        this._textAlignment = StringAlignment.Center;
        this._fontColor = Color.Black;
    }

    protected int GetBarCodePhyWidth()
    {
        //在212222这种BS单元下，要计算bsGroup对应模块宽度的倍率
        //应该要将总长度减去1（因为Stop对应长度为7），然后结果乘以11再除以6，与左右空白相加后再加上2（Stop比正常的BS多出2个模块组）
        int bsNum = (this._encodedData.Length - 1) * 11 / 6 + 2;//+ (this._showBlank ? this._blankMulriple * 2 : 0)
        return bsNum * this._barCellWidth;
    }

    /// <summary>
    /// 数据输入正确性验证
    /// </summary>
    /// <returns></returns>
    protected abstract bool RawDataCheck();

    /// <summary>
    /// 获取当前Data对应的编码数据(条空组合)
    /// </summary>
    /// <returns></returns>
    protected abstract string GetEncodedData();

    /// <summary>
    /// 获取完整的条形码
    /// </summary>
    /// <returns></returns>
    public Image GetBarCodeImage()
    {
        Image barImage = this.GetBarOnlyImage();

        int width = barImage.Width;
        int height = barImage.Height;

        width += this.HorizontalMargin * 2;
        height += this.VerticalMargin * 2;

        if (this._dataDisplay)
        {
            height += this._fontPadding + (int)this._emSize;
        }

        Image image = new Bitmap(width, height);
        Graphics g = Graphics.FromImage(image);
        g.Clear(this._backColor);

        g.DrawImage(barImage, this.HorizontalMargin, this.VerticalMargin, barImage.Width, barImage.Height);

        if (this._dataDisplay)
        {
            Font drawFont = new Font(this._fontFamily, this._emSize, this._fontStyle, GraphicsUnit.Pixel);
            Brush drawBrush = new SolidBrush(this._fontColor);
            StringFormat drawFormat = new StringFormat();
            drawFormat.Alignment = this._textAlignment;
            RectangleF reF = new RectangleF(0, barImage.Height + this.VerticalMargin + this._fontPadding, width, this._emSize);
            g.DrawString(this.PresentationData, drawFont, drawBrush, reF, drawFormat);

            drawFont.Dispose();
            drawBrush.Dispose();
            drawFormat.Dispose();
        }

        System.IO.MemoryStream ms = new System.IO.MemoryStream();
        image.Save(ms, System.Drawing.Imaging.ImageFormat.Jpeg);
        //结束绘制
        g.Dispose();
        image.Dispose();
        return Image.FromStream(ms);
    }

    /// <summary>
    /// 获取仅包含条形码的图像
    /// </summary>
    /// <returns></returns>
    private Image GetBarOnlyImage()
    {
        int width = (int)this.GetBarCodePhyWidth();
        Bitmap image = new Bitmap(width, this._barHeight);
        int ptr = 0;
        for (int i = 0; i < this._encodedData.Length; i++)
        {
            int w = (int)char.GetNumericValue(this._encodedData[i]);
            w *= this._barCellWidth;
            Color c = i % 2 == 0 ? this._barColor : this._backColor;
            for (int j = 0; j < w; j++)
            {
                for (int h = 0; h < this._barHeight; h++)
                {
                    image.SetPixel(ptr, h, c);
                }
                ptr++;
            }
        }
        return image;
    }
}
```
