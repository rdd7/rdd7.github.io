---
layout: post
title: Swift-String
date: 2016-07-13 19:32:47.000000000 +08:00
---

* 更新于2016.07.13，当前Swift版本Swift 2.3，至少支持到Swift 3.0 (3.0文档可供参考)

## 万恶的起源：扩展字形集群 Extended Grapheme Clusters
Swift中每个字符（Character）都是一个扩展字形集群（Extended Grapheme Clusters），由一个或若干个Unicode标量（Unicode  Scalar）排列组成。

抛弃以上专业的说法，先看如下代码，再进行通俗理解：

    //代表国籍的Unicode Scalars ： U 和 S
    let regionU = "\u{1F1FA}"  //🇺 (REGIONAL INDICATOR SYMBOL LETTER U)
    let regionS = "\u{1F1F8}"  //🇸 (REGIONAL INDICATOR SYMBOL LETTER S)
    //合成 US
    let regionUS = regionU + regionS // 🇺🇸
    regionUS.characters.count // 1
    
    //regionUS内的字符在显示上等同于
    let usFlagChar: Character = "\u{1F1FA}\u{1F1F8}" // 🇺🇸


上述代码中，两个Unicode标量 （**REGIONAL INDICATOR SYMBOL LETTER U**和**REGIONAL INDICATOR SYMBOL LETTER S**）排列后重新变成了一个新字符🇺🇸，即在regionUS这个字符串内，只有一个字符，他是“**\u{1F1FA}\u{1F1F8}**”，对，这就是一个字符。

在此对字符(Character)进行通俗的解释就是，一个字符代表的是直观字面意义上可供人阅读的一个字，而这个字可以是一个Unicode标量的表示，也可以是多个Unicode标量排列的表示。

由于字符的这个特性，**不同字符所占的内存大小是不一定相同的，即使是相同外观和语义上的字符，由于其Unicode标量的不同，所占内存大小也是不同的**，下面的问题大部分都是由此引申而出。**

### 字符、字符串的比较

尽管两个字符可能由不同的Unicode标量构成，但是只要其合成的字符在语义和外观上一致，则认为相同；反之亦然。

    let testChar1: Character = "\u{D55C}" // 한
    let testChar2: Character = "\u{1112}\u{1161}\u{11AB}" // 한
    testChar1 == testChar2 // true

## 直接躺枪：获取字符串长度应该尽量用characters.count

考虑以下字符 **“한”**，其Unicode标量为 “**\u{D55C}**”，同时也可以表示为“**\u{1112}\u{1161}\u{11AB}**”

参看以下迷之代码结果

    let koreaWord = "\u{D55C}"                  // 한
    let koreaWord_ = "\u{1112}\u{1161}\u{11AB}" // 한

    koreaWord.characters.count      //1
    koreaWord.utf8.count            //3
    koreaWord.utf16.count           //1

    koreaWord_.characters.count     //1
    koreaWord_.utf8.count           //9
    koreaWord_.utf16.count          //3
    
可以看出，即使是相同外观和语义上的字符，由于其Unicode标量的不同，调用utf8.count和uft16.count，出现的结果会截然不同，原因就是上文所说，他们的字符的实际内存大小是不一样的。

在我们日常开发中，我们一般希望得到的是直观字面上的字符串长度，这时只有characters.count是符合的。像utf16.count则是将字符串内的每个Unicode标量均以UTF16表示时所需的字符数量，utf8.count同理，在String的大部分应用场景并不合适。

而值得一提的是NSString得length正是以UTF16表示的，所以即使是相同的字面量，NSString.length和String.characters.count也不一定是完全相同的。代码如下：


    String = "\u{D55C}" // 한
    koreaWordRawNSString.length // 1

    let koreaWord_RawNSString: NSString = "\u{1112}\u{1161}\u{11AB}"    // 한
    koreaWord_RawNSString.length    // 3



然而由String转换为NSString的，系统会转换其为字面意义上的目标字符，所以在String转NSString时是安全的，代码如下：

    let koreaWordNSString = koreaWord as NSString // 한
    koreaWordNSString.length    // 1

    let koreaWord_NSString = koreaWord_ as NSString // 한
    koreaWordNSString.length    // 1


## 更深远的影响：String.Index的不友好

记得初见String，一切还好，直到有天需要获取子字符串时，发现下标的表示不是那么简单的事情了。。。

如上提到，因为不同字符需要不同大小的内存占用空间，所以为了去确定某个具体的字符在内存的具体位置，你不得不从字符串的头或者尾开始一个个遍历字符来找到他的内存位置，所以，这就是String难用的原因了。。

当然，其实你写个String的extension来简化自己的代码，但是千万不要忘了这个方法没你想象中高效。

## 不会总结

虽然苹果的这种表示方法看上去有点坑，但是也必有其意义，例如表示泰文，韩文以及一些符号等。上面的这些官方文档都有详细介绍，当然，如果你不爱看官方文档，那你可能在某个特定的时间特定的场合，陷入一个谜一样的达拉然巨坑。

## 参考文档
* [The Swift Programming Language - Swift3 Edition(Beta)](https://swift.org/documentation/TheSwiftProgrammingLanguage(Swift3).epub)