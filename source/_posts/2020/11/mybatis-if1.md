---
title: MyBatis-if标签判断问题
date: 2020-11-11 22:49:43
tags:
- MyBatis
categories:
- MyBatis
---

### 前言

  最近在使用mybatis的if标签时遇到一个有趣的问题，这里记录一下。
<!-- more -->

### if标签的字符比较

  mybatis的if标签是用来在拼装sql时做一些判断。
  如果我们需要判断的对象是单个字符，比如下面的例子：
{% codeblock lang:xml%}
<if test="param == 'N'">
  and column_name = #{param}
</if>
{% endcodeblock %}

  如果这样写，会发现程序抛出类型转换异常。
  究其原因是这里将*N* 当作了char类型处理，而不是字符串。而传进来的param是String类型。
  如果追踪源码会发现在比较的时候两者无法相等，会在一个switch判断中走入强转double操作的分支，导致异常报错。

  而解决该问题的办法就是避免mybatis的“误认”：

{% codeblock lang:xml%}
<if test="param == 'N'.toString()">
  and column_name = #{param}
</if>

<if test='param == "N"'>
  and column_name = #{param}
</if>
{% endcodeblock %}
  
  如上两种写法就能避免这种问题。