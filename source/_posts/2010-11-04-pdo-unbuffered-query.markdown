---
comments: true
date: 2010-11-04 00:38:31
layout: post
title: 'PDO:  2014 Cannot execute queries while other unbuffered queries are active'
wordpress_id: 7
categories:  [web_dev]
tags:  [mysql, pdo, php]

---

注：以下结论均是在php 5.2.10+mysql 5.1.30环境下得出

出现此类PDO错误往往是pdo与mysql搭档时出现，可能原因如下:

1. **PDO->exec()执行select语句后（即使该select结果集为空），再执行PDO->query()、PDO->exec()、PDOStatement->execute()等会出现该异常**

如PHP manual所述，exec方法主要是用于执行一次sql命令，并返回该sql语句所影响的条目数量的，对于select语句，exec并不返回结果。对于只执行一次的sql select，建议使用PDO->query()，如果selec需要多次执行，则建议使用准备语句(PDOStatement->execute())。

对于mysql来说，在通过PDO->exec()执行一条select后，再次执行PDO相关数据操作，如PDO->query()、PDO->exec()、PDOStatement->execute()等均会抛出“unbuffered queries are active”的异常，如：

[php]
    $dbh = new PDO(DB_CON_STRING,DB_CON_USER,DB_CON_PASSWORD);
    $dbh -> setAttribute(PDO::ATTR_ERRMODE,PDO::ERRMODE_EXCEPTION);
    $sql = 'select handle_time from workflow where 0=1'; //注意:即使这里的select必定没有返回结果，后面也会报错
    $result = $dbh->exec($sql); //执行一次exec

    $sql = 'select handle_time from workflow where list_id=3000';
    //$ret = $dbh->query($sql); //此时执行query会报错
    //$ret = $dbh->exec($sql);//如果此时执行exec也会报错

    $i=3000;
    $sql = 'select handle_time from workflow where list_id=:id';
    $sth = $dbh->prepare($sql);
    $sth->bindParam(':id',$i);
    $ret = $sth->execute();//执行execute会报错
[/php]

在[YII](http://www.yiiframework.com/) 1.0.6版本的CDbLogRoute.php中就存在这个问题，该类是用来记录日志到DB的，其init初始化方法中存在判断log表是否存在时，使用的是如下代码：

[php]
/**
 * Initializes the route.
 * This method is invoked after the route is created by the route manager.
 */
 public function init()
 {
parent::init();

$db=$this->getDbConnection();
$db->setActive(true);

$sql="SELECT * FROM {$this->logTableName} WHERE 0=1";
try
{
$db->createCommand($sql)->execute(); //这里实际是调用的PDO->exec($sql)，
//可以改为$db->createCommand($sql)->queryAll(),而调用PDO->query($sql)来避免遗产的产生
}
catch(Exception $e)
{
// The log table does not exist
if($this->autoCreateLogTable)
$this->createLogTable($db,$this->logTableName);
else
throw new CException(Yii::t('yii','CDbLogRoute requires database table "{table}" to store log messages.',
array('{table}'=>$this->logTableName)));
}
 }
[/php]

而在真正记录日志时，使用的是如下的processLogs方法:

[php]
/**
 * Stores log messages into database.
 * @param array list of log messages
 */
 protected function processLogs($logs)
 {
$sql="
INSERT INTO {$this->logTableName}
(level, category, logtime, message) VALUES
(:level, :category, :logtime, :message)
";
$command=$this->getDbConnection()->createCommand($sql);
foreach($logs as $log)
{
$command->bindValue(':level',$log[1]);
$command->bindValue(':category',$log[2]);
$command->bindValue(':logtime',(int)$log[3]);
$command->bindValue(':message',$log[0]);
$command->execute();  //实际上是调用的PDOStatement->execute(),使用mysql时会报错
}
 }
[/php]

这样就会在执行$command->execute()记录日志时就会出现“unbuffered queries are active”的异常。在Yii 1.0.7中，qiang改变了init的处理逻辑，在判断表是否存在时，不再使用

[sql]

SELECT * FROM {$this->logTableName} WHERE 0=1[/sql]

了，而是采用了

[sql]DELETE FROM {$this->logTableName} WHERE 0=1[/sql]

,从而也就避免了该异常的产生。 但如果PDO->exec()中执行的是update、delete等sql语句，则后面的query、exec、execute不会报错。

2. **PDO::exec()中连续执行两条sql语句**（包括set names, update, insert等），也会在下一次query exec execute时抛该异常，

但网上有说php 5.3以上不会出现该问题，未验证。如

[php]
$sql = "set names utf8;update workflow set status=3 where list_id=3000";
$result = $dbh->exec($sql); //exec后，后面的exec query execute等会报出异常
[/php]

3. 另外，按照php manual中关于PDO::query 和 PDOStatement->closeCursor的说明，在没有完全fetchall一个result set时，再次使用query等就会出现问题:

If you do not fetch all of the data in a result set before issuing your next  call to [PDO->query()](function.pdo-query.html), your call may  fail.

Call [PDOStatement->closeCursor()](function.pdostatement-closecursor.html) to release the database resources associated with the PDOStatement object

before  issuing your next call to [PDO->query()](function.pdo-query.html).

但实际试验中没有该问题，只有以上两种情况下会出现此问题，不过为了规避潜在问题，最好还是在每次query后，可以fetchAll或closeCursor  ****

**总结：**



	
  1. 不要使用PDO::exec()执行select，也不要一次执行分号分隔的多条sql

	
  2. 各个PDO::query及PDOStatement::execute()后，最好是closeCursor或fetchAll（并设置null值）


