# MongoDB

## 基础命令

- display the current database。

    ```MongoDB
    db
    ```

- switch databases。

    ```MongoDB
    use <db>examples
    ```

    切换前无需创建数据库。

- create database。

    ```MongoDB
    use myNewDB

    db.myNewCollection1.insertOne( { x: 1 } )
    ```

- db.collection.insertMany。

    ```MongoDB
    db.collection.insertMany(
       [ <document 1> , <document 2>, ... ],
       {
          writeConcern: <document>,
          ordered: <boolean>
       }
    )
    ```

    - document：要写入的文档。
    - writeConcern：写入策略，默认为 1，即要求确认写操作，0 是不要求。
    - ordered：指定是否按顺序写入，默认 true，按顺序写入。

    插入成功后，返回带有 "insertedIds": [ ObjectId(" ... "), ...]。

- db.collection.find(query, projection)。

    - Projection。

        ```MongoDB
        { field1: <value>, field2: <value> ... }
        ```

        value 可以是以下任何一种：
            - 1 或 true  表示返回包含该字段的内容。
            - 0 或 false 表示返回排除该字段的内容。
            - 使用 $ 运算符的表达式。
            - find() 视图上的操作不支持以下 $ 运算符：
                - $
                - $elemMatch
                - $slice
                - $meta

    - 对结果格式化添加 .pretty()。

        ```MongoDB
        db.inventory.find({}).pretty()
        ```

## DB Schema

### 概述

以 Markdown + Javascript 书写，格式如下：

````markdown
# 模块中文名

## 模块名+表名（中文描述）

```js
{
    simpleType: String, // 简单类型 String、Long、Double、Boolean、ObjectId、DateTime、Date、Time 等
    simpleTypeArray: String[], // 简单类型数组
    object: { // 对象
        field1: String
        // ...其它字段
    },
    objectArray: [{ // 对象数组
        field1: String
        // ...其它字段
    }],
    memberId: ObjectId, // member._id，通过注释说明外键关联关系
    enum: String, // 枚举，可取值：TODO（未开始）、DOING（进行中）、DONE（已完成）。枚举类型需要列举可取值范围
    multiTypes: ObjectId | String, // 多种类型，不推荐使用，会给数据分析、java、golang 带来不必要的麻烦。
    // ...其它字段
}
```

````

### 数据类型

- ObjectId：为 Mongo ObjectId (opens new window)。
- DateTime：为 Mongo Date (opens new window)。
- Date：为形如 2015-10-28 的字符串。
- Time：为形如 18:00 的字符串。

### 设计规范

- 表名使用 camelCase，*portal 模块的需要加上模块名前缀*。由于历史原因 portal collection 命名有如下几种格式：\
    - ${module}.${name} 和 ${name}，这种格式一般是产品表，其中 module 全小写，name 是小驼峰。
    - ${module}${Name} 这种格式一般是定制模块的表，其中 module 全小写，Name 是大驼峰，合在一起依然符合小驼峰要求。
- Schema 文件中还可以定义一些补充类型，补充类型使用 PascalCase。建议大家多用补充类型，从 schema 层面就复用定义。

- 如果没有额外说明，每张表的默认字段不需要重复声明。默认字段如下：

    ```js
    {
        _id: ObjectId,
        accountId: ObjectId,
        createdAt: DateTime,
        updatedAt: DateTime,
        isDeleted: Boolean,
    }
    ```

- 日期时间用 ISODate，不要用 Unix 时间戳。

- 金额（amount）单位使用分。

- 枚举值用 snake case（全小写、下划线分割字符串表示），不要用数字。Java 项目，如果操作的是独立项目内的数据，可使用全大写，如果操作的是 portal 库，也应使用全小写。
