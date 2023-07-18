# sqflite使用

适用于SQLite的Flutter插件，一个自包含、高可靠性、嵌入式的SQL数据库引擎。

https://pub.dev/packages/sqflite

## 安装

```shell
flutter pub add sqflite
```

## 导入

```dart
import 'package:sqflite/sqflite.dart';
```

## 使用

基本使用教程请查看https://pub.dev/packages/sqflite

我会根据业务情况分为三块内容

* 表Model (对应表的字段)
* 表DBHelper (该表的增删查改能力)
* DBHelper (管理db文件创建)

首先创建一个叫MM.db的库,假设他是用来保存app业务数据的,比如用户信息,xx数据等.其中可能包含多张表.

假设有一张用户表,命名为`user`,我会先创建一个`user_model.dart`文件

## user_model.dart
```dart
class UserModel {
  int? id;
  String name;
  int age;
  String descs;

  UserModel({
    this.id,
    required this.name,
    required this.age,
    required this.descs,
  });
}
```

然后在创建一个叫`db_helper.dart`的文件,用于管理db文件的创建,路径之类的

## db_helper.dart

```dart
import 'package:flutter/foundation.dart';
import 'package:sqflite/sqflite.dart';

import 'dart:io';
import 'package:path_provider/path_provider.dart';
import 'dart:developer' as developer;

// DB文件配置
const _mmDBName = "MM.db";
const _mmDBVersion = 1;

// 定义其他的db文件
// const _shopDBName = "shop.db";
// const _shopDBVersion = 1;

class DBHelper {
  /// mmdb实例
  static Database? _mmDatabase;
  // static Database? _shopDatabase;

  /// 获取mm.db的Database
  static Future<Database> get mmdb async {
    // 如果存在则直接返回,虽然是单例,但是后面的都不用判断的.
    if (_mmDatabase != null) return _mmDatabase!;

    // 创建mm.db路径
    String mmPath = await _PrivateDBHelper.getMMDbPath(_mmDBName);
    _PrivateDBHelper.createDBFile(mmPath);

    // 连接db, 默认singleInstance=true,也就是返回的是单例,只要路径一致,就不会创建多个
    _mmDatabase = await openDatabase(
      mmPath,
      version: _mmDBVersion,
    );
    developer.log("_mmDatabase 的内存地址是：${developer.inspect(_mmDatabase)}");
    return _mmDatabase!;
  }

  // static Future<Database> get shopdb async {
  //   return ...;
  // }

  /// 关闭所有db
  static Future<void> closeAllDB() async {
    if (_mmDatabase != null) {
      await _mmDatabase!.close();
      _mmDatabase = null;
    }
    // if (_shopDatabase != null) {
    //   await _shopDatabase!.close();
    //   _shopDatabase = null;
    // }
  }
}

/// 私有静态方法
class _PrivateDBHelper {
  /// 创建.db文件
  static Future<bool> createDBFile(String dbPath) async {
    debugPrint("创建db文件:$dbPath");
    final file = File(dbPath);

    if (await file.exists()) {
      debugPrint("db已经存在");
      return false;
    }
    await file.create();
    debugPrint("db创建成功");
    return true;
  }

  /// 获取MM.db的文件路径
  static Future<String> getMMDbPath(String dbName) async {
    final directory = await getApplicationDocumentsDirectory();
    final path = '${directory.path}/$dbName';
    return path;
  }
}
```

我们可以看到主要提供2个简单的方法

* `mmdb` : 获取MM库的db对象
* `closeAllDB` : 断开所有db连接,这个通常是不需要调用的,因为如果你将进程杀掉或者crash之后,系统会进行资源回收清理工作.不过如果你确定不在需要对这个库进行操作时,需要主动调用一下,断开连接,这样会节省资源.比如你的db假设是根据登录用户身份来进行隔离的,即每个用户有自己对应的db文件时,当你切换用户,或者退出登录时,需要关闭之前的数据库连接.而这时可能你的app并没有杀掉当前进程.

