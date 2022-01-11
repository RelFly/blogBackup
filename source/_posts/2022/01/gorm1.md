---
title: gorm单字段查询问题
date: 2022-01-11 14:59:38
tags:
- gorm
categories:
- ORM
- gorm
---

### 前言

  在使用gorm查询单个int类型的字段值时，遇到了库里有数据但是查不到的诡异情况。
  这里记录一下排查结果。
<!-- more -->

### 问题描述

  问题背景是想要查询某个状态值，新建了一个int32类型的对象接收。
  伪代码如下：

{% codeblock lang:go %}
var status int32
err := db.WithContext(ctx).Model(&MyModel{}).Select("status").
          Where("id = ?", id), sdkId).Find(&status).Error
{% endcodeblock %}

  测试结果这里查不到数据，找到日志发现sql是正常打印出来了，但是显示没查到结果。
  把sql复制到库里查询却能查到正常结果。

### 问题解决

  开始有怀疑到是不是status类型的问题，但无法确定，为了不影响测试进度，就改为了对应的
  model结构体，然后通过model取到对应的值。

  后来将这个问题，分享给大佬同事们，得到确定答案，就是类型问题。
  gorm在最后做反射的时候，没有处理int32类型，没有返回值导致的。

### 源码解析

  虽然成功解决了问题，但是好奇心驱使我在大佬的指导下跟踪了下源码看了看，这个反射的类型
  判断到底是啥样的。

  找到gorm源码文件scan.go中的Scan函数

{% codeblock lang:go %}
func Scan(rows *sql.Rows, db *DB, initialized bool) {
  columns, _ := rows.Columns()
  values := make([]interface{}, len(columns))
  db.RowsAffected = 0
  // dest即接收sql结果的对象
  // 这里是对dest的不同类型的情况做不同的处理
  switch dest := db.Statement.Dest.(type) {
    case map[string]interface{}, *map[string]interface{}:
    // ....
    case *[]map[string]interface{}:
    // ....
    case *int, *int64, *uint, *uint64, *float32, *float64, *string, *time.Time:
    // 可以看到这里并没有int32的处理，int32会进入default
      for initialized || rows.Next() {
        initialized = false
        db.RowsAffected++
        db.AddError(rows.Scan(dest))
      }
  default:
  // ....
}
if db.RowsAffected == 0 && db.Statement.RaiseErrorOnNotFound {
  db.AddError(ErrRecordNotFound)
}
}
{% endcodeblock %}
  
  上述源码逻辑可以看到，基本类型除了int, int64, uint, uint64, float32, float64, string, time.Time之外的都会进入default逻辑。
  下面看看default的逻辑是啥样的：

{% codeblock lang:go %}
switch db.Statement.ReflectValue.Kind() {
case reflect.Slice, reflect.Array:
	// ....
case reflect.Struct:
	// ....
}
{% endcodeblock %}

  这里去除掉case中不需要关注的处理逻辑，可以看到default中也用了switch做了进一步处理，但是只处理了slice，array，struct三种类型。
  所以这里int32是无法取到值的。








