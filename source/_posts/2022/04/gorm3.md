---
title: gorm学习笔记-gorm中的那些方法
date: 2022-04-29 15:13:26
tags:
- gorm
categories:
- ORM
- gorm
---

### 前言

  本章来看看gorm中各种操作的方法。
<!-- more -->

### 链式操作

  gorm支持链式操作，也就是可以这样写：

{% codeblock lang:go %}
err := db.Model(user).Where("id = ?", user.Id).Find(&user).Error
{% endcodeblock %}

  上述代码中包含两种方法，一种是Model(),Where()，一种是Find()。
  两种方法的区别是，前者是链式方法，用来构造sql，不会真正的去执行；
  后者被称作finisher方法，相当于结束信号，会执行回调方法执行sql。

### 链式方法
  
  gorm的chainable_api.go文件中包含了所有的链式方法。
  下面来看看常用的一些链式方法。

#### Model/Table

  Model()和Table()方法的作用一样，都是用来指定查询的表名。
  不同点在于Model方法的入参是我们定义的模型结构体，按照约定用结构体名的蛇形命名复数形式或者其实现的TableName方法返回值作为sql的表名。
  Table的入参则是一个字符串，直接用这个字符串作为表名。
{% codeblock lang:go %}
// 后文的例子无特别说明皆使用该结构体
type UserInfo struct {
    Id   int64
    Name string
    Age  int32
}

func (user User) TableName() string {
    return "user"
}

db.Model(user).Where("id = ?", user.Id).Find(&user)
{% endcodeblock %}
  这里解释下Model方法，如上例，这里因为结构体实现了TableName方法，所以会以user作为sql的表名。
  如果这里没有TableName方法，会以user_infos作为表名，注意这里会取蛇形命名的复数形式。

#### Select

  Select方法用于在查询时指定查询的字段。

{% codeblock lang:go %}
names := make([]string,0)
db.Model(&user).Select("name").Find(&names).Error
// sql:SELECT name FROM user
{% endcodeblock %}

#### Omit

  Omit与Select作用相反，他是用来忽略字段，可以在创建，更新，查询等操作时用来忽略某个字段。

{% codeblock lang:go %}
db.Model(&user).Omit("name").Find(&user)
// sql:SELECT id,age FROM user
{% endcodeblock %}

#### Where/Not/Or

  Where方法不用多说，就是添加Where后的条件，比较通用可以在参数列表中写好各种条件语句。
  Not，Or方法则可以认为是不等于，或条件的附加

{% codeblock lang:go %}
db.Model(&user).Where("id = ?", 1).Find(&user)
// sql:SELECT * FROM user WHERE id = 1

params := []int{1, 2}
db.Model(&user).Where("id in ?", params).Find(&user)
// sql:SELECT * FROM user WHERE id in (1,2)

db.Model(&user).Not("id = ?", 1).Find(&user)
// sql:SELECT * FROM user WHERE id <> 1

// Or方法需要和其他条件配合使用才行
db.Model(&user).Or("id = ?", 1).Find(&user)
// sql:SELECT * FROM user WHERE id = 1

db.Model(&user).Or("id = ?", 1).Not("id = ?", 1).Find(&user)
// sql:SELECT * FROM user WHERE NOT id = 1 OR id = 1
{% endcodeblock %}

#### Group/Having

  Group和Having方法对应group和having关键字。

{% codeblock lang:go %}
db.Model(&user).Select("name").Group("name").Find(&users)
// sql:SELECT name FROM user GROUP BY name
db.Model(&user).Select("name").Group("name").Having("name like ?", "p").Find(&users)
// sql:SELECT name FROM user GROUP BY name HAVING name like 'p'
{% endcodeblock %}

#### Order

 Order方法对应order关键字，用于附加排序条件。

{% codeblock lang:go %}
db.Model(&user).Order("name desc,age asc").Find(&users)
// sql:SELECT * FROM user ORDER BY name desc,age asc
{% endcodeblock %}

#### Limit/Offset

  Limit和Offset则是分页操作的组合，对应sql中limit和offset关键字。

{% codeblock lang:go %}
db.Model(&user).Offset(1).Limit(2).Find(&users)
// sql:SELECT * FROM user LIMIT 2 OFFSET 1
{% endcodeblock %}

