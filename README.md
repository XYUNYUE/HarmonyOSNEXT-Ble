# HarmonyOSNEXT-Ble
HarmonyOS NEXT-蓝牙(Ble)开发流程
# HarmonyOS Next 中 BLE开发流程及注意事项！

- 对于不熟悉 Ble 的开发者来讲，第一次接触会一头雾水，不知从何处入手。
- 现写一份 入门级 文档，希望能帮助到各位开发者。

## 流程说明
1. 检查蓝牙是否开启
2. 开启扫描，发现附近设备
3. 连接指定设备
4. 获取 固件 携带的蓝牙服务
5. 通过 写入特征 来进行向 固件 写入内容

## 详细步骤
### 1. 检查蓝牙状态
```
// 判断蓝牙是否开启
isBluetoothEnabled(): boolean {
  const state: access.BluetoothState = access.getState();
  if (state === access.BluetoothState.STATE_ON || state === access.BluetoothState.STATE_TURNING_ON) {
    return true;
  }
  return false;
}

// 检查并开启蓝牙
checkAndTurnOnBluetooth() {
  if (!this.isBluetoothEnabled()) {
    access.enableBluetooth();
  }
}
```
### 2. 开启扫描发现附近的蓝牙设备
- [.on('BLEDeviceFind')](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V14/js-apis-bluetooth-ble-V14#bleonbledevicefind)订阅BLE设备发现上报事件
- [startBLEScan](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V14/js-apis-bluetooth-ble-V14#blestartblescan)发起BLE扫描流程
```
// 订阅 BLEDeviceFind 事件。当开启扫描发现设备时会执行回调函数
ble.on('BLEDeviceFind', (data: Array<ble.ScanResult>) => {
  console.log('BLEDeviceFind事件发现了附近设备', data[0]);
  // 这里可以把数据加入到你需要展示的列表中
  ...
});

// 开启扫描 -- 只有调用了 startBLEScan 才会开启扫描附近设备
// 如果不需要特殊过滤需求 第一个参数可以直接写 null。具体其它参数参考上方超链接
ble.startBLEScan(null, {
    interval: 50,
  });
```
### 3. 连接指定设备（超级重要）⭐⭐⭐⭐⭐⭐⭐⭐⭐⭐
- 刚刚通过`BLEDeviceFind`扫描到的设备后的 [结果(点击查看介绍)](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V14/js-apis-bluetooth-ble-V14#scanfilter) 中可以获取到每个设备的 `deviceId`,也可以称为 `临时性 MAC地址`。
- 注意：这个`deviceId`因为是临时性的所以它会改变。不可以使用它来进行重连。
- 而且在这一步需要创建一个`GattClientDevice`。后续所有的蓝牙操作都需要基于这个来进行！！！
- ⭐ 有一个小知识点需要知道：你看下面的 demo 我是在 `createGattClientDevice` 创建完成之后，就 `on('BLECharacteristicChange')`监听了 特征值 。但是在文档中所说`需要先调用setCharacteristicChangeNotification 接口或 setCharacteristicChangeIndication 接口才能接收server端的通知` 这并不是意味着，是要先调用 这两个接口在去进行监听。实际上：这个并无先后顺序，而是 你在没有调用 上面任意一个接口之前 你的监听都是不会被触发。我发现很多开发者都被这个迷惑了。
- [createGattClientDevice](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V14/js-apis-bluetooth-ble-V14#blecreategattclientdevice) 创建GattClientDevice
- [on('BLEConnectionStateChange')](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V14/js-apis-bluetooth-ble-V14#onbleconnectionstatechange)订阅设备的连接状态变化事件
- [on('BLECharacteristicChange')](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V14/js-apis-bluetooth-ble-V14#onblecharacteristicchange)订阅蓝牙低功耗设备的特征值变化事件
- [on('BLEMtuChange')](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V14/js-apis-bluetooth-ble-V14#onblemtuchange-1)订阅MTU状态变化事件
- [connect](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V14/js-apis-bluetooth-ble-V14#connect)发起连接远端蓝牙低功耗设备

```
// deviceId 就是你要连接的设备的 deviceId
// ⭐ createGattClientDevice 创建 GattClientDevice
let gattClient: ble.GattClientDevice = ble.createGattClientDevice(deviceId);

// ⭐ 添加与设备连接状态的监听
gattClient.on("BLEConnectionStateChange", (state: ble.BLEConnectionChangeState) => { 
    // 这里监听连接状态
    if(state.state === constant.ProfileConnectionState.STATE_CONNECTED){
        // 连接成功
        // 获取 固件 携带的蓝牙服务.详见第4步
        ...
    }
    ...
});

// ⭐⭐⭐⭐ 监听 特征值变化。
// ⭐⭐⭐⭐ 固件向你发送的信息都在这里接收。包括你向 固件写入的信息之后固件给你返回的内容也是在这里
gattClient.on('BLECharacteristicChange', (characteristicChangeReq: ble.BLECharacteristic) => {
  console.log('监听并解析特征值')
  // 这里根据业务去做对应操作
  // 如果有多个服务发送通知，需要在这里根据服务id来区分
  // 根据 serviceUuid 或者 characteristicUuid 都可以根据业务来判断
  ...
});

// ⭐ 监听mtu变化
gattClient.on('BLEMtuChange', (mtu: number) => {
   // 获取到设备 mtu 大小
  console.log('BLEMtuChange', mtu);
});

// ⭐ 做了这些操作之后就可以进行连接了
gattClient.connect();

```

### 4. 获取 固件 携带的蓝牙服务
- 当在 `BLEConnectionStateChange` 监听里面，监听到连接成功后就可以获取蓝牙里面的服务端了。
- [getServices](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V13/js-apis-bluetooth-ble-V13#getservices-1) 获取蓝牙低功耗设备的所有服务
- [setCharacteristicChangeNotification](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V13/js-apis-bluetooth-ble-V13#setcharacteristicchangenotification)向服务端发送设置通知此特征值请求。这步必须要做。要不然在上方 `BLECharacteristicChange` 是监听不到消息的。
- [setBLEMtuSize](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V13/js-apis-bluetooth-ble-V13#setblemtusize)协商远端蓝牙低功耗设备的最大传输单元
```
// *** 代表固件服务的特征点。用来寻找到你想要的服务。具体是什么需要向嵌入式的人咨询 UUID

// 获取服务端集合
let serverResult = await gattClient.getServices();

// 从这个集合里面找到你所需要的服务
let firmService = serverResult.find(item => item.serviceUuid === '***');

// 找到你需要的服务后，获取里面的 写入特征和 通知特征

// 写入特征
let writeTrait = firmService.characteristics.find(item => item.characteristicUuid === '***');

// 通知特征
let notifyTrait = firmService.characteristics.find(item => item.characteristicUuid ===
'***');

// 获取到 通知特征 时候需要调用 setCharacteristicChangeNotification 
await gattClient.setCharacteristicChangeNotification(notifyTrait, true)

// 设置 mtu 大小，这里设置之后可以在 BLEMtuChange 中监听到设置后的值。
gattClient.setBLEMtuSize(512);

```

### 5.  通过 写入特征 来进行向 固件 写入内容
- 通过 `writeCharacteristicValue` 来进行写入动作。
- 通过刚刚获取的 `writeTrait` 来进行写入数据的装配。
- ⭐⭐⭐⭐⭐注意：同一时间只能有一个写入操作！不可以有多个。
```
// BLEServiceAndData 这是我当时随意定义的一个用于队列中区分多个服务的消息
// 发送的数据结果和需要使用的服务
export class BLEServiceAndData {
  // 发送的数据内容
  serviceData: Uint8Array;
  // 使用哪个服务发送
  setviceType: ble.BLECharacteristic

  constructor(source: BLEServiceAndData) {
    this.serviceData = source.serviceData || new Uint8Array();
    this.serviceType = source.serviceType;
  }
}

// 消息队列
let BLEPROPERTY_messageQueue:Array<BLEServiceAndData> = [];
let BLEPROPERTY_action?:number;

// 蓝牙向设备发送数据
class BLESendClass {
  // 追加消息后并开启发送动作
  BLESEND_autoPushSend(source: BLEServiceAndData) {
    this.BLESEND_pushMessage(source);
    this.BLESEND_openSend()
  }

  /**
   * 将数据加入到消息队列
   */
  BLESEND_pushMessage(source: BLEServiceAndData) {
    let data = source.serviceData || new Uint8Array();
    let length = data.length;
    for (let i = 0; i < length; ) {
      // 这里根据 你刚刚获取的mtu大小 来进行数据分包
      // -6 是因为每个包的数据都需要为 协议头留出位置
      const chunkLength = Math.min(MTUSIZE - 6, length - i);
      const chunk = data.slice(i, i + chunkLength);
      let BLEData: ESObject = {
        serviceData: chunk,
        serviceType: source.serviceType
      } as ESObject;
      let messageData = new BLEServiceAndData(BLEData);
      // 将分包好的数据加入到数据队列中
      BLEServiceAndData.push(messageData);
      i += chunkLength;
    }
  }

  //   开启发送动作
  BLESEND_openSend() {
    if (BLEPROPERTY_action || BLEPROPERTY_messageQueue.length < 1 ||
      !gattClient) {
      // 发送动作已经开启
      // 消息队列没有数据
      // 没有通过 createGattClientDevice 创建蓝牙工具
      // 则不用开启发送动作
      return;
    }
    // 开启发送动作 不要使用 while 改造。后续会改版成 多线程
    BLEPROPERTY_action = setTimeout(this.BLESEND_sendMessage, 0);
  }

  //   向设备发送数据
  BLESEND_sendMessage = async () => {
    try {
      if (BLEPROPERTY_messageQueue.length > 0 && gattClient) {
        // 发送信息
        // 获取本次发送消息所需要的服务
        // 这个需要重新构建
        let service = BLEPROPERTY_messageQueue[0].serviceType;
        // 获取本地发送的消息内容
        let data = 4BLEPROPERTY_messageQueue[0].serviceData;

        if (writeTrait && data) {
          let write: ble.BLECharacteristic = {
            serviceUuid: writeTrait.serviceUuid ?? "",
            characteristicUuid: writeTrait.characteristicUuid ?? "",
            characteristicValue: data.buffer,
            descriptors: writeTrait.descriptors ?? [],
            properties: writeTrait.properties
          };
          await gattClient.writeCharacteristicValue(write, ble.GattWriteType.WRITE_NO_RESPONSE);
        }

        // 消息发送后从消息队列中移除
        BLEPROPERTY_messageQueue.shift();
      }
      // 移除数据后判断是否还有未发送的数据
      // 如果已经发送完成则清除发送动作，等待下次发送后开启。
      if (BLEPROPERTY_messageQueue.length < 1 || !gattClient) {
        // 当消息队列信息为空的时候则关闭发送动作
        clearTimeout(BLEPROPERTY_action);
        BLEPROPERTY_action = undefined;
      }
    } catch (err) {
      promptAction.showToast({
        message:err.message || "数据发送失败",
        duration: 2000
      })
    }
  }
}

```

### 除了以上标记的重点注意事项外，额外在开发过程中遇到的问题记录：
#### 1. ArrayBuffer 和 Uint8Array 关系？
     ArrayBuffer 用来表示通用的原始二进制数据缓冲区，但你不能直接操作其中的数据
     Uint8Array 是一种类型化数组（TypedArray），它是 `ArrayBuffer` 的一种视图, 可以通过它来操作 ArrayBuffer
     至于我为什么提出这个问题，是因为我在分包的时候第一次使用的是 `subarray`
     导致，虽然视图改变了，但是我在写入时候，通过 "视图" 获取 ArrayBuffer 时候，还是同一个未改变的 ArrayBuffer。导致写入失败。后来发现了改成了 slice
#### 2. MTU 是什么东西？
    每个固件每次数据包所容纳的数据的大小不一致，而 MTU 就是代表最大的容量
    所以要根据 mtu 来进行分包发送
#### 3. 为什么要选用 setTimeout 而不是使用 setInterval或者while？
    setInterval 是一个周期性定时器，它并不会因为回调没执行结束而等待下一次执行。所以不满足 ble 的同时只能有一个写入的要求
    while 是一个会阻塞主线程的循环，所以为了避免出现意外情况就没用
    setTimeout 比较符合目前的需求，不过是个暂时性方案，后续会使用多线程改造
#### 4. 特征值是什么东西？
    一个蓝牙会有n个服务，而每个服务会有n个特征值。
    而特征值你可以理解为 每个服务所具有的功能。只有有这些功能你才可以使用它。
    比如：
        把蓝牙比作： 房屋(蓝牙)
        而一个 房屋(蓝牙) 里面又有很多 物品(服务)
             解释：物品就比如是 床、空调等用来给你提供服务的
        而一个 物品(服务) 又具有很多 功能(特征值)
             解释：比如 空调这个物品 ，它用来给你提供服务。
                  但是，有的空调有 制热功能(特征值) 有的空调又没有。
                  只有具有 制热功能(特征值) 的空调，你才可以使用 制热。
              换句话讲：
                  只有你获取到具有 写入的特征值(上面代码的writeTrait) 你才可以向设备写入。因为别的特征值的 characteristicUuid 设备不认可。
                   
#### 5. 客户端和服务端？
    我刚开始接触蓝牙的时候看文档中的 api和指南 都会有 服务端和客户端 两种。所以很晕不知道咋回事。
    对于我写 app 来讲，只需要关注 客户端相关的 api 即可
### 剩余问题：
#### 1. setBLEMtuSize 执行时机
     因为这个 setBLEMtuSize 并不是设置了就可以直接用的，需要等到设备响应才可以使用一些服务。
     我遇到这个是因为刚开始的时候 setBLEMtuSize 被我放到最前面，在 固件 未响应的时候就去向 固件 写入 导致失败。
     这个问题需要考虑一下，如何改造和执行时机
#### 2. 设备重连机制
     因为现在获取的都是 动态MAC地址(deviceId) 所以无法根据这个 deviceId 来进行 重连。
     目前想法是：固件 连接成功后，获取真实的 mac地址 然后通过 扫描的时候解析 设备广播 来匹配 匹配真实MAC地址 从而获取到 deviceId 。然后进行重连。
      但是需要要求固件在广播中携带 真实MAC地址

### 有什么疑问或问题可以留言，如果哪里有问题欢迎指正