简单整理一下,私有方法我们统一放在私有`_PrivateDBHelper`中,方便阅读.

## user_db_helper.dart

```dart
import 'package:flutter/foundation.dart';
import 'package:sqflite/sqflite.dart';
import 'package:test_flutter_animate/models/user_model.dart';
import 'package:test_flutter_animate/db_helper/db_helper.dart';


/// MM.db库中的'user'表管理器
/// 
/// 用于增删查改该表
/// ```dart
/// //创建表
/// await UserDBHelper.createTable();
/// 
/// // 事务插入数据
/// await UserDBHelper.insertUserModels(models);
/// 
/// // 查询数据
/// List<UserModel> models = await UserDBHelper.selectUserModels();
/// 
/// // 其他
/// // ...
/// ```
class UserDBHelper {
  // 定义表名,字段名等常量
  static const String _tableName = 'user';
  static const String _id = '_id';
  static const String _name = '_name';
  static const String _age = '_age';
  static const String _descs = '_descs';

  /// 创建USER表
  static Future<void> createTable() async {
    final db = await _getDB();
    await db.execute("""CREATE TABLE IF NOT EXISTS "$_tableName" (
          "$_id" INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL UNIQUE, 
          "$_name" VARCHAR, 
          "$_age" INTEGER, 
          "$_descs" TEXT)
      """);
  }

  /// 批量插入UserModel
  static Future<void> insertUserModels(List<UserModel> models) async {
    final db = await _getDB();
    db.transaction((txn) async {
      for (var i = 0; i < models.length; i++) {
        UserModel userModel = models[i];
        /**
         * 请使用"txn"进行操作,不要使用"db",否则会死锁.
         */
        int id = await txn.rawInsert(
            'INSERT INTO $_tableName($_name, $_age, $_descs) VALUES(?, ?, ?)',
            [userModel.name, userModel.age, userModel.descs]);
        debugPrint('inserted done: $id');
      }
    });
  }

  /// 批量更新UserModel
  static Future<void> updateUserModels(List<UserModel> models) async {
    final db = await _getDB();
    db.transaction((txn) async {
      for (var i = 0; i < models.length; i++) {
        UserModel userModel = models[i];
        int id = await txn.rawUpdate(
            'UPDATE $_tableName SET $_name = ?, $_age = ?, $_descs = ? WHERE $_id = ?',
            [userModel.name, userModel.age, userModel.descs, userModel.id]);
        debugPrint('updated done: $id');
      }
    });
  }

  /// 查询所有UserModel
  static Future<List<UserModel>> selectUserModels() async {
    final db = await _getDB();
    final results = await db.rawQuery('SELECT * FROM $_tableName');
    List<UserModel> list = [];
    for (var i = 0; i < results.length; i++) {
      Map map = results[i];
      UserModel userModel = UserModel(
        id: map[_id],
        name: map[_name],
        age: map[_age],
        descs: map[_descs],
      );
      list.add(userModel);
    }
    debugPrint("selected done!");
    return list;
  }

  /// 删除model
  static Future<void> deleteUserModels(List<UserModel> models) async {
    final db = await _getDB();
    db.transaction((txn) async {
      for (var i = 0; i < models.length; i++) {
        UserModel userModel = models[i];
        await txn.rawDelete(
            'DELETE FROM $_tableName WHERE $_id = ?', [userModel.id]);
      }
    });
    debugPrint("delete done!");
  }

  /// 删除USER表
  static Future<void> dropTable() async {
    final db = await _getDB();
    await db.execute('DROP TABLE $_tableName;');
    debugPrint('drop done!');
  }

  /// 获取该表对应的db
  static Future<Database> _getDB() async {
    final mmdb = await DBHelper.mmdb;
    return mmdb;
  }
}
```

以上只是实现了基础的增删查改,如果需要根据业务做出调整,可以加入自定义的方法,记得做测试.

项目中多个表就需要创建多个Model,和XXX_db_helper,按照这种思路进行扩展.