#### Scopes

  Scopes方法简单解释就是可以复用一些通用逻辑。
{% codeblock lang:go %}
// 通用分页方法
func paginate(pageNum, pageSize int) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if pageNum < 1 {
            pageNum = 1
        }
        if pageSize < 0 || pageSize > 50 {
            pageSize = 50
        }
        offset := (pageNum - 1) * pageSize
        return db.Offset(offset).Limit(pageSize)
    }
}
db.Model(&user).Scopes(gormDb.Paginate(2, 3)).Find(&users)
// sql:SELECT * FROM user LIMIT 3 OFFSET 3
{% endcodeblock %}

  如上例中一样，先定义一个返回值为 func(db *gorm.DB) *gorm.DB的函数，里面是通用的分页参数附加的逻辑。
  这个函数就可以通过Scopes方法复用到所有需要分页的sql构造中，减少了重复代码。

### Finisher方法

  Finisher方法在文件finisher_api.go中。
  下面看一下常用的一些Finisher方法。

#### Create/CreateInBatches

  Create根据入参类型创建单条或者多条数据，CreateInBatches是批量创建数据，不同的是他的入参可以限制每一批插入的量。

{% codeblock lang:go %}
user := gormDb.User{
    Id:   13,
    Name: "xxx",
    Age:  11,
}
db.Create(&user)
// sql: INSERT INTO user (name,age,id) VALUES ('xxx',11,13)

user1 := gormDb.User{
    Name: "xxx",
    Age:  11,
}
user2 := gormDb.User{
    Name: "xxx",
    Age:  11,
}
users := make([]gormDb.User, 0)
users = append(users, user1)
users = append(users, user2)
db.Create(&users)
// sql: INSERT INTO user (name,age) VALUES ('xxx',11),('xxx',11)

user1 := gormDb.User{
    Name: "dfdf",
    Age:  12,
}
user2 := gormDb.User{
    Name: "bbbb",
    Age:  15,
}
users := make([]gormDb.User, 0)
users = append(users, user1)
users = append(users, user2)
db.CreateInBatches(&users, 1)
// sql: INSERT INTO user (name,age) VALUES ('dfdf',12)
// sql: INSERT INTO user (name,age) VALUES ('bbbb',15)
{% endcodeblock %}

  可以看到CreateInBatches会根据传入的limit，分批执行insert。

#### Save

  Save方法比较特殊，如果主键id有值，他会执行update操作，如果主键id没有值会执行insert操作。

{% codeblock lang:go %}
user := gormDb.User{
    Name: "ddd",
}
db.Save(&user)
// sql:INSERT INTO user (name,age,) VALUES ('ddd',0)
user.id = 10
db.Save(&user)
// sql: UPDATE user SET name='ddd',age=0 WHERE id = 10
users := make([]gormDb.User, 0)
users = append(users, user)
db.Save(&users)
// sql: INSERT INTO user (name,age,id) VALUES ('ddd',0,10) ON DUPLICATE KEY UPDATE name=VALUES(name),age=VALUES(age)
{% endcodeblock %}

  Save方法的入参可以是结构体也可以是切片，执行的sql不一样但是最终效果是一样的。
  这里要注意的是，如果结构体的某个字段没有赋值，Save时会自动为其设置零值参与sql的执行。
  在做更新的时候，没有赋值可能会错误的将不应更新的字段更新为零值。

#### First/Take/Last

  First,Take,Last三个方法都是查询单条数据的，不同点在于First，Last会根据主键排序，Take则不指定排序。
  另外，需要注意，这三个方法查不到数据时，会抛出ErrNotFound错误。

{% codeblock lang:go %}
db.First(&user)
// sql: SELECT * FROM user ORDER BY user.id LIMIT 1
db.Take(&user)
// sql: SELECT * FROM user WHERE user.id = 1 LIMIT 1
db.Last(&user)
// sql: SELECT * FROM user WHERE user.id = 1 ORDER BY user.id DESC LIMIT 1
{% endcodeblock %}

#### Find/FindInBatches

  Find是通用查询方法，FindInBatches是分批查询，并且可以对每条数据做自定义处理。

{% codeblock lang:go %}
users := make([]*gormDb.User, 0)
db.Find(&users)
// sql: SELECT * FROM user
db.FindInBatches(&users, 10, func(tx *gorm.DB, batch int) error {
    for _, e := range users {
        / 批量处理找到的记录
        fmt.Printf("element:%v\n", e)
    }
    return nil
})
// sql: SELECT * FROM user ORDER BY user.id LIMIT 10
// sql: SELECT * FROM user WHERE user.id > 10 ORDER BY user.id LIMIT 10
{% endcodeblock %}

  FindInBatches会根据第二个参数batchSize分批执行select，从第二批开始，会添加id大于的条件，所以建议这种用法最好用于主键自增的表。
  然后，FindBatches还有一个函数类型的参数，可以在这个函数内定义对每条数据的处理逻辑。
  最后需要注意的是，FindInBAtches会一直执行select直到某次查询结果行数小于batchSize时才会结束，所以他无法用于类似分页的查询场景。

