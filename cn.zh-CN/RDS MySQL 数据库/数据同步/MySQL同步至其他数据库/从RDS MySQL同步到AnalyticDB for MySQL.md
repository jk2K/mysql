# 从RDS MySQL同步到AnalyticDB for MySQL

分析型数据库MySQL版（AnalyticDB for MySQL），是阿里巴巴自主研发的海量数据实时高并发在线分析（Realtime OLAP）云计算服务，使得您可以在毫秒级针对千亿级数据进行即时的多维分析透视和业务探索。通过数据传输服务DTS（Data Transmission Service），您可以将RDS MySQL同步到AnalyticDB for MySQL，帮助您快速构建企业内部BI、交互查询、实时报表等系统。

-   RDS MySQL中待同步的数据表必须具备主键。
-   已创建目标AnalyticDB for MySQL集群，详情请参见[创建AnalyticDB for MySQL（2.0）](https://help.aliyun.com/document_detail/98724.html)或[创建AnalyticDB for MySQL（3.0）](https://help.aliyun.com/document_detail/122234.html)。

    **说明：** 不支持将华北 1（青岛）、美国东部 1（弗吉尼亚）、英国（伦敦）地域的AnalyticDB MySQL（2.0）集群作为同步的目标集群；不支持将美国西部 1（硅谷）地域的AnalyticDB for MySQL（3.0）集群作为同步的目标集群。

-   确保目标AnalyticDB for MySQL具备充足的存储空间。
-   如果同步的目标为AnalyticDB for MySQL（2.0），那么源RDS MySQL待同步的对象不能包含AnalyticDB for MySQL（2.0）[保留的库名和列名](https://help.aliyun.com/document_detail/98800.html)，否则将造成数据同步失败或DDL操作同步失败。

## 优惠活动

[DTS优惠活动，最低0折](/cn.zh-CN/产品定价/产品定价.md)

## 注意事项

-   DTS在执行全量数据初始化时将占用源库和目标库一定的读写资源，可能会导致数据库的负载上升，在数据库性能较差、规格较低或业务量较大的情况下（例如源库有大量慢SQL、存在无主键表或目标库存在死锁等），可能会加重数据库压力，甚至导致数据库服务不可用。因此您需要在执行数据同步前评估源库和目标库的性能，同时建议您在业务低峰期执行数据同步（例如源库和目标库的CPU负载在30%以下）。
-   请勿在数据同步时，对源库的同步对象使用gh-ost或pt-online-schema-change等类似工具执行在线DDL变更，否则会导致同步失败。

    **说明：** 如果同步的目标为AnalyticDB for MySQL（3.0），您可以使用[DMS企业版](https://help.aliyun.com/document_detail/57278.html)提供的相关功能来执行在线DDL变更，详情请参见[不锁表结构变更](https://help.aliyun.com/document_detail/98373.html)。

-   由于AnalyticDB for MySQL（3.0）本身的使用限制，当AnalyticDB for MySQL（3.0）集群中的节点磁盘空间使用量超过80%，该集群将被锁定。请提前根据待同步的对象预估所需空间，确保目标集群具备充足的存储空间。
-   暂不支持同步前缀索引，如果源库存在前缀索引可能导致数据同步失败。

## 术语/概念对应关系

|MySQL|AnalyticDB for MySQL|
|-----|--------------------|
|数据库|-   AnalyticDB for MySQL（2.0）：表组
-   AnalyticDB for MySQL（3.0）：数据库 |
|表|-   AnalyticDB for MySQL（2.0）：表
-   AnalyticDB for MySQL（3.0）：表 |

**说明：** 关于AnalyticDB for MySQL中表组和表的相关介绍，请参见[基本概念]()。

## 支持同步的SQL操作

|目标数据库版本|支持的SQL操作|
|:------|:-------|
|AnalyticDB for MySQL 2.0|-   DDL操作：ADD COLUMN
-   DML操作：INSERT、UPDATE、DELETE |
|AnalyticDB for MySQL 3.0|-   DDL操作：CREATE TABLE、DROP TABLE、RENAME TABLE、TRUNCATE TABLE、ADD COLUMN、DROP COLUMN
-   DML操作：INSERT、UPDATE、DELETE

**说明：** 如果在数据同步的过程中变更了源表的字段类型，同步作业将报错并中断。您可以[提交工单](https://selfservice.console.aliyun.com/ticket/category/dts/today)处理或参照文末的方法来手动修复，详情请参见[修复因变更字段类型导致的同步失败](#section_o6l_gcd_cqe)。 |

## 数据库账号的权限要求

|数据库|所需权限|
|---|----|
|RDS MySQL|REPLICATION CLIENT、REPLICATION SLAVE、SHOW VIEW和所有同步对象的SELECT权限。|
|AnalyticDB for MySQL（2.0）|无需填写数据库账号信息，DTS会自动创建账号并授权。|
|AnalyticDB for MySQL（3.0）|读写权限。|

## 数据类型映射关系

由于MySQL和AnalyticDB for MySQL的数据类型并不是一一对应的，所以DTS在进行结构初始化时，会根据数据类型定义进行类型映射，详情请参见[结构初始化涉及的数据类型映射关系](/cn.zh-CN/数据同步/结构初始化涉及的数据类型映射关系.md)。

## 操作步骤

1.  购买数据同步作业，详情请参见[购买流程](/cn.zh-CN/快速入门/购买流程.md)。

    **说明：** 购买时，选择源实例为**MySQL**，目标实例为**分析型数据库AnalyticDB**，并选择同步拓扑为**单向同步**。

2.  登录[数据传输控制台](https://dts.console.aliyun.com/)。

3.  在左侧导航栏，单击**数据同步**。

4.  在同步作业列表页面顶部，选择数据同步实例所属地域。

    ![选择地域](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/7349459951/p50604.png)

5.  定位至已购买的数据同步实例，单击**配置同步链路**。

6.  配置同步通道的源实例及目标实例信息。

    ![源目实例信息配置](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/2520649951/p55263.png)

    |类别|配置|说明|
    |:-|:-|:-|
    |无|同步作业名称|DTS会自动生成一个同步作业名称，建议配置具有业务意义的名称（无唯一性要求），便于后续识别。|
    |源实例信息|实例类型|选择**RDS实例**。|
    |实例地区|购买数据同步实例时选择的源实例地域信息，不可变更。|
    |实例ID|选择源RDS实例ID。|
    |数据库账号|填入源RDS的数据库账号，权限要求请参见[数据库账号的权限要求](#section_w01_xz7_hp4)。 **说明：** 当源RDS实例的数据库类型为**MySQL 5.5**或**MySQL 5.6**时，无需配置**数据库账号**和**数据库密码**。 |
    |数据库密码|填入该数据库账号对应的密码。|
    |连接方式|根据需求选择**非加密连接**或**SSL安全连接**。如果设置为**SSL安全连接**，您需要提前开启RDS实例的SSL加密功能，详情请参见[设置SSL加密](https://help.aliyun.com/document_detail/96120.html)。|
    |目标实例信息|实例类型|固定为**ADS**，不可变更。|
    |实例地区|购买数据同步实例时选择的目标实例地域信息，不可变更。|
    |版本|根据目标AnalyticDB MySQL集群的版本，选择**2.0**或**3.0**。**说明：**

    -   选择为**2.0**后，无需配置**数据库账号**和**数据库密码**。
    -   选择为**3.0**后，您还需要配置**数据库账号**和**数据库密码**，权限要求请参见[数据库账号的权限要求](#section_w01_xz7_hp4)。 |
    |数据库|选择目标AnalyticDB for MySQL的集群ID。|
    |数据库账号|填入AnalyticDB for MySQL的数据库账号。|
    |数据库密码|填入该数据库账号对应的密码。|

7.  单击页面右下角的**授权白名单并进入下一步**。

8.  配置同步策略及对象信息。

    ![配置同步策略和对象](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/7030649951/p55267.png)

    |配置|说明|
    |:-|:-|
    |同步初始化|默认情况下，您需要同时选中**结构初始化**和**全量数据初始化**。预检查完成后，DTS会将源实例中待同步对象的结构及数据在目标集群中初始化，作为后续增量同步数据的基线数据。|
    |目标已存在表的处理模式|    -   **预检查并报错拦截**：检查目标数据库中是否有同名的表。如果目标数据库中没有同名的表，则通过该检查项目；如果目标数据库中有同名的表，则在预检查阶段提示错误，数据同步作业不会被启动。

**说明：** 如果目标库中同名的表不方便删除或重命名，您可以更改该表在目标库中的名称，详情请参见[设置同步对象在目标实例中的名称](/cn.zh-CN/数据同步/同步作业管理/设置同步对象在目标实例中的名称.md)。

    -   **忽略报错并继续执行**：跳过目标数据库中是否有同名表的检查项。

**警告：** 选择为**忽略报错并继续执行**，可能导致数据不一致，给业务带来风险，例如：

        -   表结构一致的情况下，在目标库遇到与源库主键的值相同的记录，则会保留目标集群中的该条记录，即源库中的该条记录不会同步至目标数据库中。
        -   表结构不一致的情况下，可能会导致无法初始化数据、只能同步部分列的数据或同步失败。 |
    |多表归并|    -   选择为**是**：DTS将在每个表中增加`__dts_data_source`列来存储数据来源，且不再支持DDL同步。
    -   选择为**否**：默认选项，支持DDL同步。
**说明：** 多表归并功能基于任务级别，即不支持基于表级别执行多表归并。如果需要让部分表执行多表归并，另一部分不执行多表归并，您可以创建两个数据同步作业。 |
    |同步操作类型|根据业务选中需要同步的操作类型，默认情况下都处于选中状态。 **说明：** 目前仅支持INSERT、UPDATE、DELETE、ADD COLUMN。 |
    |选择同步对象|在源库对象框中单击待同步的对象，然后单击![向右小箭头](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/8502659951/p40698.png)图标将其移动至已选择对象框。

同步对象的选择粒度为库、表。

**说明：**

    -   如果选择整个库作为同步对象，那么该库中所有对象的结构变更操作会同步至目标库。
    -   如果选择某个表作为同步对象，那么只有这个表的ADD COLUMN操作会同步至目标库。
    -   默认情况下，同步对象的名称保持不变。如果您需要同步对象在目标集群上名称不同，请使用对象名映射功能，详情请参见[设置同步对象在目标实例中的名称](/cn.zh-CN/数据同步/同步作业管理/设置同步对象在目标实例中的名称.md)。 |

9.  上述配置完成后，单击页面右下角的**下一步**。

10. 设置待同步的表在目标库中类型。

    ![设置表类型](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/6230649951/p55270.png)

    **说明：** 选择了**结构初始化**后，您需要定义待同步的表在AnalyticDB for MySQL中的**类型**、主**键列**、**分区列**等信息，详情请参见[ADB 2.0 SQL手册](https://help.aliyun.com/document_detail/94860.html)和[ADB 3.0 SQL手册](https://help.aliyun.com/document_detail/123333.html)。

11. 上述配置完成后，单击页面右下角的**预检查并启动**。

    **说明：**

    -   在数据同步作业正式启动之前，会先进行预检查。只有预检查通过后，才能成功启动数据同步作业。
    -   如果预检查失败，单击具体检查项后的![提示](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/8502659951/p47468.png)图标，查看失败详情。根据提示修复后，重新进行预检查。
12. 在预检查对话框中显示**预检查通过**后，关闭预检查对话框，同步作业将正式开始。

13. 等待同步作业的链路初始化完成，直至处于**同步中**状态。

    您可以在数据同步页面，查看数据同步作业的状态。

    ![数据同步状态](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/1349459951/p41059.png)


## 修复因变更字段类型导致的同步失败

本案例中，同步失败的表在AnalyticDB for MySQL中的名称为customer。

1.  在AnalyticDB for MySQL3.0中创建一个新表（customer\_new），表结构与customer表保持一致。

2.  通过INSERT INTO SELECT命令，将customer表的数据复制并插入到新创建的customer\_new表中，确保两张表的数据保持一致。

3.  重命名或删除同步失败的表，然后将customer\_new表的名称修改为customer。

4.  在DTS控制台，重新启动数据同步作业。


