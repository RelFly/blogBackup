---
title: gorm学习笔记-模型及标签
date: 2022-04-08 16:32:31
tags:
- gorm
categories:
- ORM
- gorm
---

### 前言

  使用了一段时间gorm，看他的官方文档发现有很多功能没用到，所以做一个学习笔记总结一下。
  本章首先了解下gorm中模型的定义，字段标签等功能。
<!-- more -->

### 模型定义

  模型就是对应表字段的映射，通常一个model定义如下：

{% codeblock lang:go %}
type User struct {
    Id              int64 
    Name            string 
    Age             int32 
    MarketAppStatus int32  
    DefaultValue    int32  
    CreateTime      time.Time
    UpdateTime      time.Time
}

func (user User) TableName() string {
    return "user"
}
{% endcodeblock %}

 结构体的字段与表字段一一对应，这里TableName方法就是标识对应的表名，可在使用Model(&User{})方法时指定表名。

### 约定

  model中有一些约定，依据这些约定声明模型可以减少我们的使用成本。

{% codeblock lang:go %}
// gorm.Model 的定义
type Model struct {
    ID        uint
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt
}
{% endcodeblock %}

  gorm默认id为主键字段，并且会使用CreateAt,UpdateAt,DeleteAt字段追踪创建，更新，删除时间。
  我们可以使用这些约定好的字段交给gorm去维护其变化。

### 字段标签
  
  gorm提供了一些字段标签，赋予了权限控制，时间更新等功能。

#### 字段权限控制

  字段权限主要分为读，写权限，gorm使用->表示读权限控制，<-表示写权限控制，具体规则如下：

|  标签          | 功能  |
| :----         | :---  |
| ->            | 只读权限 |
| ->:false      | 只写权限  |            
| <-:create     | 只有创建权限和读权限 |
| <-:update     | 只有更新权限和读权限 |
| <-:false      | 无创建，更新权限，有读权限 |
| <-            | 有创建，更新权限和读权限 |
| \-            | 读写时忽略 |
| \-:migration  | 迁移时忽略 |
| \-:all        | 读写，迁移时忽略 |

  默认是拥有全部权限的，设置的时候则按照设置的规则取交集。

#### 时间自动更新

  前面说过gorm约定使用CreateAt,UpdateAt字段跟踪创建，更新时间。
  但如果我们此时已经建了表，而且字段名与gorm的约定并不一致，则可以使用自动更新的标签来让gorm维护创建，更新时间。

|  标签          | 功能  |
| :----         | :---  |
| autoCreateTime| 创建时填充当前时间|
| autoUpdateTime| 更新时填充当前时间|

{% codeblock lang:go %}
type User struct {
    CreatedTime time.Time `gorm:"autoCreateTime"`       // 在创建时，如果该字段值为零值，则使用当前时间填充
    Created     int64     `gorm:"autoCreateTime"`       // 使用时间戳秒数填充创建时间
    Updated     int64     `gorm:"autoUpdateTime:nano"`  // 使用时间戳纳秒数填充更新时间
    Updated     int64     `gorm:"autoUpdateTime:milli"` // 使用时间戳毫秒数填充更新时间
}
{% endcodeblock %}

  这里gorm会根据字段的类型选择填充时间戳还是time

#### 嵌套标签

  gorm允许model嵌套匿名字段，如下：
{% codeblock lang:go %}
type User struct {
    gorm.Model
    Name string
}
// 等效于
type User struct {
    ID        uint           `gorm:"primaryKey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
    Name string
}
{% endcodeblock %}

  注意，这里必须是匿名字段才行，正常字段则需要添加embedded标签才能达到相同效果。
{% codeblock lang:go %}
type User struct {
    Id              int64
    UserInfo        UserInfo  `gorm:"embedded"`
}

type UserInfo struct {
    Name string
    Age  int32
}

// 等价于
type User struct {
    Id    int64
    Name  string
    Age   int32
}
{% endcodeblock %}  
  嵌套标签可以在model字段较多较复杂的时候做一拆分，减小结构体的复杂度。
  另外，gorm还提供了嵌套前缀标签：
{% codeblock lang:go %}
type Blog struct {
    ID      int
    Author  Author `gorm:"embedded;embeddedPrefix:author_"`
    Upvotes int32
}
// 等效于
type Blog struct {
    ID          int64
    AuthorName  string
    AuthorEmail string
    Upvotes     int32
}
{% endcodeblock %}

### 总结

      gorm提供了比较丰富的标签功能来减少开发成本，以上只是列举了比较有用的一些标签。
    还有一些如column，default标签可以在官方文档查看。

### 参考
>  [模型定义](https://gorm.io/zh_CN/docs/models.html)