#### Update/Updates/UpdateColumn/UpdateColumns

  Update用于更新单个字段，而且必须有where条件，否则会返回ErrMissingWhereClause错误。
  Updates用于更新多个字段，入参可以是结构体也可以是map类型，当是结构体时，只会更新非零值的字段。
  UpdateColumn，UpdateColumns与Update，Updates用法一致，不同点在于前者不会出发Hook方法，且不会自动为更新时间赋值。

{% codeblock lang:go %}
// 为了区分两者的区别，这里加上更新时间字段
type UserInfo struct {
    Id   int64
    Name string
    Age  int32
    UpdatedAt time.Time
}
user := gormDb.User{}

// 不附加where条件(包括mdel没有主键)的情况
db.Model(&user).Update("name", "33")
// sql: UPDATE user SET name='33',updated_at='2022-05-05 17:23:25.672'
// err: WHERE conditions required

// 正常使用的例子
db.Model(&user).Where("name like ?", "%p%").Update("name", "33")
// sql: UPDATE user SET name='33',updated_at='2022-05-06 14:34:14.455' WHERE name like '%p%'
user.id = 1
db.Model(&user).Update("name", "33")
// sql: UPDATE user SET name='33',updated_at='2022-05-06 14:35:02.652' WHERE id = 1

// UpdateColumn和Update一样，也需要附加where条件，这里看看他与Update的不同点
db.Model(&user).UpdateColumn("name", "33")
// sql: UPDATE user SET name='33' WHERE id = 1
{% endcodeblock %}

  可以从例子中看到，只要附带了条件，无论是model中主键信息不为空还是附加了where条件都能正常执行Update方法，否则会抛出错误信息。
  另外，可以看到，这里Update会自动更新updated_at字段，UpdateColumn则不会。

{% codeblock lang:go %}
user := gormDb.User{}
user.Name = "test_updates"
db.Model(&user).Where("name like ?", "%l%").Updates(&user)
// sql: UPDATE user SET name='test_updates',updated_at='2022-05-06 15:32:36.804' WHERE name like '%l%'

updateMap := make(map[string]interface{})
updateMap["name"] = ""
updateMap["age"] = 0
db.Model(&user).Where("age = ?", 11).Updates(updateMap)
// sql: UPDATE user SET age=0,name='',updated_at='2022-05-06 15:43:06.481' WHERE age = 11
{% endcodeblock %}

  Updates可以更新多个字段，其入参类型struct和map的区别是，struct入参不会更新零值的字段，map入参可以。
  UpdateColumns这里就不再演示了，与Updates的区别如上所说不会触发Hook方法及追踪更新时间。

#### Row/Rows/Scan/ScanRows

  Row，Rows方法可以再没有定义模型的情况下获取查询结果。
  Scan，ScanRows方法则是用来从Row，Rows中取出结果。

{% codeblock lang:go %}
row := db.Table("user").Select("name,age").Row()
name := ""
age := 0
err := row.Scan(&name, &age)
fmt.Printf("result:%v,%v,err:%v", name, age, err)
rows, err := db.Table("user").Select("name,age").Rows()
if err != nil {
    return
}
defer rows.Close()
for rows.Next() {
    err := rows.Scan(&name, &age)
    fmt.Printf("result1:%v,%v,err:%v\n", name, age, err)
    user := gormDb.User{}
    err = db.ScanRows(rows, &user)
    fmt.Printf("result2:%v,err:%v\n", user, err)
    // 其他业务逻辑
}
{% endcodeblock %}

#### Pluck

  Pluck用于查询单个字段信息。
{% codeblock lang:go %}
var names []string
// 下面两个操作效果一样
db.Model(&user).Pluck("name", &names)
db.Model(user).Select("name").Find(&names)
{% endcodeblock %}

### 总结

    以上，就是gorm中比较常用的一些方法，基本涵盖了日常开发中增删查改的需求。
    使用时需要注意链式方法必须在Finisher方法之前才有效，还有update零值更新的问题。