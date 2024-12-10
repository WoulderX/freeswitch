<style>
pre {
  overflow-y: auto;
  max-height: 350px;
}
</style>

### OpenSIPS+FreeSWITCH实现呼叫负载均衡
[TOC]
在实际生产环境中，FreeSWITCH为实现高并发，可以通过升级服务器配置来解决，除了该方法外，我们还可以通过多台FreeSWITCH负载均衡来提高并发数，使用OpenSIPS作为代理服务器，将呼叫路由至FreeSWITCH，并通过算法实现并发动态平衡。

---
#### 架构流程图
<img src="http://xinchenai.oss-cn-hangzhou.aliyuncs.com/freeswitch/freeswitch-imgs/opensips-freeswitch-flow.png" width="100%" />

---
#### 环境准备
>以下环境准备仅为示例(采用OpenSIPS单节点并于MySQL共部署)，实际部署过程中可根据具体情况进行变动，例如MySQL可单独部署等。

| IP   |      用途      |
|----------|:-------------:|
| 192.168.2.50 |  OpenSIPS + MySQL |
| 192.168.2.48 |    FreeSWITCH-01   |
| 192.168.2.49 | FreeSWITCH-02 |

>本文不做MySQL部署介绍，默认使用MySQL5.7。
---
#### 安装多台FreeSWITCH并确保通话正常
FreeSWITCH的安装步骤详见「[__《FreeSWITCH安装手册》__](https://codeup.aliyun.com/614a9324e43534781c03c140/cc/freeswitch/freeswitch-docs/blob/master/%E8%BF%90%E7%BB%B4%E6%89%8B%E5%86%8C/FreeSWITCH%E5%AE%89%E8%A3%85%E6%89%8B%E5%86%8C.md)」,本文仅介绍适配OpenSIPS进行负载均衡所需要变更的地方，可直接在已部署的FreeSWITCH上进行配置更改。

为了实现FreeSWITCH多台数据同步，需采用数据库存储用户、通话等信息，多台FreeSWITCH共用一台数据库，**否则会出现注册信息不同步，导致呼叫时FreeSWITCH会因为用户未注册而呼叫失败**。FreeSWITCH默认使用SQLite，需修改成MySQL或Postgre，本文以MySQL为例。
```bash
#安装mysql连接组件
yum -y install unixODBC unixODBC-devel mysql-connector-odbc

#编辑odbc文件，没有则新增
#注意修改配置文件中的数据库用户名密码
vim /etc/odbc.ini
[freeswitch]
Description=MySQL freeswitch database
Driver          = MySQL
SERVER          = 192.168.2.50
PORT            = 3306
DATABASE        = freeswitch
OPTION          = 67108864
PASSWORD        = 123456
CHARSET         = UTF8
```
登录MySQL实例，创建数据库freeswitch。
```sql
CREATE DATABASE `freeswitch` CHARACTER SET 'utf8mb4' COLLATE 'utf8mb4_general_ci';
```
测试数据库连接是否正常。
```bash
isql -v freeswitch
#显示如下即为正常
+---------------------------------------+
| Connected!                            |
| sql-statement                         |
| help [tablename]                      |
| quit                                  |
+---------------------------------------+
```
登录数据库，创建freeswitch相关数据表。
```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;
-- ----------------------------
-- Table structure for aliases
-- ----------------------------
DROP TABLE IF EXISTS `aliases`;
CREATE TABLE `aliases`  (
  `sticky` int(11) NULL DEFAULT NULL,
  `alias` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `command` varchar(4096) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `alias1`(`alias`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for calls
-- ----------------------------
DROP TABLE IF EXISTS `calls`;
CREATE TABLE `calls`  (
  `call_uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `call_created` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `call_created_epoch` int(11) NULL DEFAULT NULL,
  `caller_uuid` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `callee_uuid` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `callsidx1`(`hostname`) USING BTREE,
  INDEX `eruuindex`(`caller_uuid`, `hostname`) USING BTREE,
  INDEX `eeuuindex`(`callee_uuid`) USING BTREE,
  INDEX `eeuuindex2`(`call_uuid`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for cdr_table_a
-- ----------------------------
DROP TABLE IF EXISTS `cdr_table_a`;
CREATE TABLE `cdr_table_a`  (
  `uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '唯一ID',
  `call_uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '关联ID，同主叫方UUID',
  `caller_id_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '主叫方昵称',
  `caller_id_number` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '主叫号码',
  `destination_number` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT ' 被叫号码',
  `start_stamp` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '呼叫发起的日期/时间',
  `answer_stamp` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '实际应答呼叫远端的日期/时间 如果未接听电话，则为空字符串',
  `end_stamp` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '呼叫终止的日期/时间',
  `uduration` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '总呼叫持续时间（以微秒为单位）',
  `billsec` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '可计费的通话时长（秒）可计费时间不包括在远端接听电话之前在“早期媒体”中花费的通话时间',
  `hangup_cause` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '挂断原因',
  `sip_network_ip` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '网络IP',
  `depart_guid` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `recordfile` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '录音文件路径（含文件名称）',
  `recordname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '录音文件名称',
  INDEX `idx_uuid`(`uuid`) USING BTREE,
  INDEX `idx_caller_id_number`(`caller_id_number`) USING BTREE,
  INDEX `idx_destination_number`(`destination_number`) USING BTREE,
  INDEX `idx_start_stamp`(`start_stamp`) USING BTREE,
  INDEX `idx_recordname`(`recordname`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for cdr_table_ab
-- ----------------------------
DROP TABLE IF EXISTS `cdr_table_ab`;
CREATE TABLE `cdr_table_ab`  (
  `guid` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '唯一ID',
  `call_uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '关联ID，同主叫方UUID',
  `caller_id_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '主叫方昵称',
  `caller_id_number` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '主叫号码',
  `destination_number` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT ' 被叫号码',
  `start_stamp` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '呼叫发起的日期/时间',
  `answer_stamp` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '实际应答呼叫远端的日期/时间 如果未接听电话，则为空字符串',
  `end_stamp` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '呼叫终止的日期/时间',
  `uduration` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '总呼叫持续时间（以微秒为单位）',
  `billsec` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '可计费的通话时长（秒）可计费时间不包括在远端接听电话之前在“早期媒体”中花费的通话时间',
  `hangup_cause` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '挂断原因',
  `sip_network_ip` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '网络IP',
  `depart_guid` int(11) NULL DEFAULT NULL,
  `recordfile` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '录音文件路径（含文件名称）',
  `recordname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '录音文件名称',
  PRIMARY KEY (`guid`) USING BTREE,
  INDEX `idx_uuid`(`uuid`) USING BTREE,
  INDEX `idx_caller_id_number`(`caller_id_number`) USING BTREE,
  INDEX `idx_destination_number`(`destination_number`) USING BTREE,
  INDEX `idx_start_stamp`(`start_stamp`) USING BTREE,
  INDEX `idx_recordname`(`recordname`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 4819153 AVG_ROW_LENGTH = 1820 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for cdr_table_b
-- ----------------------------
DROP TABLE IF EXISTS `cdr_table_b`;
CREATE TABLE `cdr_table_b`  (
  `uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '唯一ID',
  `call_uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '关联ID，同主叫方UUID',
  `caller_id_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '主叫方昵称',
  `caller_id_number` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '主叫号码',
  `destination_number` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT ' 被叫号码',
  `start_stamp` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '呼叫发起的日期/时间',
  `answer_stamp` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '实际应答呼叫远端的日期/时间 如果未接听电话，则为空字符串',
  `end_stamp` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '呼叫终止的日期/时间',
  `uduration` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '总呼叫持续时间（以微秒为单位）',
  `billsec` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '可计费的通话时长（秒）可计费时间不包括在远端接听电话之前在“早期媒体”中花费的通话时间',
  `hangup_cause` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '挂断原因',
  `sip_network_ip` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '网络IP',
  `depart_guid` int(11) NULL DEFAULT NULL,
  `recordfile` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '录音文件路径（含文件名称）',
  `recordname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '录音文件名称',
  INDEX `idx_uuid`(`uuid`) USING BTREE,
  INDEX `idx_caller_id_number`(`caller_id_number`) USING BTREE,
  INDEX `idx_destination_number`(`destination_number`) USING BTREE,
  INDEX `idx_start_stamp`(`start_stamp`) USING BTREE,
  INDEX `idx_recordname`(`recordname`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for channels
-- ----------------------------
DROP TABLE IF EXISTS `channels`;
CREATE TABLE `channels`  (
  `uuid` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `direction` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `created` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `created_epoch` int(11) NULL DEFAULT NULL,
  `name` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `state` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `cid_name` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `cid_num` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `ip_addr` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `dest` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `application` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `application_data` text CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL,
  `dialplan` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `context` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `read_codec` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `read_rate` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `read_bit_rate` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `write_codec` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `write_rate` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `write_bit_rate` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `secure` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `presence_id` text CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL,
  `presence_data` text CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL,
  `accountcode` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `callstate` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `callee_name` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `callee_num` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `callee_direction` varchar(5) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `call_uuid` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `sent_callee_name` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `sent_callee_num` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `initial_cid_name` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `initial_cid_num` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `initial_ip_addr` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `initial_dest` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `initial_dialplan` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `initial_context` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `channels1`(`hostname`) USING BTREE,
  INDEX `chidx1`(`hostname`) USING BTREE,
  INDEX `uuindex`(`uuid`, `hostname`) USING BTREE,
  INDEX `uuindex2`(`call_uuid`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for complete
-- ----------------------------
DROP TABLE IF EXISTS `complete`;
CREATE TABLE `complete`  (
  `sticky` int(11) NULL DEFAULT NULL,
  `a1` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `a2` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `a3` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `a4` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `a5` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `a6` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `a7` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `a8` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `a9` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `a10` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `complete1`(`a1`, `hostname`) USING BTREE,
  INDEX `complete2`(`a2`, `hostname`) USING BTREE,
  INDEX `complete3`(`a3`, `hostname`) USING BTREE,
  INDEX `complete4`(`a4`, `hostname`) USING BTREE,
  INDEX `complete5`(`a5`, `hostname`) USING BTREE,
  INDEX `complete6`(`a6`, `hostname`) USING BTREE,
  INDEX `complete7`(`a7`, `hostname`) USING BTREE,
  INDEX `complete8`(`a8`, `hostname`) USING BTREE,
  INDEX `complete9`(`a9`, `hostname`) USING BTREE,
  INDEX `complete10`(`a10`, `hostname`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for db_data
-- ----------------------------
DROP TABLE IF EXISTS `db_data`;
CREATE TABLE `db_data`  (
  `hostname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `realm` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `data_key` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `data` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  UNIQUE INDEX `dd_data_key_realm`(`data_key`, `realm`) USING BTREE,
  INDEX `dd_realm`(`realm`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for fifo_bridge
-- ----------------------------
DROP TABLE IF EXISTS `fifo_bridge`;
CREATE TABLE `fifo_bridge`  (
  `fifo_name` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `caller_uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `caller_caller_id_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `caller_caller_id_number` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `consumer_uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `consumer_outgoing_uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `bridge_start` int(11) NULL DEFAULT NULL
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for fifo_callers
-- ----------------------------
DROP TABLE IF EXISTS `fifo_callers`;
CREATE TABLE `fifo_callers`  (
  `fifo_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `caller_caller_id_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `caller_caller_id_number` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `timestamp` int(11) NULL DEFAULT NULL
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for fifo_outbound
-- ----------------------------
DROP TABLE IF EXISTS `fifo_outbound`;
CREATE TABLE `fifo_outbound`  (
  `uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `fifo_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `originate_string` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `simo_count` int(11) NULL DEFAULT NULL,
  `use_count` int(11) NULL DEFAULT NULL,
  `timeout` int(11) NULL DEFAULT NULL,
  `lag` int(11) NULL DEFAULT NULL,
  `next_avail` int(11) NOT NULL DEFAULT 0,
  `expires` int(11) NOT NULL DEFAULT 0,
  `static` int(11) NOT NULL DEFAULT 0,
  `outbound_call_count` int(11) NOT NULL DEFAULT 0,
  `outbound_fail_count` int(11) NOT NULL DEFAULT 0,
  `hostname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `taking_calls` int(11) NOT NULL DEFAULT 1,
  `status` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `outbound_call_total_count` int(11) NOT NULL DEFAULT 0,
  `outbound_fail_total_count` int(11) NOT NULL DEFAULT 0,
  `active_time` int(11) NOT NULL DEFAULT 0,
  `inactive_time` int(11) NOT NULL DEFAULT 0,
  `manual_calls_out_count` int(11) NOT NULL DEFAULT 0,
  `manual_calls_in_count` int(11) NOT NULL DEFAULT 0,
  `manual_calls_out_total_count` int(11) NOT NULL DEFAULT 0,
  `manual_calls_in_total_count` int(11) NOT NULL DEFAULT 0,
  `ring_count` int(11) NOT NULL DEFAULT 0,
  `start_time` int(11) NOT NULL DEFAULT 0,
  `stop_time` int(11) NOT NULL DEFAULT 0
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for group_data
-- ----------------------------
DROP TABLE IF EXISTS `group_data`;
CREATE TABLE `group_data`  (
  `hostname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `groupname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `url` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `gd_groupname`(`groupname`) USING BTREE,
  INDEX `gd_url`(`url`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for interfaces
-- ----------------------------
DROP TABLE IF EXISTS `interfaces`;
CREATE TABLE `interfaces`  (
  `type` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `name` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `description` varchar(4096) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `ikey` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `filename` varchar(4096) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `syntax` varchar(4096) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for limit_data
-- ----------------------------
DROP TABLE IF EXISTS `limit_data`;
CREATE TABLE `limit_data`  (
  `hostname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `realm` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `id` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `ld_hostname`(`hostname`) USING BTREE,
  INDEX `ld_uuid`(`uuid`) USING BTREE,
  INDEX `ld_realm`(`realm`) USING BTREE,
  INDEX `ld_id`(`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for nat
-- ----------------------------
DROP TABLE IF EXISTS `nat`;
CREATE TABLE `nat`  (
  `sticky` int(11) NULL DEFAULT NULL,
  `port` int(11) NULL DEFAULT NULL,
  `proto` int(11) NULL DEFAULT NULL,
  `hostname` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `nat_map_port_proto`(`port`, `proto`, `hostname`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for recovery
-- ----------------------------
DROP TABLE IF EXISTS `recovery`;
CREATE TABLE `recovery`  (
  `runtime_uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `technology` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `profile_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `metadata` text CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL,
  INDEX `recovery1`(`technology`) USING BTREE,
  INDEX `recovery2`(`profile_name`) USING BTREE,
  INDEX `recovery3`(`uuid`) USING BTREE,
  INDEX `recovery4`(`runtime_uuid`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for registrations
-- ----------------------------
DROP TABLE IF EXISTS `registrations`;
CREATE TABLE `registrations`  (
  `reg_user` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `realm` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `token` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `url` text CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL,
  `expires` int(11) NULL DEFAULT NULL,
  `network_ip` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `network_port` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `network_proto` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `metadata` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `regindex1`(`reg_user`, `realm`, `hostname`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for sip_authentication
-- ----------------------------
DROP TABLE IF EXISTS `sip_authentication`;
CREATE TABLE `sip_authentication`  (
  `nonce` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `expires` bigint(20) NULL DEFAULT NULL,
  `profile_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `last_nc` int(11) NULL DEFAULT NULL,
  `algorithm` int(11) NOT NULL DEFAULT 1,
  INDEX `sa_nonce`(`nonce`) USING BTREE,
  INDEX `sa_hostname`(`hostname`) USING BTREE,
  INDEX `sa_expires`(`expires`) USING BTREE,
  INDEX `sa_last_nc`(`last_nc`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for sip_dialogs
-- ----------------------------
DROP TABLE IF EXISTS `sip_dialogs`;
CREATE TABLE `sip_dialogs`  (
  `call_id` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `sip_to_user` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `sip_to_host` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `sip_from_user` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `sip_from_host` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `contact_user` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `contact_host` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `state` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `direction` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `user_agent` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `profile_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `contact` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `presence_id` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `presence_data` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `call_info` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `call_info_state` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT '',
  `expires` bigint(20) NULL DEFAULT 0,
  `status` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `rpid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `sip_to_tag` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `sip_from_tag` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `rcd` int(11) NOT NULL DEFAULT 0,
  INDEX `sd_uuid`(`uuid`) USING BTREE,
  INDEX `sd_hostname`(`hostname`) USING BTREE,
  INDEX `sd_presence_data`(`presence_data`) USING BTREE,
  INDEX `sd_call_info`(`call_info`) USING BTREE,
  INDEX `sd_call_info_state`(`call_info_state`) USING BTREE,
  INDEX `sd_expires`(`expires`) USING BTREE,
  INDEX `sd_rcd`(`rcd`) USING BTREE,
  INDEX `sd_sip_to_tag`(`sip_to_tag`) USING BTREE,
  INDEX `sd_sip_from_user`(`sip_from_user`) USING BTREE,
  INDEX `sd_sip_from_host`(`sip_from_host`) USING BTREE,
  INDEX `sd_sip_to_host`(`sip_to_host`) USING BTREE,
  INDEX `sd_presence_id`(`presence_id`) USING BTREE,
  INDEX `sd_call_id`(`call_id`) USING BTREE,
  INDEX `sd_sip_from_tag`(`sip_from_tag`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for sip_presence
-- ----------------------------
DROP TABLE IF EXISTS `sip_presence`;
CREATE TABLE `sip_presence`  (
  `sip_user` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `sip_host` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `status` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `rpid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `expires` bigint(20) NULL DEFAULT NULL,
  `user_agent` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `profile_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `network_ip` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `network_port` varchar(6) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `open_closed` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `sp_hostname`(`hostname`) USING BTREE,
  INDEX `sp_open_closed`(`open_closed`) USING BTREE,
  INDEX `sp_sip_user`(`sip_user`) USING BTREE,
  INDEX `sp_sip_host`(`sip_host`) USING BTREE,
  INDEX `sp_profile_name`(`profile_name`) USING BTREE,
  INDEX `sp_expires`(`expires`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for sip_registrations
-- ----------------------------
DROP TABLE IF EXISTS `sip_registrations`;
CREATE TABLE `sip_registrations`  (
  `id` bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT,
  `call_id` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `sip_user` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `sip_host` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `presence_hosts` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `contact` varchar(1024) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `status` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `ping_status` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `ping_count` int(11) NULL DEFAULT NULL,
  `ping_time` bigint(20) NULL DEFAULT NULL,
  `force_ping` int(11) NULL DEFAULT NULL,
  `rpid` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `expires` bigint(20) NULL DEFAULT NULL,
  `ping_expires` int(11) NOT NULL DEFAULT 0,
  `user_agent` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `server_user` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `server_host` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `profile_name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `hostname` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `network_ip` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `network_port` varchar(6) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `sip_username` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `sip_realm` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `mwi_user` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `mwi_host` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `orig_server_host` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `orig_hostname` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `sub_host` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `sr_call_id`(`call_id`) USING BTREE,
  INDEX `sr_sip_user`(`sip_user`) USING BTREE,
  INDEX `sr_sip_host`(`sip_host`) USING BTREE,
  INDEX `sr_sub_host`(`sub_host`) USING BTREE,
  INDEX `sr_mwi_user`(`mwi_user`) USING BTREE,
  INDEX `sr_mwi_host`(`mwi_host`) USING BTREE,
  INDEX `sr_profile_name`(`profile_name`) USING BTREE,
  INDEX `sr_presence_hosts`(`presence_hosts`) USING BTREE,
  INDEX `sr_expires`(`expires`) USING BTREE,
  INDEX `sr_ping_expires`(`ping_expires`) USING BTREE,
  INDEX `sr_hostname`(`hostname`) USING BTREE,
  INDEX `sr_status`(`status`) USING BTREE,
  INDEX `sr_ping_status`(`ping_status`) USING BTREE,
  INDEX `sr_network_ip`(`network_ip`) USING BTREE,
  INDEX `sr_network_port`(`network_port`) USING BTREE,
  INDEX `sr_sip_username`(`sip_username`) USING BTREE,
  INDEX `sr_sip_realm`(`sip_realm`) USING BTREE,
  INDEX `sr_orig_server_host`(`orig_server_host`) USING BTREE,
  INDEX `sr_orig_hostname`(`orig_hostname`) USING BTREE,
  INDEX `sr_contact`(`contact`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 26370 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for sip_shared_appearance_dialogs
-- ----------------------------
DROP TABLE IF EXISTS `sip_shared_appearance_dialogs`;
CREATE TABLE `sip_shared_appearance_dialogs`  (
  `profile_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `contact_str` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `call_id` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `network_ip` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `expires` bigint(20) NULL DEFAULT NULL,
  INDEX `ssd_profile_name`(`profile_name`) USING BTREE,
  INDEX `ssd_hostname`(`hostname`) USING BTREE,
  INDEX `ssd_contact_str`(`contact_str`) USING BTREE,
  INDEX `ssd_call_id`(`call_id`) USING BTREE,
  INDEX `ssd_expires`(`expires`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for sip_shared_appearance_subscriptions
-- ----------------------------
DROP TABLE IF EXISTS `sip_shared_appearance_subscriptions`;
CREATE TABLE `sip_shared_appearance_subscriptions`  (
  `subscriber` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `call_id` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `aor` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `profile_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `hostname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `contact_str` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `network_ip` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `ssa_hostname`(`hostname`) USING BTREE,
  INDEX `ssa_network_ip`(`network_ip`) USING BTREE,
  INDEX `ssa_subscriber`(`subscriber`) USING BTREE,
  INDEX `ssa_profile_name`(`profile_name`) USING BTREE,
  INDEX `ssa_aor`(`aor`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for sip_subscriptions
-- ----------------------------
DROP TABLE IF EXISTS `sip_subscriptions`;
CREATE TABLE `sip_subscriptions`  (
  `id` bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT,
  `proto` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `sip_user` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `sip_host` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `sub_to_user` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `sub_to_host` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `presence_hosts` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `event` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `contact` varchar(1024) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `call_id` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `full_from` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `full_via` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `expires` bigint(20) NULL DEFAULT NULL,
  `user_agent` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `accept` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `profile_name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `hostname` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `network_port` varchar(6) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `network_ip` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `version` int(11) NOT NULL DEFAULT 0,
  `orig_proto` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `full_to` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `ss_call_id`(`call_id`) USING BTREE,
  INDEX `ss_multi`(`call_id`, `profile_name`, `hostname`) USING BTREE,
  INDEX `ss_hostname`(`hostname`) USING BTREE,
  INDEX `ss_network_ip`(`network_ip`) USING BTREE,
  INDEX `ss_sip_user`(`sip_user`) USING BTREE,
  INDEX `ss_sip_host`(`sip_host`) USING BTREE,
  INDEX `ss_presence_hosts`(`presence_hosts`) USING BTREE,
  INDEX `ss_event`(`event`) USING BTREE,
  INDEX `ss_proto`(`proto`) USING BTREE,
  INDEX `ss_sub_to_user`(`sub_to_user`) USING BTREE,
  INDEX `ss_sub_to_host`(`sub_to_host`) USING BTREE,
  INDEX `ss_expires`(`expires`) USING BTREE,
  INDEX `ss_orig_proto`(`orig_proto`) USING BTREE,
  INDEX `ss_network_port`(`network_port`) USING BTREE,
  INDEX `ss_profile_name`(`profile_name`) USING BTREE,
  INDEX `ss_version`(`version`) USING BTREE,
  INDEX `ss_full_from`(`full_from`) USING BTREE,
  INDEX `ss_contact`(`contact`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for tasks
-- ----------------------------
DROP TABLE IF EXISTS `tasks`;
CREATE TABLE `tasks`  (
  `task_id` int(11) NULL DEFAULT NULL,
  `task_desc` varchar(4096) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `task_group` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `task_runtime` bigint(20) NULL DEFAULT NULL,
  `task_sql_manager` int(11) NULL DEFAULT NULL,
  `hostname` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `tasks1`(`hostname`, `task_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for user
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user`  (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user` varchar(10) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '分机号',
  `password` varchar(30) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '分机号密码',
  `comment` longtext CHARACTER SET utf8 COLLATE utf8_general_ci NULL COMMENT '分机号描述',
  `create_time` timestamp(0) NULL DEFAULT NULL COMMENT '分机号创建时间',
  `update_time` timestamp(0) NULL DEFAULT NULL COMMENT '分机号更新时间',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `user`(`user`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 87 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for voicemail_msgs
-- ----------------------------
DROP TABLE IF EXISTS `voicemail_msgs`;
CREATE TABLE `voicemail_msgs`  (
  `created_epoch` int(11) NULL DEFAULT NULL,
  `read_epoch` int(11) NULL DEFAULT NULL,
  `username` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `domain` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `uuid` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `cid_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `cid_number` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `in_folder` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `file_path` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `message_len` int(11) NULL DEFAULT NULL,
  `flags` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `read_flags` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `forwarded_by` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `voicemail_msgs_idx1`(`created_epoch`) USING BTREE,
  INDEX `voicemail_msgs_idx2`(`username`) USING BTREE,
  INDEX `voicemail_msgs_idx3`(`domain`) USING BTREE,
  INDEX `voicemail_msgs_idx4`(`uuid`) USING BTREE,
  INDEX `voicemail_msgs_idx5`(`in_folder`) USING BTREE,
  INDEX `voicemail_msgs_idx6`(`read_flags`) USING BTREE,
  INDEX `voicemail_msgs_idx7`(`forwarded_by`) USING BTREE,
  INDEX `voicemail_msgs_idx8`(`read_epoch`) USING BTREE,
  INDEX `voicemail_msgs_idx9`(`flags`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- Table structure for voicemail_prefs
-- ----------------------------
DROP TABLE IF EXISTS `voicemail_prefs`;
CREATE TABLE `voicemail_prefs`  (
  `username` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `domain` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `name_path` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `greeting_path` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `password` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  INDEX `voicemail_prefs_idx1`(`username`) USING BTREE,
  INDEX `voicemail_prefs_idx2`(`domain`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
-- ----------------------------
-- View structure for basic_calls
-- ----------------------------
DROP VIEW IF EXISTS `basic_calls`;
CREATE ALGORITHM = UNDEFINED SQL SECURITY DEFINER VIEW `basic_calls` AS select `a`.`uuid` AS `uuid`,`a`.`direction` AS `direction`,`a`.`created` AS `created`,`a`.`created_epoch` AS `created_epoch`,`a`.`name` AS `name`,`a`.`state` AS `state`,`a`.`cid_name` AS `cid_name`,`a`.`cid_num` AS `cid_num`,`a`.`ip_addr` AS `ip_addr`,`a`.`dest` AS `dest`,`a`.`presence_id` AS `presence_id`,`a`.`presence_data` AS `presence_data`,`a`.`accountcode` AS `accountcode`,`a`.`callstate` AS `callstate`,`a`.`callee_name` AS `callee_name`,`a`.`callee_num` AS `callee_num`,`a`.`callee_direction` AS `callee_direction`,`a`.`call_uuid` AS `call_uuid`,`a`.`hostname` AS `hostname`,`a`.`sent_callee_name` AS `sent_callee_name`,`a`.`sent_callee_num` AS `sent_callee_num`,`b`.`uuid` AS `b_uuid`,`b`.`direction` AS `b_direction`,`b`.`created` AS `b_created`,`b`.`created_epoch` AS `b_created_epoch`,`b`.`name` AS `b_name`,`b`.`state` AS `b_state`,`b`.`cid_name` AS `b_cid_name`,`b`.`cid_num` AS `b_cid_num`,`b`.`ip_addr` AS `b_ip_addr`,`b`.`dest` AS `b_dest`,`b`.`presence_id` AS `b_presence_id`,`b`.`presence_data` AS `b_presence_data`,`b`.`accountcode` AS `b_accountcode`,`b`.`callstate` AS `b_callstate`,`b`.`callee_name` AS `b_callee_name`,`b`.`callee_num` AS `b_callee_num`,`b`.`callee_direction` AS `b_callee_direction`,`b`.`sent_callee_name` AS `b_sent_callee_name`,`b`.`sent_callee_num` AS `b_sent_callee_num`,`c`.`call_created_epoch` AS `call_created_epoch` from ((`channels` `a` left join `calls` `c` on(((`a`.`uuid` = `c`.`caller_uuid`) and (`a`.`hostname` = `c`.`hostname`)))) left join `channels` `b` on(((`b`.`uuid` = `c`.`callee_uuid`) and (`b`.`hostname` = `c`.`hostname`)))) where ((`a`.`uuid` = `c`.`caller_uuid`) or (not(`a`.`uuid` in (select `calls`.`callee_uuid` from `calls`))));
-- ----------------------------
-- View structure for detailed_calls
-- ----------------------------
DROP VIEW IF EXISTS `detailed_calls`;
CREATE ALGORITHM = UNDEFINED SQL SECURITY DEFINER VIEW `detailed_calls` AS select `a`.`uuid` AS `uuid`,`a`.`direction` AS `direction`,`a`.`created` AS `created`,`a`.`created_epoch` AS `created_epoch`,`a`.`name` AS `name`,`a`.`state` AS `state`,`a`.`cid_name` AS `cid_name`,`a`.`cid_num` AS `cid_num`,`a`.`ip_addr` AS `ip_addr`,`a`.`dest` AS `dest`,`a`.`application` AS `application`,`a`.`application_data` AS `application_data`,`a`.`dialplan` AS `dialplan`,`a`.`context` AS `context`,`a`.`read_codec` AS `read_codec`,`a`.`read_rate` AS `read_rate`,`a`.`read_bit_rate` AS `read_bit_rate`,`a`.`write_codec` AS `write_codec`,`a`.`write_rate` AS `write_rate`,`a`.`write_bit_rate` AS `write_bit_rate`,`a`.`secure` AS `secure`,`a`.`hostname` AS `hostname`,`a`.`presence_id` AS `presence_id`,`a`.`presence_data` AS `presence_data`,`a`.`accountcode` AS `accountcode`,`a`.`callstate` AS `callstate`,`a`.`callee_name` AS `callee_name`,`a`.`callee_num` AS `callee_num`,`a`.`callee_direction` AS `callee_direction`,`a`.`call_uuid` AS `call_uuid`,`a`.`sent_callee_name` AS `sent_callee_name`,`a`.`sent_callee_num` AS `sent_callee_num`,`b`.`uuid` AS `b_uuid`,`b`.`direction` AS `b_direction`,`b`.`created` AS `b_created`,`b`.`created_epoch` AS `b_created_epoch`,`b`.`name` AS `b_name`,`b`.`state` AS `b_state`,`b`.`cid_name` AS `b_cid_name`,`b`.`cid_num` AS `b_cid_num`,`b`.`ip_addr` AS `b_ip_addr`,`b`.`dest` AS `b_dest`,`b`.`application` AS `b_application`,`b`.`application_data` AS `b_application_data`,`b`.`dialplan` AS `b_dialplan`,`b`.`context` AS `b_context`,`b`.`read_codec` AS `b_read_codec`,`b`.`read_rate` AS `b_read_rate`,`b`.`read_bit_rate` AS `b_read_bit_rate`,`b`.`write_codec` AS `b_write_codec`,`b`.`write_rate` AS `b_write_rate`,`b`.`write_bit_rate` AS `b_write_bit_rate`,`b`.`secure` AS `b_secure`,`b`.`hostname` AS `b_hostname`,`b`.`presence_id` AS `b_presence_id`,`b`.`presence_data` AS `b_presence_data`,`b`.`accountcode` AS `b_accountcode`,`b`.`callstate` AS `b_callstate`,`b`.`callee_name` AS `b_callee_name`,`b`.`callee_num` AS `b_callee_num`,`b`.`callee_direction` AS `b_callee_direction`,`b`.`call_uuid` AS `b_call_uuid`,`b`.`sent_callee_name` AS `b_sent_callee_name`,`b`.`sent_callee_num` AS `b_sent_callee_num`,`c`.`call_created_epoch` AS `call_created_epoch` from ((`channels` `a` left join `calls` `c` on(((`a`.`uuid` = `c`.`caller_uuid`) and (`a`.`hostname` = `c`.`hostname`)))) left join `channels` `b` on(((`b`.`uuid` = `c`.`callee_uuid`) and (`b`.`hostname` = `c`.`hostname`)))) where ((`a`.`uuid` = `c`.`caller_uuid`) or (not(`a`.`uuid` in (select `calls`.`callee_uuid` from `calls`))));
 
SET FOREIGN_KEY_CHECKS = 1;
```
修改FreeSWITCH相关数据库配置。
```bash
#注意修改配置文件中的数据库用户名密码
cd /usr/local/freeswitch/etc/freeswitch/
vim autoload_configs/db.conf.xml
<param name="odbc-dsn" value="freeswitch:root:123456"/>

vim autoload_configs/switch.conf.xml
<param name="core-db-dsn" value="freeswitch:root:123456" />
<param name="odbc-dsn" value="freeswitch:root:123456"/>

vim autoload_configs/voicemail.conf.xml
<param name="odbc-dsn" value="freeswitch:root:123456"/>

vim autoload_configs/callcenter.conf.xml
<param name="odbc-dsn" value="freeswitch:root:123456"/>

vim sip_profiles/external.xml
<param name="odbc-dsn" value="freeswitch:root:123456"/>

vim sip_profiles/internal.xml
<param name="odbc-dsn" value="freeswitch:root:123456"/>

vim autoload_configs/fifo.conf.xml
<settings>
    <param name="delete-all-outbound-member-on-startup" value="false"/>
    <param name="odbc-dsn" value="freeswitch:root:123456"/>
</settings>

vim vars.xml
<X-PRE-PROCESS cmd="set" data="json_db_handle=odbc://freeswitch:root:123456"/>
```
修改lua配置。
```bash
vim /usr/local/freeswitch/etc/freeswitch/autoload_configs/lua.conf.xml
<param name="script-directory" value="$${script_dir}/?.lua"/>
<param name="xml-handler-script" value="gen_dir_user_xml.lua" />
<param name="xml-handler-bindings" value="directory" />
```

创建lua脚本管理用户注册验证。
```bash
#注意修改配置文件中的数据库用户名密码
vi /usr/local/freeswitch/share/freeswitch/scripts/gen_dir_user_xml.lua

freeswitch.consoleLog("NOTICE","lua take the users\r\n");
local req_domain = params:getHeader("domain")
local req_key    = params:getHeader("key")
local req_user   = params:getHeader("user")
local req_password = params:getHeader("pass")
local dbh = freeswitch.Dbh("freeswitch","root","123456");
if dbh:connected() == false then
  freeswitch.consoleLog("NOTICE", "gen_dir_user_xml.lua cannot connect to database\n")
  return
end
if req_user ~= nil then
    freeswitch.consoleLog("NOTICE","lua take the users:"..req_user.."\r\n");
    freeswitch.consoleLog("NOTICE","req_user..."..req_user.."\r\n");
    local my_query = "select * from tb_sip_account where status = 0 and account="..req_user.." limit 1;";
    freeswitch.consoleLog("NOTICE", "the query string is:"..my_query.."\n")
    dbh:query(my_query,function(u)
       if u == nil then
          freeswitch.consoleLog("NOTICE", "user not exist\n")
          return
       end
       req_password=u.password
    end);
    dbh:release();
    freeswitch.consoleLog("NOTICE","info:"..req_domain.."--"..req_key.."--"..req_user.."\n");
    --assert (req_domain and req_key and req_user,
    --"This example script only supports generating directory xml for a single user !\n")
    if req_domain ~= nil and req_key~=nil and req_user~=nil and req_password~=nil then
       XML_STRING =
       [[<?xml version="1.0" encoding="UTF-8" standalone="no"?>
       <document type="freeswitch/xml">
         <section name="directory">
           <domain name="]]..req_domain..[[">
             <params>
           <param name="dial-string"
           value="{presence_id=${dialed_user}@${dialed_domain}}${sofia_contact(${dialed_user}@${dialed_domain})}"/>
             </params>
             <groups>
           <group name="default">
             <users>
               <user id="]] ..req_user..[[">
                 <params>
               <param name="password" value="]]..req_password..[["/>
               <param name="vm-password" value="]]..req_password..[["/>
                 </params>
                 <variables>
               <variable name="toll_allow" value="domestic,international,local"/>
               <variable name="accountcode" value="]] ..req_user..[["/>
               <variable name="user_context" value="default"/>
               <variable name="directory-visible" value="true"/>
               <variable name="directory-exten-visible" value="true"/>
               <variable name="limit_max" value="15"/>
               <variable name="effective_caller_id_name" value="Extension ]] ..req_user..[["/>
               <variable name="effective_caller_id_number" value="]] ..req_user..[["/>
               <variable name="outbound_caller_id_name" value="${outbound_caller_name}"/>
               <variable name="outbound_caller_id_number" value="${outbound_caller_id}"/>
               <variable name="callgroup" value="techsupport"/>
                 </variables>
               </user>
             </users>
           </group>
             </groups>
           </domain>
         </section>
       </document>]]
    else
       XML_STRING =
       [[<?xml version="1.0" encoding="UTF-8" standalone="no"?>
       <document type="freeswitch/xml">
         <section name="directory">
         </section>
       </document>]]
    end
end
-- comment the following line for production:
--freeswitch.consoleLog("notice", "Debug from init_user_xml.lua, generated XML:\n" .. XML_STRING .. "\n");
```
删除默认注册文件配置内容，使通过xml验证用户功能失效，默认让lua接管用户注册。
```bash
vim /usr/local/freeswitch/etc/freeswitch/directory/default.xml
#删除如下内容
<group name="default">
    <users>
      <X-PRE-PROCESS cmd="include" data="default/*.xml"/>
    </users>
 </group>
```
修改拨号计划，使所有用户都可以注册呼叫。
```xml
vim /usr/local/freeswitch/etc/freeswitch/dialplan/public.xml
    <extension name="public_extensions">
      <!--
      <condition field="destination_number" expression="^(10[01][0-9])$">
      -->
    <condition field="destination_number" expression="^(.*)$">
    <action application="transfer" data="$1 XML default"/>
      </condition>
    </extension>
```
登录数据库，创建用户表并新增用户。
```sql
CREATE TABLE `freeswitch1`.`tb_sip_account`  (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `account` varchar(10) NULL,
  `password` varchar(30) NULL,
  `status` int(2) NULL,
  PRIMARY KEY (`id`)
)

insert into tb_sip_account (account,password,status) values('6001','1234', 0);
insert into tb_sip_account (account,password,status) values('6002','1234', 0)
insert into tb_sip_account (account,password,status) values('6003','1234', 0);
insert into tb_sip_account (account,password,status) values('6004','1234', 0)
```
根据创建用户，新建拨号计划。
```bash
vim /usr/local/freeswitch/etc/freeswitch/dialplan/default.xml
    <extension name="Local_Extension">
      <condition field="destination_number" expression="^(60[01][0-9])$">
        <action application="export" data="dialed_extension=$1"/>
        <!-- bind_meta_app can have these args <key> [a|b|ab] [a|b|o|s] <app> -->
        <action application="bind_meta_app" data="1 b s execute_extension::dx XML features"/>
        <action application="bind_meta_app" data="2 b s record_session::$${recordings_dir}/${caller_id_number}.${strftime(%Y-%m-%d-%H-%M-%S)}.wav"/>
        <action application="bind_meta_app" data="3 b s execute_extension::cf XML features"/>
        <action application="bind_meta_app" data="4 b s execute_extension::att_xfer XML features"/>
        <action application="set" data="ringback=${us-ring}"/>
        <action application="set" data="transfer_ringback=$${hold_music}"/>
        <action application="set" data="call_timeout=30"/>
        <!-- <action application="set" data="sip_exclude_contact=${network_addr}"/> -->
        <action application="set" data="hangup_after_bridge=true"/>
        <!--<action application="set" data="continue_on_fail=NORMAL_TEMPORARY_FAILURE,USER_BUSY,NO_ANSWER,TIMEOUT,NO_ROUTE_DESTINATION"/> -->
        <action application="set" data="continue_on_fail=true"/>
        <action application="hash" data="insert/${domain_name}-call_return/${dialed_extension}/${caller_id_number}"/>
        <action application="hash" data="insert/${domain_name}-last_dial_ext/${dialed_extension}/${uuid}"/>
        <action application="set" data="called_party_callgroup=${user_data(${dialed_extension}@${domain_name} var callgroup)}"/>
        <action application="hash" data="insert/${domain_name}-last_dial_ext/${called_party_callgroup}/${uuid}"/>
        <action application="hash" data="insert/${domain_name}-last_dial_ext/global/${uuid}"/>
        <!--<action application="export" data="nolocal:rtp_secure_media=${user_data(${dialed_extension}@${domain_name} var rtp_secure_media)}"/>-->
        <action application="hash" data="insert/${domain_name}-last_dial/${called_party_callgroup}/${uuid}"/>
        <action application="bridge" data="user/${dialed_extension}@${domain_name}"/>
        <action application="answer"/>
        <action application="sleep" data="1000"/>
        <action application="bridge" data="loopback/app=voicemail:default ${domain_name} ${dialed_extension}"/>
      </condition>
    </extension>

```
**重启FreeSWITCH使上述各配置生效，并测试直接注册呼叫正常即可。**
>**注意：修改完成后，默认xml注册用户方式失效，后续新增用户均需在数据库中添加。**

---
#### 安装配置OpenSIPS
##### Docket部署方法
已将OpenSIPS 2.4.6与MySQL 5.7打包为Docker镜像，可直接部署后添加调度节点即可。
```bash
#已将OpenSIPS和MySQL配置为系统任务并开机自启
docker pull woulder97/opensips:2.4.6
docker run -it --name=opensips --network=host -v /sys/fs/cgroup:/sys/fs/cgroup --privileged=true -d woulder97/opensips:2.4.6 /usr/sbin/init
```
容器部署完成后，参照二进制部署方法中<添加负载节点>内容进行FreeSWITCH节点新增后，验证功能即可。

---
##### 二进制部署方法
文档部署方法环境基于 CentOS 7.9。
###### 安装准备
```bash
#安装基础依赖
yum -y install wget vim make gcc gcc-c++ flex bison lynx ncurses libncurses-dev ncurses-devel mysql-devel mysql
```
---
###### 安装OpenSIPS
当前OpenSIPS新版本为3.5.2，但是实际测试下来配置文件与现有资料存在出入，本文使用2.4.6，版本较老，后续尝试配置新版本进行调试。
```bash
cd /usr/local/src
wget https://opensips.org/pub/opensips/2.4.6/opensips-2.4.6.tar.gz
tar -xzvf opensips-2.4.6.tar.gz
cd opensips-2.4.6
make menuconfig
```
进入OpenSIPS配置菜单，菜单操作方法：
- `→` - 确认
- `←` - 返回
- `空格` - 选择

根据菜单提示依次选择：`Configure Compile Options`->`Configure Excluded Modules`->选择`db_mysql`->返回`Save Changes`确认->返回`Compile And Install OpenSIPS`确认->`Exit & Save All Changes`确认。

---
##### 配置OpenSIPS
配置数据库连接信息。
```bash
#注意修改配置文件中的数据库用户名密码
vim /usr/local/etc/opensips/opensipsctlrc
DBENGINE=MYSQL
DBHOST=192.168.2.50
DBNAME=opensips
DBWUSER=root
DBWPW="123456"
DBROOTUSER="root"
```
配置OpenSIPS主配置文件。
```bash
#备份旧配置文件
mv /usr/local/etc/opensips/opensips.cfg /usr/local/etc/opensips/opensips.cfg.bak

#注意修改配置文件中的连接信息以及数据库用户名密码
vim /usr/local/etc/opensips/opensips.cfg
log_level=1
sip_warning=0
log_stderror=no
log_facility=LOG_LOCAL0
log_name="opensips"
debug_mode=no
children=4
dns_try_ipv6=no
auto_aliases=no
listen=udp:192.168.2.50:5060
mpath="/usr/local/lib64/opensips/modules/"

loadmodule "db_mysql.so"
loadmodule "signaling.so"
loadmodule "sl.so"
loadmodule "tm.so"
loadmodule "rr.so"
loadmodule "uri.so"
loadmodule "dialog.so"
loadmodule "maxfwd.so"
loadmodule "textops.so"
loadmodule "mi_fifo.so"
loadmodule "dispatcher.so"
loadmodule "load_balancer.so"
loadmodule "sipmsgops.so"
loadmodule "proto_udp.so"
modparam("mi_fifo", "fifo_name", "/tmp/opensips_fifo")
modparam("dialog", "db_mode", 1)
modparam("dialog", "db_url", "mysql://root:123456@192.168.2.50/opensips")
modparam("rr", "enable_double_rr", 1)
modparam("rr", "append_fromtag", 1)
modparam("dispatcher", "db_url", "mysql://root:123456@192.168.2.50/opensips")
modparam("dispatcher", "ds_ping_method", "OPTIONS")
modparam("dispatcher", "ds_ping_interval", 5)
modparam("dispatcher", "ds_probing_threshhold", 2)
modparam("dispatcher", "ds_probing_mode", 1)
modparam("load_balancer", "db_url", "mysql://root:123456@192.168.2.50/opensips")
modparam("load_balancer", "probing_method", "OPTIONS")
modparam("load_balancer", "probing_interval", 5)
route{

        if (!mf_process_maxfwd_header("10")) {
                sl_send_reply("483","Too Many Hops");
                exit;
        }

        if (!has_totag()) {
                record_route();
        }
        else {
                loose_route();
                t_relay();
                exit;
        }
        if (is_method("CANCEL")) {
                if ( t_check_trans() )
                        t_relay();
                exit;
        }
        if (is_method("INVITE")) {
                if (!load_balance("1","pstn","1")) {
                        send_reply("503","Service Unavailable");
                        exit;
                }
        }
        else if (is_method("REGISTER")) {
                if (!ds_select_dst("1", "4")) {
                        send_reply("503","Service Unavailable");
                        exit;
                }
        }
        else {
                send_reply("405","Method Not Allowed");
                exit;
        }
        if (!t_relay()) {
                sl_reply_error();
        }
}
```
创建数据库，安装过程中出现提示均按"Y"即可。
```bash
opensipsdbctl create
```
添加负载节点。
```sql
#在数据库中添加两个负载节点信息
INSERT INTO `opensips`.`load_balancer` (`id`, `group_id`, `dst_uri`, `resources`, `probe_mode`, `description`) VALUES ('1', '1', 'sip:192.168.2.48', 'pstn=500', '2', 'FS1');
INSERT INTO `opensips`.`load_balancer` (`id`, `group_id`, `dst_uri`, `resources`, `probe_mode`, `description`) VALUES ('2', '1', 'sip:192.168.2.49', 'pstn=500', '2', 'FS2');
```
- 各配置项含义
    - group_id - 负载均衡组id
    - dst_uri - 负载均衡目的地URI，必须以sip开头
    - resources - 配置的资源类型以及能容纳的最大负载数
    - probe_mode - 探测目的服务器模式
        - 0 - 从不探测目的服务可用性
        - 1 - 仅当目的服务不可用时，才进行探测，如果探测成功，自动将改地址置为可用
        - 2 - 实时发起探测请求，不管目的地是否可用。
        **实际测试下来，该参数配置为1时，OpenSIPS并不能及时发现FreeSWITCH的状态是否正常，会导致呼叫路由至状态不可用服务器而失败**

添加调度信息。
```bash
#将两台FreeSWITCH添加到调度列表
#这一步需要先在数据库里面修改opensips的默认密码
ALTER USER `opensips`@`%` IDENTIFIED BY '123456';
ALTER USER `opensips`@`192.168.2.50` IDENTIFIED BY '123456';

#再在OpenSIPS服务器上执行
opensipsctl dispatcher addgw 1 sip:192.168.2.48 5060 0 50 'FS1' 'node1'
opensipsctl dispatcher addgw 1 sip:192.168.2.49 5060 0 50 'FS2' 'node2'
```
>可能会出现如下报错，可以不管，在数据库中可查看到FreeSWITCH已经添加到调度列表中了
ERROR: /tmp/opensips_fifo does not exist
ERROR: Make sure you have the line 'modparam("mi_fifo", "fifo_name", "/tmp/opensips_fifo")' in your config
ERROR: and also have loaded the mi_fifo module.

可在opensips服务器上执行`opensipsctl dispatcher show`查看FreeSWITCH信息
+----+-------+------------------+--------+-------+--------+----------+-------+-------------+
| id | setid | destination      | socket | state | weight | priority | attrs | description |
+----+-------+------------------+--------+-------+--------+----------+-------+-------------+
|  1 |     1 | sip:192.168.2.48 | 5060   |     0 | 50     |        0 | FS1   | node1       |
|  2 |     1 | sip:192.168.2.49 | 5060   |     0 | 50     |        0 | FS2   | node2       |
+----+-------+------------------+--------+-------+--------+----------+-------+-------------+


>添加调度列表成功后，可在FreeSWITCH命令行通过`sofia profile 192.168.2.10 siptrace on`命令打开sip消息跟踪即可看到OpenSIPS节点不停再跟FreeSWITCH通过"OPTIONS"消息进行握手。
```
send 706 bytes to udp/[192.168.2.50]:5060 at 01:52:39.814418:
------------------------------------------------------------------------
SIP/2.0 200 OK
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bK369e.a6f2f446.0
From: <sip:prober@localhost>;tag=6cb8ea30c7209ae31645dca09d4b97da-60fe
To: <sip:192.168.2.49>;tag=3F1Q594N3BZHH
Call-ID: 6cbbb92959c7c3ae-19720@192.168.2.50
CSeq: 10 OPTIONS
Contact: <sip:192.168.2.49>
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release~64bit
Accept: application/sdp
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, REGISTER, REFER, NOTIFY, PUBLISH, SUBSCRIBE
Supported: timer, path, replaces
Allow-Events: talk, hold, conference, presence, as-feature-event, dialog, line-seize, call-info, sla, include-session-description, presence.winfo, message-summary, refer
Content-Length: 0
```

启动OpenSIPS
```bash
opensipsctl restart
```

---
#### FreeSWITCH配置修改
##### domain修改
修改域名IP地址，默认为FreeSWITCH服务器IP地址，需要修改为OpenSIPS的服务器IP地址。
```bash
vim /usr/local/freeswitch/etc/freeswitch/vars.xml
<!--X-PRE-PROCESS cmd="set" data="domain=$${local_ip_v4}"/-->
<X-PRE-PROCESS cmd="set" data="domain=192.168.2.10"/>
```
修改完成后，需要重启FreeSWITCH使配置生效。
#### 测试
##### 注册验证
使用LinPhone客户端注册，可注册成功，在FreeSWITCH命令行通过`sofia status profile sofia 192.168.2.10 reg`或`show registrations`可以查看用户注册情况，如下所示。
```
freeswitch@freeswitch-1> sofia status profile 192.168.2.50 reg
Registrations:
=================================================================================================
Call-ID:        uqS1ZvNpoR
User:           6001@192.168.2.50
Contact:        "6001" <sip:6001@192.168.2.47;transport=udp>
Agent:          Linphone-Desktop/5.2.6 (xinchen) windows/10 Qt/5.15.2 LinphoneSDK/5.3.72
Status:         Registered(UDP)(unknown) EXP(2024-12-09 02:25:20) EXPSECS(1741)
Ping-Status:    Reachable
Ping-Time:      0.00
Host:           freeswitch-1
IP:             192.168.2.47
Port:           5060
Auth-User:      6001
Auth-Realm:     192.168.2.50
MWI-Account:    6001@192.168.2.50

Call-ID:        uqS1ZvNpoR
User:           6001@192.168.2.50
Contact:        "6001" <sip:6001@192.168.2.47;transport=udp>
Agent:          Linphone-Desktop/5.2.6 (xinchen) windows/10 Qt/5.15.2 LinphoneSDK/5.3.72
Status:         Registered(UDP)(unknown) EXP(2024-12-09 02:52:49) EXPSECS(3390)
Ping-Status:    Reachable
Ping-Time:      0.00
Host:           freeswitch-1
IP:             192.168.2.47
Port:           5060
Auth-User:      6001
Auth-Realm:     192.168.2.48
MWI-Account:    6001@192.168.2.50

Total items returned: 2
=================================================================================================
freeswitch@freeswitch-1> show registrations
reg_user,realm,token,url,expires,network_ip,network_port,network_proto,hostname,metadata
6001,192.168.2.50,uqS1ZvNpoR,sofia/internal/sip:6001@192.168.2.47;transport=udp,1733711120,192.168.2.47,5060,udp,freeswitch-1,
6001,192.168.2.50,uqS1ZvNpoR,sofia/internal/sip:6001@192.168.2.47;transport=udp,1733712769,192.168.2.47,5060,udp,freeswitch-1,

2 total.

```

##### 通话负载均衡验证
使用分机6001拨打6002，在FS1服务器上可以看到通话日志，也可以打开sip消息跟踪查看通话信令，日志如下所示。
```log
recv 1745 bytes from udp/[192.168.2.50]:5060 at 02:05:50.220956:
------------------------------------------------------------------------
INVITE sip:6002@192.168.2.50 SIP/2.0
Record-Route: <sip:192.168.2.50;lr;ftag=1dRJ46afW;did=dd9.c9f99b7>
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKc5e4.3f83ce45.0
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.PvMZvseFJ;rport=5060
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: sip:6002@192.168.2.50
CSeq: 20 INVITE
Call-ID: gz3ZstBOsS
Max-Forwards: 69
Supported: replaces, outbound, gruu, path, record-aware
Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, REFER, NOTIFY, MESSAGE, SUBSCRIBE, INFO, PRACK, UPDATE
Content-Type: application/sdp
Content-Length: 988
Contact: <sip:6001@192.168.2.47;transport=udp>;expires=3600
User-Agent: Linphone-Desktop/5.2.6 (xinchen) windows/10 Qt/5.15.2 LinphoneSDK/5.3.72

v=0
o=6001 1515 191 IN IP4 192.168.2.47
s=Talk
c=IN IP4 192.168.2.47
t=0 0
a=rtcp-xr:rcvr-rtt=all:10000 stat-summary=loss,dup,jitt,TTL voip-metrics
a=group:BUNDLE as
a=record:off
m=audio 62044 RTP/SAVP 96 97 98 0 8 18 101 99 100
a=rtpmap:96 opus/48000/2
a=fmtp:96 useinbandfec=1
a=rtpmap:97 speex/16000
a=fmtp:97 vbr=on
a=rtpmap:98 speex/8000
a=fmtp:98 vbr=on
a=fmtp:18 annexb=yes
a=rtpmap:101 telephone-event/48000
a=rtpmap:99 telephone-event/16000
a=rtpmap:100 telephone-event/8000
a=crypto:1 AEAD_AES_128_GCM inline:oryW8dlZ6u4em+nO/M+R8TDKfFoQHQ5I8GLpkg==
a=crypto:2 AES_CM_128_HMAC_SHA1_80 inline:HFcveLOhW0o4af5DV2gYbEmGBRxXgdXacSwbq2Ue
a=crypto:3 AEAD_AES_256_GCM inline:YMh0VofVZlKvZeXb74zGgi+mDpoDJL7+oFHMfqsLnEknkuOzzc9sh/BYmz8=
a=crypto:4 AES_256_CM_HMAC_SHA1_80 inline:YeZTdQAqurnFjggnHl5YkeTDxKORWXxh+PqwOlnCTIYIxucXipc6gtCw0EbhHg==
a=rtcp-mux
a=mid:as
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=rtcp-fb:* trr-int 1000
a=rtcp-fb:* ccm tmmbr

send 879 bytes to udp/[192.168.2.50]:5060 at 02:05:50.227240:
------------------------------------------------------------------------
SIP/2.0 407 Proxy Authentication Required
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKc5e4.3f83ce45.0
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.PvMZvseFJ;rport=5060
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: <sip:6002@192.168.2.50>;tag=0tHKQ6FUpHgHj
Call-ID: gz3ZstBOsS
CSeq: 20 INVITE
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release~64bit
Accept: application/sdp
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, REGISTER, REFER, NOTIFY, PUBLISH, SUBSCRIBE
Supported: timer, path, replaces
Allow-Events: talk, hold, conference, presence, as-feature-event, dialog, line-seize, call-info, sla, include-session-description, presence.winfo, message-summary, refer
Proxy-Authenticate: Digest realm="192.168.2.50", nonce="6165c5a5-7232-45d1-95be-771b3fffd07a", algorithm=MD5, qop="auth"
Content-Length: 0


recv 319 bytes from udp/[192.168.2.50]:5060 at 02:05:50.228189:
------------------------------------------------------------------------
ACK sip:6002@192.168.2.50 SIP/2.0
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKc5e4.3f83ce45.0
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
Call-ID: gz3ZstBOsS
To: <sip:6002@192.168.2.50>;tag=0tHKQ6FUpHgHj
CSeq: 20 ACK
Max-Forwards: 70
User-Agent: OpenSIPS (2.4.6 (x86_64/linux))
Content-Length: 0


recv 1999 bytes from udp/[192.168.2.50]:5060 at 02:05:50.506543:
------------------------------------------------------------------------
INVITE sip:6002@192.168.2.50 SIP/2.0
Record-Route: <sip:192.168.2.50;lr;ftag=1dRJ46afW;did=dd9.d9f99b7>
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKd5e4.d2f37c61.0
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.go80nntWP;rport=5060
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: sip:6002@192.168.2.50
CSeq: 21 INVITE
Call-ID: gz3ZstBOsS
Max-Forwards: 69
Supported: replaces, outbound, gruu, path, record-aware
Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, REFER, NOTIFY, MESSAGE, SUBSCRIBE, INFO, PRACK, UPDATE
Content-Type: application/sdp
Content-Length: 988
Contact: <sip:6001@192.168.2.47;transport=udp>;expires=3600
User-Agent: Linphone-Desktop/5.2.6 (xinchen) windows/10 Qt/5.15.2 LinphoneSDK/5.3.72
Proxy-Authorization:  Digest realm="192.168.2.50", nonce="6165c5a5-7232-45d1-95be-771b3fffd07a", algorithm=MD5, username="6001",  uri="sip:6002@192.168.2.50", response="9801859799f6d5fe91cdb875fd6be5e5", cnonce="nW0rPU31LhqqrVIq", nc=00000001, qop=auth

v=0
o=6001 1515 191 IN IP4 192.168.2.47
s=Talk
c=IN IP4 192.168.2.47
t=0 0
a=rtcp-xr:rcvr-rtt=all:10000 stat-summary=loss,dup,jitt,TTL voip-metrics
a=group:BUNDLE as
a=record:off
m=audio 62044 RTP/SAVP 96 97 98 0 8 18 101 99 100
a=rtpmap:96 opus/48000/2
a=fmtp:96 useinbandfec=1
a=rtpmap:97 speex/16000
a=fmtp:97 vbr=on
a=rtpmap:98 speex/8000
a=fmtp:98 vbr=on
a=fmtp:18 annexb=yes
a=rtpmap:101 telephone-event/48000
a=rtpmap:99 telephone-event/16000
a=rtpmap:100 telephone-event/8000
a=crypto:1 AEAD_AES_128_GCM inline:oryW8dlZ6u4em+nO/M+R8TDKfFoQHQ5I8GLpkg==
a=crypto:2 AES_CM_128_HMAC_SHA1_80 inline:HFcveLOhW0o4af5DV2gYbEmGBRxXgdXacSwbq2Ue
a=crypto:3 AEAD_AES_256_GCM inline:YMh0VofVZlKvZeXb74zGgi+mDpoDJL7+oFHMfqsLnEknkuOzzc9sh/BYmz8=
a=crypto:4 AES_256_CM_HMAC_SHA1_80 inline:YeZTdQAqurnFjggnHl5YkeTDxKORWXxh+PqwOlnCTIYIxucXipc6gtCw0EbhHg==
a=rtcp-mux
a=mid:as
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=rtcp-fb:* trr-int 1000
a=rtcp-fb:* ccm tmmbr

send 280 bytes to udp/[192.168.2.50]:5060 at 02:05:50.527959:
------------------------------------------------------------------------
SIP/2.0 100 Trying
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKd5e4.d2f37c61.0
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: sip:6002@192.168.2.50
Call-ID: gz3ZstBOsS
CSeq: 21 INVITE
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release~64bit
Content-Length: 0


send 1402 bytes to udp/[192.168.3.43]:5060 at 02:05:50.544812:
------------------------------------------------------------------------
INVITE sip:6002@192.168.3.43;transport=udp SIP/2.0
Via: SIP/2.0/UDP 192.168.2.48;rport;branch=z9hG4bK2tZr9B4tBNFFr
Max-Forwards: 68
From: "Extension 6001" <sip:6001@192.168.2.48>;tag=2c44tvH2g3vpS
To: <sip:6002@192.168.3.43;transport=udp>
Call-ID: f454af74-3074-123e-ec8a-000c29633c29
CSeq: 92301183 INVITE
Contact: <sip:mod_sofia@192.168.2.48:5060>
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release~64bit
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, REGISTER, REFER, NOTIFY, PUBLISH, SUBSCRIBE
Supported: timer, path, replaces
Allow-Events: talk, hold, conference, presence, as-feature-event, dialog, line-seize, call-info, sla, include-session-description, presence.winfo, message-summary, refer
Content-Type: application/sdp
Content-Disposition: session
Content-Length: 453
X-FS-Support: update_display,send_info
Remote-Party-ID: "Extension 6001" <sip:6001@192.168.2.48>;party=calling;screen=yes;privacy=off

v=0
o=FreeSWITCH 1733686540 1733686541 IN IP4 192.168.2.48
s=FreeSWITCH
c=IN IP4 192.168.2.48
t=0 0
m=audio 23410 RTP/AVP 102 0 8 103 101
a=rtpmap:102 opus/48000/2
a=fmtp:102 useinbandfec=1; maxaveragebitrate=30000; maxplaybackrate=48000; ptime=20; minptime=10; maxptime=40; stereo=1
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:103 telephone-event/48000
a=fmtp:103 0-15
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-15
a=ptime:20

recv 266 bytes from udp/[192.168.3.43]:5060 at 02:05:50.557086:
------------------------------------------------------------------------
SIP/2.0 100 Trying
Via: SIP/2.0/UDP 192.168.2.48;rport;branch=z9hG4bK2tZr9B4tBNFFr
From: "Extension 6001" <sip:6001@192.168.2.48>;tag=2c44tvH2g3vpS
To: <sip:6002@192.168.3.43;transport=udp>
Call-ID: f454af74-3074-123e-ec8a-000c29633c29
CSeq: 92301183 INVITE


recv 437 bytes from udp/[192.168.3.43]:5060 at 02:05:50.744645:
------------------------------------------------------------------------
SIP/2.0 180 Ringing
Via: SIP/2.0/UDP 192.168.2.48;rport;branch=z9hG4bK2tZr9B4tBNFFr
From: "Extension 6001" <sip:6001@192.168.2.48>;tag=2c44tvH2g3vpS
To: <sip:6002@192.168.3.43;transport=udp>;tag=W8ngrG~
Call-ID: f454af74-3074-123e-ec8a-000c29633c29
CSeq: 92301183 INVITE
User-Agent: Linphone-Desktop/5.2.6 (jishudeMacBook-Pro.local) osx/11.7 Qt/5.15.2 LinphoneSDK/5.3.72
Supported: replaces, outbound, gruu, path, record-aware


send 1389 bytes to udp/[192.168.2.50]:5060 at 02:05:50.792964:
------------------------------------------------------------------------
SIP/2.0 183 Session Progress
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKd5e4.d2f37c61.0
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.go80nntWP;rport=5060
Record-Route: <sip:192.168.2.50;lr;ftag=1dRJ46afW;did=dd9.d9f99b7>
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: <sip:6002@192.168.2.50>;tag=13acS10yKt63D
Call-ID: gz3ZstBOsS
CSeq: 21 INVITE
Contact: <sip:6002@192.168.2.48:5060;transport=udp>
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release~64bit
Accept: application/sdp
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, REGISTER, REFER, NOTIFY, PUBLISH, SUBSCRIBE
Supported: timer, path, replaces
Allow-Events: talk, hold, conference, presence, as-feature-event, dialog, line-seize, call-info, sla, include-session-description, presence.winfo, message-summary, refer
Content-Type: application/sdp
Content-Disposition: session
Content-Length: 376
Remote-Party-ID: "6002" <sip:6002@192.168.2.50>;party=calling;privacy=off;screen=no

v=0
o=FreeSWITCH 1733683550 1733683551 IN IP4 192.168.2.48
s=FreeSWITCH
c=IN IP4 192.168.2.48
t=0 0
m=audio 26400 RTP/SAVP 96 101
a=rtpmap:96 opus/48000/2
a=fmtp:96 useinbandfec=1
a=rtpmap:101 telephone-event/48000
a=fmtp:101 0-15
a=ptime:20
a=rtcp-mux
a=rtcp:26400 IN IP4 192.168.2.48
a=crypto:1 AEAD_AES_128_GCM inline:TbCpWtOnu2Zf0zOuQmaO3u1lVdpmY0q438Oh+A==

recv 929 bytes from udp/[192.168.3.43]:5060 at 02:05:52.917343:
------------------------------------------------------------------------
SIP/2.0 200 Ok
Via: SIP/2.0/UDP 192.168.2.48;rport;branch=z9hG4bK2tZr9B4tBNFFr
From: "Extension 6001" <sip:6001@192.168.2.48>;tag=2c44tvH2g3vpS
To: <sip:6002@192.168.3.43;transport=udp>;tag=W8ngrG~
Call-ID: f454af74-3074-123e-ec8a-000c29633c29
CSeq: 92301183 INVITE
User-Agent: Linphone-Desktop/5.2.6 (jishudeMacBook-Pro.local) osx/11.7 Qt/5.15.2 LinphoneSDK/5.3.72
Supported: replaces, outbound, gruu, path, record-aware
Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, REFER, NOTIFY, MESSAGE, SUBSCRIBE, INFO, PRACK, UPDATE
Contact: <sip:6002@192.168.3.43;transport=udp>;expires=3600;+org.linphone.specs="lime"
Content-Type: application/sdp
Content-Length: 259

v=0
o=6002 1074 2284 IN IP4 192.168.3.43
s=Talk
c=IN IP4 192.168.3.43
t=0 0
m=audio 56904 RTP/AVP 102 0 8 103 101
a=rtpmap:102 opus/48000/2
a=fmtp:102 useinbandfec=1
a=rtpmap:103 telephone-event/48000
a=rtpmap:101 telephone-event/8000
a=rtcp:57377

send 385 bytes to udp/[192.168.3.43]:5060 at 02:05:52.924668:
------------------------------------------------------------------------
ACK sip:6002@192.168.3.43;transport=udp SIP/2.0
Via: SIP/2.0/UDP 192.168.2.48;rport;branch=z9hG4bK33rHB7my8X51K
Max-Forwards: 70
From: "Extension 6001" <sip:6001@192.168.2.48>;tag=2c44tvH2g3vpS
To: <sip:6002@192.168.3.43;transport=udp>;tag=W8ngrG~
Call-ID: f454af74-3074-123e-ec8a-000c29633c29
CSeq: 92301183 ACK
Contact: <sip:mod_sofia@192.168.2.48:5060>
Content-Length: 0


send 1359 bytes to udp/[192.168.2.50]:5060 at 02:05:52.962398:
------------------------------------------------------------------------
SIP/2.0 200 OK
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKd5e4.d2f37c61.0
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.go80nntWP;rport=5060
Record-Route: <sip:192.168.2.50;lr;ftag=1dRJ46afW;did=dd9.d9f99b7>
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: <sip:6002@192.168.2.50>;tag=13acS10yKt63D
Call-ID: gz3ZstBOsS
CSeq: 21 INVITE
Contact: <sip:6002@192.168.2.48:5060;transport=udp>
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release~64bit
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, REGISTER, REFER, NOTIFY, PUBLISH, SUBSCRIBE
Supported: timer, path, replaces
Allow-Events: talk, hold, conference, presence, as-feature-event, dialog, line-seize, call-info, sla, include-session-description, presence.winfo, message-summary, refer
Content-Type: application/sdp
Content-Disposition: session
Content-Length: 376
Remote-Party-ID: "Outbound Call" <sip:6002@192.168.2.50>;party=calling;privacy=off;screen=no

v=0
o=FreeSWITCH 1733683550 1733683551 IN IP4 192.168.2.48
s=FreeSWITCH
c=IN IP4 192.168.2.48
t=0 0
m=audio 26400 RTP/SAVP 96 101
a=rtpmap:96 opus/48000/2
a=fmtp:96 useinbandfec=1
a=rtpmap:101 telephone-event/48000
a=fmtp:101 0-15
a=ptime:20
a=rtcp-mux
a=rtcp:26400 IN IP4 192.168.2.48
a=crypto:1 AEAD_AES_128_GCM inline:TbCpWtOnu2Zf0zOuQmaO3u1lVdpmY0q438Oh+A==

send 1359 bytes to udp/[192.168.2.50]:5060 at 02:05:53.464029:
------------------------------------------------------------------------
SIP/2.0 200 OK
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKd5e4.d2f37c61.0
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.go80nntWP;rport=5060
Record-Route: <sip:192.168.2.50;lr;ftag=1dRJ46afW;did=dd9.d9f99b7>
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: <sip:6002@192.168.2.50>;tag=13acS10yKt63D
Call-ID: gz3ZstBOsS
CSeq: 21 INVITE
Contact: <sip:6002@192.168.2.48:5060;transport=udp>
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release~64bit
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, REGISTER, REFER, NOTIFY, PUBLISH, SUBSCRIBE
Supported: timer, path, replaces
Allow-Events: talk, hold, conference, presence, as-feature-event, dialog, line-seize, call-info, sla, include-session-description, presence.winfo, message-summary, refer
Content-Type: application/sdp
Content-Disposition: session
Content-Length: 376
Remote-Party-ID: "Outbound Call" <sip:6002@192.168.2.50>;party=calling;privacy=off;screen=no

v=0
o=FreeSWITCH 1733683550 1733683551 IN IP4 192.168.2.48
s=FreeSWITCH
c=IN IP4 192.168.2.48
t=0 0
m=audio 26400 RTP/SAVP 96 101
a=rtpmap:96 opus/48000/2
a=fmtp:96 useinbandfec=1
a=rtpmap:101 telephone-event/48000
a=fmtp:101 0-15
a=ptime:20
a=rtcp-mux
a=rtcp:26400 IN IP4 192.168.2.48
a=crypto:1 AEAD_AES_128_GCM inline:TbCpWtOnu2Zf0zOuQmaO3u1lVdpmY0q438Oh+A==

send 1359 bytes to udp/[192.168.2.50]:5060 at 02:05:54.466263:
------------------------------------------------------------------------
SIP/2.0 200 OK
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKd5e4.d2f37c61.0
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.go80nntWP;rport=5060
Record-Route: <sip:192.168.2.50;lr;ftag=1dRJ46afW;did=dd9.d9f99b7>
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: <sip:6002@192.168.2.50>;tag=13acS10yKt63D
Call-ID: gz3ZstBOsS
CSeq: 21 INVITE
Contact: <sip:6002@192.168.2.48:5060;transport=udp>
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release~64bit
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, REGISTER, REFER, NOTIFY, PUBLISH, SUBSCRIBE
Supported: timer, path, replaces
Allow-Events: talk, hold, conference, presence, as-feature-event, dialog, line-seize, call-info, sla, include-session-description, presence.winfo, message-summary, refer
Content-Type: application/sdp
Content-Disposition: session
Content-Length: 376
Remote-Party-ID: "Outbound Call" <sip:6002@192.168.2.50>;party=calling;privacy=off;screen=no

v=0
o=FreeSWITCH 1733683550 1733683551 IN IP4 192.168.2.48
s=FreeSWITCH
c=IN IP4 192.168.2.48
t=0 0
m=audio 26400 RTP/SAVP 96 101
a=rtpmap:96 opus/48000/2
a=fmtp:96 useinbandfec=1
a=rtpmap:101 telephone-event/48000
a=fmtp:101 0-15
a=ptime:20
a=rtcp-mux
a=rtcp:26400 IN IP4 192.168.2.48
a=crypto:1 AEAD_AES_128_GCM inline:TbCpWtOnu2Zf0zOuQmaO3u1lVdpmY0q438Oh+A==

recv 708 bytes from udp/[192.168.2.50]:5060 at 02:05:54.544684:
------------------------------------------------------------------------
ACK sip:6002@192.168.2.48:5060;transport=udp SIP/2.0
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKd5e4.d2f37c61.2
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;rport=5060;branch=z9hG4bK.4YqD9Rczo
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: <sip:6002@192.168.2.50>;tag=13acS10yKt63D
CSeq: 21 ACK
Call-ID: gz3ZstBOsS
Max-Forwards: 69
Proxy-Authorization:  Digest realm="192.168.2.50", nonce="6165c5a5-7232-45d1-95be-771b3fffd07a", algorithm=MD5, username="6001",  uri="sip:6002@192.168.2.50", response="9801859799f6d5fe91cdb875fd6be5e5", cnonce="nW0rPU31LhqqrVIq", nc=00000001, qop=auth
User-Agent: Linphone-Desktop/5.2.6 (xinchen) windows/10 Qt/5.15.2 LinphoneSDK/5.3.72


recv 708 bytes from udp/[192.168.2.50]:5060 at 02:05:54.897023:
------------------------------------------------------------------------
ACK sip:6002@192.168.2.48:5060;transport=udp SIP/2.0
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKd5e4.d2f37c61.2
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.4YqD9Rczo;rport=5060
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: <sip:6002@192.168.2.50>;tag=13acS10yKt63D
CSeq: 21 ACK
Call-ID: gz3ZstBOsS
Max-Forwards: 69
Proxy-Authorization:  Digest realm="192.168.2.50", nonce="6165c5a5-7232-45d1-95be-771b3fffd07a", algorithm=MD5, username="6001",  uri="sip:6002@192.168.2.50", response="9801859799f6d5fe91cdb875fd6be5e5", cnonce="nW0rPU31LhqqrVIq", nc=00000001, qop=auth
User-Agent: Linphone-Desktop/5.2.6 (xinchen) windows/10 Qt/5.15.2 LinphoneSDK/5.3.72


recv 708 bytes from udp/[192.168.2.50]:5060 at 02:05:54.973239:
------------------------------------------------------------------------
ACK sip:6002@192.168.2.48:5060;transport=udp SIP/2.0
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKd5e4.d2f37c61.2
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.4YqD9Rczo;rport=5060
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: <sip:6002@192.168.2.50>;tag=13acS10yKt63D
CSeq: 21 ACK
Call-ID: gz3ZstBOsS
Max-Forwards: 69
Proxy-Authorization:  Digest realm="192.168.2.50", nonce="6165c5a5-7232-45d1-95be-771b3fffd07a", algorithm=MD5, username="6001",  uri="sip:6002@192.168.2.50", response="9801859799f6d5fe91cdb875fd6be5e5", cnonce="nW0rPU31LhqqrVIq", nc=00000001, qop=auth
User-Agent: Linphone-Desktop/5.2.6 (xinchen) windows/10 Qt/5.15.2 LinphoneSDK/5.3.72


recv 727 bytes from udp/[192.168.2.50]:5060 at 02:05:56.831886:
------------------------------------------------------------------------
BYE sip:6002@192.168.2.48:5060;transport=udp SIP/2.0
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKa5e4.c4130395.0
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.-cBBLef1J;rport=5060
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: <sip:6002@192.168.2.50>;tag=13acS10yKt63D
CSeq: 22 BYE
Call-ID: gz3ZstBOsS
Max-Forwards: 69
User-Agent: Linphone-Desktop/5.2.6 (xinchen) windows/10 Qt/5.15.2 LinphoneSDK/5.3.72
Proxy-Authorization:  Digest realm="192.168.2.50", nonce="6165c5a5-7232-45d1-95be-771b3fffd07a", algorithm=MD5, username="6001",  uri="sip:6002@192.168.2.48:5060;transport=udp", response="8fa875bf640eca5939ce69907d32cef2", cnonce="6vQAU-a0yMi64Y3Z", nc=00000002, qop=auth


send 531 bytes to udp/[192.168.2.50]:5060 at 02:05:56.837159:
------------------------------------------------------------------------
SIP/2.0 200 OK
Via: SIP/2.0/UDP 192.168.2.50:5060;branch=z9hG4bKa5e4.c4130395.0
Via: SIP/2.0/UDP 192.168.2.47:5060;received=192.168.2.47;branch=z9hG4bK.-cBBLef1J;rport=5060
From: "6001" <sip:6001@192.168.2.50>;tag=1dRJ46afW
To: <sip:6002@192.168.2.50>;tag=13acS10yKt63D
Call-ID: gz3ZstBOsS
CSeq: 22 BYE
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release~64bit
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, REGISTER, REFER, NOTIFY, PUBLISH, SUBSCRIBE
Supported: timer, path, replaces
Content-Length: 0


send 588 bytes to udp/[192.168.3.43]:5060 at 02:05:56.856923:
------------------------------------------------------------------------
BYE sip:6002@192.168.3.43;transport=udp SIP/2.0
Via: SIP/2.0/UDP 192.168.2.48;rport;branch=z9hG4bK4cjaD25156UmF
Max-Forwards: 70
From: "Extension 6001" <sip:6001@192.168.2.48>;tag=2c44tvH2g3vpS
To: <sip:6002@192.168.3.43;transport=udp>;tag=W8ngrG~
Call-ID: f454af74-3074-123e-ec8a-000c29633c29
CSeq: 92301184 BYE
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release~64bit
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, REGISTER, REFER, NOTIFY, PUBLISH, SUBSCRIBE
Supported: timer, path, replaces
Reason: Q.850;cause=16;text="NORMAL_CLEARING"
Content-Length: 0


recv 429 bytes from udp/[192.168.3.43]:5060 at 02:05:56.880060:
------------------------------------------------------------------------
SIP/2.0 200 Ok
Via: SIP/2.0/UDP 192.168.2.48;rport;branch=z9hG4bK4cjaD25156UmF
From: "Extension 6001" <sip:6001@192.168.2.48>;tag=2c44tvH2g3vpS
To: <sip:6002@192.168.3.43;transport=udp>;tag=W8ngrG~
Call-ID: f454af74-3074-123e-ec8a-000c29633c29
CSeq: 92301184 BYE
User-Agent: Linphone-Desktop/5.2.6 (jishudeMacBook-Pro.local) osx/11.7 Qt/5.15.2 LinphoneSDK/5.3.72
Supported: replaces, outbound, gruu, path, record-aware

```
在OpenSIPS服务器上执行`opensipsctl fifo lb_list`可以看到FS1上的负载为1。
```
Destination:: sip:192.168.2.48 id=1 group=1 enabled=yes auto-reenable=on
        Resources::
                Resource:: pstn max=500 load=1
Destination:: sip:192.168.2.49 id=2 group=1 enabled=yes auto-reenable=on
        Resources::
                Resource:: pstn max=500 load=0
```
使用分机6003拨打6004，在FS2服务器上可以看到类似通话日志。
在OpenSIPS服务器上执行`opensipsctl fifo lb_list`可以看到FS2上的负载为1，通话负载均衡验证成功。
```
Destination:: sip:192.168.2.48 id=1 group=1 enabled=yes auto-reenable=on
        Resources::
                Resource:: pstn max=500 load=1
Destination:: sip:192.168.2.49 id=2 group=1 enabled=yes auto-reenable=on
        Resources::
                Resource:: pstn max=500 load=1
```

#### 附录
##### 使用Mac端LinPhone注册完成后，可以正常发起呼叫，但无法接收呼叫
该问题原因为Mac端LinPhone安装完成后，默认未打开SIP网络协议端口，需要在LinPhone客户端<设置>-<系统设置>-<网络>-<网络协议和端口>中打开`SIP UDP 端口`和`SIP TCP 端口`.