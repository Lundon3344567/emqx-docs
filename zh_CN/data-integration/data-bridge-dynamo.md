# DynamoDB

通过 DynamoDB 数据桥接可以将 MQTT 消息和客户端事件存储到 DynamoDB 中，也可以通过事件触发对 DynamoDB 中数据的更新或删除操作，从而实现对诸如设备在线状态、上下线历史等的记录。

{% emqxce %}
:::tip
EMQX 企业版功能。EMQX 企业版可以为您带来更全面的关键业务场景覆盖、更丰富的数据集成支持，更高的生产级可靠性保证以及 24/7 的全球技术支持，欢迎[免费试用](https://www.emqx.com/zh/try?product=enterprise)。
:::
{% endemqxce %}

## 前置条件

- 了解 EMQX 数据桥接[规则](./rules.md)。
- 了解[数据桥接](./data-bridges.md)。

## 特性

- [连接池](./data-bridges.md#连接池)
- [异步请求模式](./data-bridges.md#异步请求模式)
- [批量模式](./data-bridges.md#批量模式)
- [缓存队列](./data-bridges.md#缓存队列)
- [SQL 预处理](./data-bridges.md#sql-预处理)

## 快速开始教程

本节介绍如何配置 DynamoDB 数据桥接，包括如何设置 DynamoDB 服务器、创建数据桥接和规则以将数据转发到 DynamoDB、以及如何测试数据桥接和规则。

本教程假定您在本地机器上同时运行 EMQX 和 DynamoDB。如果您在远程运行 DynamoDB 和 EMQX，请相应地调整设置。

### 安装 DynamoDB 本地服务器

1. 准备一份 docker-compose 文件 `dynamo.yaml` 来部署 DynamoDB 本地服务器。

```json
version: '3.8'
services:
  dynamo:
    command: "-jar DynamoDBLocal.jar -sharedDb"
    image: "amazon/dynamodb-local:latest"
    container_name: dynamo
    ports:
      - "8000:8000"
    environment:
      AWS_ACCESS_KEY_ID: root 
      AWS_SECRET_ACCESS_KEY: public
      AWS_DEFAULT_REGION: us-west-2
```

2. 通过这份文件启动 DynamoDB 服务器。

```bash
docker-compose -f dynamo.yaml up
```

3. 准备一份数据表定义文件，将其存放到 home 目录，并取名为 `mqtt_msg.json`。

```json
{
    "TableName": "mqtt_msg",
    "KeySchema": [
        { "AttributeName": "id", "KeyType": "HASH" }
    ],
    "AttributeDefinitions": [
        { "AttributeName": "id", "AttributeType": "S" }
    ],
    "ProvisionedThroughput": {
        "ReadCapacityUnits": 5,
        "WriteCapacityUnits": 5
    }
}

```

4. 通过这份文件创建一个新的表。

```bash
docker run --rm -v ${HOME}:/dynamo_data -e AWS_ACCESS_KEY_ID=root -e AWS_SECRET_ACCESS_KEY=public -e AWS_DEFAULT_REGION=us-west-2 amazon/aws-cli dynamodb create-table --cli-input-json file:///dynamo_data/mqtt_msg.json --endpoint-url http://host.docker.internal:8000
```

5. 通过下面的指令检查创建是否成功。

```bash
docker run --rm -e AWS_ACCESS_KEY_ID=root -e AWS_SECRET_ACCESS_KEY=public -e AWS_DEFAULT_REGION=us-west-2 amazon/aws-cli dynamodb list-tables --endpoint-url http://host.docker.internal:8000
```

如果创建成功了，我们将会看到下面的输出。

```json
{
    "TableNames": [
        "mqtt_msg"
    ]
}
```

### 创建 DynamoDB 数据桥接

1. 转到 Dashboard **数据集成** -> **数据桥接**页面。

2. 点击页面右上角的**创建**。

3. 在数据桥接类型中选择 **DynamoDB**，点击**下一步**。

4. 输入数据桥接名称，要求是大小写英文字母和数字组合。

5. 输入 DynamoDB 连接信息:

   - **DynamoDB 地址**：填写 `http://127.0.0.1:8000`
   - **表名**：填写 `mqtt_msg`
   - **连接访问 ID**： 填写 `root`，
   - **连接访问密钥**：填写 `public`。

6. **模版**设置为默认值，即为空；模版为空时将会将整个消息转发给 DynamoDB，实际值为 JSON 模版数据。

7. 高级配置（可选），根据情况配置同步/异步模式，队列与批量等参数，详细请参考[数据桥接简介](./data-bridges.md)。

8. 在完成创建之前，您可以点击**测试连接**来测试桥接可以连接到 DynamoDB 服务器。

9. 点击**创建**按钮完成数据桥接创建。

   在弹出的**创建成功**对话框中您可以点击**创建规则**，继续创建规则以指定需要写入 DynamoDB 的数据。您也可以按照[创建 DynamoDB 数据桥接规则](#创建-dynamodb-数据桥接规则)章节的步骤来创建规则。

### 创建 DynamoDB 数据桥接规则

至此您已经完成数据桥接创建，接下来将继续创建一条规则来指定需要写入的数据。您需要为消息转发和设备上下线记录创建两条不同的规则。

1. 转到 Dashboard **数据集成** -> **规则**页面。

2. 点击页面右上角的**创建**。

3. 输入规则 ID `my_rule`，在 **SQL 编辑器**中根据业务实现需要输入规则：

   - 如需实现对指定主题消息的转发，例如将 `t/#` 主题的 MQTT 消息转发至 DynamoDB，输入以下 SQL 语法：

     注意：如果您希望制定自己的 SQL 语法，需要确保规则选出的字段（SELECT 部分）包含所有 SQL 模板中用到的变量。

     ```sql
     SELECT 
       *
     FROM
       "t/#"
     ```

   - 如需实现设备上下线记录，输入以下 SQL 语法：

     ```sql
     SELECT
       str(event) + timestamp as id, *
     FROM 
       "$events/client_connected", "$events/client_disconnected"
     ```

     ::: tip

     为了演示方便，上下线消息也设置为复用 `mqtt_msg` 表。

     :::

3. 点击**添加动作**，在动作下拉框中选择**使用数据桥接转发**选项，选择先前创建好的 DynamoDB 数据桥接。点击**添加**。

5. 点击最下方**创建**按钮完成规则创建。

至此您已经完成整个创建过程，可以前往 **数据集成** -> **Flows** 页面查看拓扑图，此时应当看到 `t/#` 主题的消息经过名为 `my_rule` 的规则处理，处理结果交由 DynamoDB 存储。

### 测试数据桥接与规则

使用 MQTTX 向 `t/1` 主题发布消息，此操作同时会触发上下线事件：

```bash
mqttx pub -i emqx_c -t t/1 -m '{ "msg": "hello DynamoDB" }'
```

分别查看两个数据桥接运行统计，命中、发送成功次数均 +1。

查看数据是否已经写入表中，`mqtt_msg` 表：

```bash
docker run --rm -e AWS_ACCESS_KEY_ID=root -e AWS_SECRET_ACCESS_KEY=public -e AWS_DEFAULT_REGION=us-west-2 amazon/aws-cli dynamodb scan --table-name=mqtt_msg --endpoint-url http://host.docker.internal:8000
```

输出如下：

```json
{
    "Items": [
        {
            "metadata": {
                "S": "{\"rule_id\":\"90d98f59\"}"
            },
            "peerhost": {
                "S": "127.0.0.1"
            },
            "clientid": {
                "S": "emqx_c"
            },
            "flags": {
                "S": "{\"retain\":false,\"dup\":false}"
            },
            "node": {
                "S": "emqx@127.0.0.1"
            },
            "qos": {
                "N": "0"
            },
            "payload": {
                "S": "{ \"msg\": \"hello DynamoDB\" }"
            },
            "pub_props": {
                "S": "{\"User-Property\":{}}"
            },
            "publish_received_at": {
                "N": "1678263363503"
            },
            "topic": {
                "S": "t/1"
            },
            "id": {
                "S": "0005F65F239F03FEF44300000BB40002"
            },
            "event": {
                "S": "message.publish"
            },
            "username": {
                "S": "undefined"
            },
            "timestamp": {
                "N": "1678263363503"
            }
        },
        {
            "conn_props": {
                "S": "{\"User-Property\":{},\"Request-Problem-Information\":1}"
            },
            "peername": {
                "S": "127.0.0.1:59582"
            },
            "metadata": {
                "S": "{\"rule_id\":\"703890a5\"}"
            },
            "clientid": {
                "S": "emqx_c"
            },
            "is_bridge": {
                "S": "false"
            },
            "keepalive": {
                "N": "30"
            },
            "proto_ver": {
                "N": "5"
            },
            "proto_name": {
                "S": "MQTT"
            },
            "connected_at": {
                "N": "1678263363499"
            },
            "receive_maximum": {
                "N": "32"
            },
            "sockname": {
                "S": "127.0.0.1:1883"
            },
            "mountpoint": {
                "S": "undefined"
            },
            "node": {
                "S": "emqx@127.0.0.1"
            },
            "id": {
                "S": "client.connected1678263363499"
            },
            "expiry_interval": {
                "N": "0"
            },
            "event": {
                "S": "client.connected"
            },
            "username": {
                "S": "undefined"
            },
            "timestamp": {
                "N": "1678263363499"
            },
            "clean_start": {
                "S": "true"
            }
        },
        {
            "reason": {
                "S": "normal"
            },
            "peername": {
                "S": "127.0.0.1:59582"
            },
            "metadata": {
                "S": "{\"rule_id\":\"703890a5\"}"
            },
            "clientid": {
                "S": "emqx_c"
            },
            "proto_ver": {
                "N": "5"
            },
            "proto_name": {
                "S": "MQTT"
            },
            "sockname": {
                "S": "127.0.0.1:1883"
            },
            "disconn_props": {
                "S": "{\"User-Property\":{}}"
            },
            "node": {
                "S": "emqx@127.0.0.1"
            },
            "id": {
                "S": "client.disconnected1678263363503"
            },
            "event": {
                "S": "client.disconnected"
            },
            "disconnected_at": {
                "N": "1678263363503"
            },
            "username": {
                "S": "undefined"
            },
            "timestamp": {
                "N": "1678263363503"
            }
        }
    ],
    "Count": 3,
    "ScannedCount": 3,
    "ConsumedCapacity": null
}
```
