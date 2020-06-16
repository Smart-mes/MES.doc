### MES基于数据点采集

##### 点位分类
- 数据点：定义点位的位置，和设备、PLC形成关联关系
- 标准点：定义业务逻辑，可监听该类点位触发业务事件
- 过程点：定义生产过程数据采集的点位，与工序管控项、工艺步骤相关联

##### 基础数据点
- SFC：过站条码
- Result：过站结果
- ResultCode：PLC写入异常代码的数据点
  > 1：提交成功
    2：工单不存在
    3：无法获取工位
    4：SN重码
    5：序列号不存在
    6：工艺路径错误
    7：编码规则错误
    8：无法获取后工序
    F：数据库提交错误

##### 业务场景
场景：监听标准点

1. 服务端调用`GET: api/BMachineStandardPoint/WithData`获取标准点数据
    ```
    [
      {
          "machineCode": "string",  // 设备编号
          "dataPointCode": "string",  // PLC数据点位
          "dataPointName": "string",  // 数据点名称
          "driveName": "string",  // 驱动名称
          "taskDrive": "string",  // 任务驱动
          "parameter": "string",  // 参数
          "businessCode": "string",  // 业务编号
          "businessName": "string"  // 业务名称
      }
    ]
    ```
2. 遍历每个标准点，用标准点的`dataPointCode`去PLC注册监听事件
3. 当监听事件触发时，根据标准点的`businessCode`执行不同的**业务事件**
---
场景：业务事件－过站数据采集（PLC）

1. 客户端的**PLC数采开关**从0变成1时，会触发服务端的监听事件
2. 服务端调用`GET: api/PassStation/Point?pointCode={触发点dataPointCode}`获取过站Job数据
    ```
    {
      "Code": "string",
      "Message": "string",
      "ResultCode": "string",  // 过站成功或失败回写的数据点
      "PointOrder": {  // 过站批次信息
        "MachineCode": "string",
        "ProcessCode": "string",
        "StationCode": "string",
        "LineCode": "string",
        "ParentOrder": "string",
        "MainOrder": "string",
        "OrderNo": "string",
        "ProductCode": "string",
        "Qty": 0,
        "WorkshopCode": "string",
        "FlowId": 0,
        "Sfc": "string",  // 过站批次
        "Result": "string",  // 过站结果
        "Ip": "string" // 取B_StationList的IpAdress
      },
      "ProcessStep": [  // 过站明细数据
        {
          "Pid": 0,
          "StepId": 0,
          "Idx": 0,
          "StepType": "string",
          "MatCode": "string",
          "Val": "string",
          "DataPointCode": "string", // 取B_Machine_DataPoint的dataPoint_code
          "Column": "string"  // 取B_Machine_Analog_Point的business_name
        }
      ]
    }
    ```
3. 服务端根据`PointOrder.SFC`和`PointOrder.Result`配置的数据点去PLC读取数据回填
4. 服务端遍历`ProcessStep`，如果`Val`为空且`DataPointCode`不为空时，根据`DataPointCode`去PLC读取数据回填到`Val`
5. 调用`POST: api/PassStation/Submit`提交过站Job数据，把返回的`Code`写到`GET: api/PassStation/Point`接口获取到的`ResultCode`
---
场景：业务事件－过站数据采集（Ding文件）

1. 客户端的**Ding文件数采开关**从0变成1时，会触发服务端的监听事件
2. 服务端调用`GET: api/PassStation/Point?pointCode={触发点dataPointCode}`获取过站Job数据，数据结构参考《业务事件－过站数据采集（PLC）
3. 服务端根据`PointOrder.SFC`配置的数据点去PLC读取数据回填，`PointOrder.Result`默认为`"OK"`
4. 服务端根据`PointOrder.Ip`连接客户端，读取当日的Ding文件
5. 服务端根据`PointOrder.SFC`查找到Ding文件当前批次**过站明细数据**
6. 服务端遍历`ProcessStep`，如果`Val`为空且`Column`不为空时，根据`Column`去当前批次**过站明细数据**读取数据回填到`Val`
7. 调用`POST: api/PassStation/Submit`提交过站Job数据，把返回的`Code`写到`GET: api/PassStation/Point`接口获取到的`ResultCode`

---
场景：业务事件－合普动力SFC校验
1. 客户端的**SFC校验**标准点的值发生改变，会触发服务端的监听事件
2. 服务端调用`POST: api/PassStation/HepuVaildSfc`
    ```
    {
      "DataPointCode": "HP_3_4_Dxxxx",
      "Sfcs": "sfc1_xxx&sfc2_xxx"
    }
    ```
3. API跟据`DataPointCode`获取该设备的**数据回执**标准点
4. API校验sfc2_xxx是否重码
5. 如果**数据回执**标准点的参数不为空，根据`sfc1_xxx`和参数`pid,stepId`去过站明细数据查询**旋变值**
6. 返回数据Value和DataPointCode
  ```
  {
    "Value": "NULL", // NG是失败，其他值都是成功
    "DataPointCode": "HP_3_4_Dxxxx"
  }
  ```