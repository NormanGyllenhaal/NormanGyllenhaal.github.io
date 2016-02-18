---
layout: post
title: "yaml入门笔记"
date: 2014-05-22 13:53:45 +0800
comments: true
categories: yaml
---

**Wiki：**

YAML（IPA: /ˈjæməl/，尾音类似camel骆驼）是一个可读性高，用来表达资料序列的格式。
YAML参考了其他多种语言，包括：XML、C语言、Python、Perl以及电子邮件格式RFC2822。
Clark Evans在2001年在首次发表了这种语言[1] ，
另外Ingy döt Net与Oren Ben-Kiki也是这语言的共同设计者。
目前已经有数种编程语言或脚本语言支援（或者说解析）这种语言。

YAML是”YAML Ain’t a Markup Language”（YAML不是一种置标语言）的递回缩写。
在开发的这种语言时，YAML 的意思其实是：”Yet Another Markup Language”（仍是一种置标语言），
但为了强调这种语言以数据做为中心，而不是以置标语言为重点，而用返璞词重新命名。

最新版本为1.2，官方说明地址： <http://www.yaml.org/spec/1.2/spec.html>

使用方式：作为配置文件，数据交换格式，序列化对象存储，测试数据文件，

一个简单的示例：<!--more-->

    ---
    receipt:     Oz-Ware Purchase Invoice
    date:        2007-08-06
    customer:
        given:   Dorothy
        family:  Gale

    items:
        - part_no:   A4786
          descrip:   Water Bucket (Filled)
          price:     1.47
          quantity:  4

        - part_no:   E1628
          descrip:   High Heeled "Ruby" Slippers
          price:     100.27
          quantity:  1

    bill-to:  &id001
        street: |
                123 Tornado Alley
                Suite 16
        city:   East Westville
        state:  KS

    ship-to:  *id001

    specialDelivery:  >
        Follow the Yellow Brick
        Road to the Emerald City.
        Pay no attention to the
        man behind the curtain.
    ...

**基本技巧：**

1，列表

使用- 表示，也就是用短杠+空白字符作为起始。

另外还有一种内置格式（inline format）可以选择──用方括号围住，并用逗号+空白区隔（类似JSON的语法）。
比如：shopping: [milk, pumpkin pie, eggs, juice]

2，映射

    — # 區塊形式
    person:
    name: John Smith
    age: 33
    — # 內置形式
    person: {name: John Smith, age: 33}

3，重复元素

使用&id001先标记，然后后面用*id001指针引用

    # & 的作用，它表示一个“锚点标记”，其它节点可以使用“*”或“<<: *”来引用它的值
    node3: &node3
      a: 001
      b: 002

    # * 的作用，指node4的内容与node3完全一致
    node4:
      *node3

    # <<: * 的作用，指node5的内容包含但不完全相同于node3的值。
    node5:
      <<: *node3
      c: 003

    #眼部雷射手術之標準程序
    ---
    - step:  &amp;id001                    #定義錨點標籤 &amp;id001
        instrument:      Lasik 2000
        pulseEnergy:     5.4
        pulseDuration:   12
        repetition:      1000
        spotSize:        1mm

    - step:
         &lt;&lt;: *id001                  # 合併鍵值：使用在錨點標籤定義的內容
         spotSize:       2mm               # 覆寫"spotSize"鍵值

    - step:
         &lt;&lt;: *id001                  # 合併鍵值：使用在錨點標籤定義的內容
         pulseEnergy:    500.0             # 覆寫鍵值
         alert: &gt;                       # 加入其他鍵值
               warn patient of
               audible pop

4，需要换行书写的字符串，两种方式：

再次强调，字串不需要包在引号之内。

保存新行(Newlines preserved)

    poetry: |                                  #譯者注：這是一首著名的五行民謠
      There once was a man from Darjeeling     #這裡曾有一個人來自大吉嶺
      Who got on a bus bound for Ealing        #他搭上一班往伊靈的公車
          It said on the door                  #門上這麼說的
          "Please don't spit on the floor"     #"請勿在地上吐痰"
      So he carefully spat on the ceiling      #所以他小心翼翼的吐在天花板上

根据设定，前方的引领空白符号（leading white space）必须对齐，以便和其他资料或是行为（如范例中的缩排）明显区分。

折叠新行(Newlines folded)

    Wrapped text         #摺疊的文字
    will be folded       #將會被收
    into a single        #進單一一個
    paragraph            #段落

    Blank lines denote   #空白的行代表
    paragraph breaks     #段落之間的區隔

和保存新行不同的是，换行字元会被转换成空白字符，空行被转换成换行，而前导空白字符则会被自动消去。上面会变成两行。

5，混合使用：

在列表中使用映射

    - {name: John Smith, age: 33}
    - name: Mary Smith
      age: 27

在映射中使用列表

    men: [John Smith, Bill Jones]
    women:
      - Mary Smith
      - Susan Williams

**更多资源：**

<http://www.yaml.org/>
