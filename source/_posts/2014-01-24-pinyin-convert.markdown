---
layout: post
title: "iOS 实现类似通讯录按拼音排序 - PinYin4Objc"
date: 2014-01-24 10:15:15 +0800
comments: true
categories: iOS
despription: iOS 实现了类似通讯录按拼音排序，索引显示的效果
keywords: iOS,通讯录,排序,索引,拼音
---
最近项目中需要实现类似通讯录那样按拼音进行排序以及索引列表的显示的功能，我这里使用了 `PinYin4Objc` 这个库来实现此功能。

`PinYinObjc`是一个效率很高的汉字转拼音类库，智齿简体和繁体中文，有如下特点：

1.效率高，使用数据缓存，第一次初始化以后，拼音数据存入文件缓存和内存缓存，后面转换效率大大提高；
2.支持自定义格式化，拼音大小写等等；
3.拼音数据完整，支持中文简体和繁体，与网络上流行的相关项目比，数据很全，几乎没有出现转换错误的问题。

下载 [PinYinObjc](https://github.com/kimziv/PinYin4Objc)

#项目中的实际应用
###项目需求：
显示一个班级的成员列表，有一个是管理员要排在最上面，下面按照拼音排序实现索引列表，效果图如下：
<!--more-->

![image](http://boy-feng.github.io/blog/images/pinyin_convert_1.png)

###代码实现过程
查询数据库获取成员列表

```
//成员列表根据 isAdmin 字段进行排序查询——order by isAdmin
NSMutableArray *members = [[ASMemberDao sharedInstance] queryAllMembersByGroupId:groupId];
//根据排序查询结果第一个为管理员
ASContact *memeberAdmin = [members objectAtIndex:0];
```

将每个成员的名字转化成拼音

```
//初始化HanyuPinyinOutputFormat对象，设置相应的 type
HanyuPinyinOutputFormat *outputFormat=[[HanyuPinyinOutputFormat alloc] init];
[outputFormat setToneType:ToneTypeWithoutTone];
[outputFormat setVCharType:VCharTypeWithV];
[outputFormat setCaseType:CaseTypeUppercase];
//遍历成员列表，将成员名字 contactName 转成拼音，并存放到 categoryName 字段中，用于排序
[members enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    ASContact *contact = (ASContact *)obj;
    NSString *outputPinyin=[PinyinHelper toHanyuPinyinStringWithNSString:contact.contactName withHanyuPinyinOutputFormat:outputFormat withNSString:@""];
    contact.categoryName = [outputPinyin uppercaseString];
}];
[outputFormat release];
```

将成员列表按照拼音字段 categoryName进行排序

```
NSSortDescriptor *sortDescriptor = [[NSSortDescriptor alloc] initWithKey:@"categoryName" ascending:YES];
NSArray *array = [[NSArray alloc] initWithObjects:sortDescriptor, nil];
[members sortUsingDescriptors:array];
[array release];
[sortDescriptor release];
```
定义一个全局变量 dataDictionary 来组织数据结构

`key: 将汉字转完拼音后的第一个字母, 也就是上图 section 中的 A、B、C...`

`value: 是一个成员数组，存放每个 section 下的成员列表`

如上图： A 是字典的一个 Key, 对应的 value 就是成员数组 {af1, af10},当然数组中存放的是成员对象。

```
dataDictionary = [[NSMutableDictionary alloc] init];
//存放每个 section 下的成员数组
NSMutableArray *currentArray = nil;
//用于获取拼音中第一个字母
NSRange aRange = NSMakeRange(0, 1);
NSString *firstLetter = nil;
//遍历成员列表组织数据结构
for (ASContact *contact in members) {
    //如果是管理员，则暂时不放如 dataDictionary 中
    if (contact.isAdmin == 1) {
         continue;
    }
    //获取拼音中第一个字母，如果已经存在则直接将该成员加入到当前的成员数组中，如果不存在，创建成员数据，添加一个 key-value 结构到 dataDictionary 中
    firstLetter = [contact.categoryName substringWithRange:aRange];
    if ([dataDictionary objectForKey:firstLetter] == nil) {
        currentArray = [NSMutableArray array];
        [dataDictionary setObject:currentArray forKey:firstLetter];
    }
    [currentArray addObject:contact];
}
```
在定义一个全局变量 `allKeys` 用于显示索引列表中索引

```
//取出 dataDictionary 中的 key 并进行排序
allKeys = [[NSMutableArray alloc] initWithArray:[[dataDictionary allKeys] sortedArrayUsingFunction:sortObjectsByKey context:NULL]];
//然后将管理员加入到排好序 allKeys 的最前面
[allKeys insertObject:@"管理员" atIndex:0];
[dataDictionary setObject:[NSArray arrayWithObjects:contactAdmin, nil] forKey:@"管理员"];
```

最后就是通过 `allKeys` 和 `dataDictionary` 进行配置一下 tableview 的各个代理就 OK 了，这里不在赘述

希望对阅读本文的你有帮助
