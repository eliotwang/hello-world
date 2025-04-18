测试app与智能助手或智能app交互方式及接口提案

`总体需求`

1. 测试app发送文本（图像可选）给智能助手或智能app，触发智能助手或智能app的数据处理流程；
2. 智能助手或智能app把处理结果文本（图像可选）发送给测试app；
3. 智能助手或智能app需要流式传输文本处理结果；
4. 测试app会自行计算ttft（首字延迟）及tps（每秒生成token数）指标；
5. 发送及返回内容拟包含如下json字段；

发送给智能助手或智能app的json:
```json
{
  "test_id": "TEST_001",
  "text_field": "这是一个测试文本字段",
  "image_local_path": "/path/to/images/test_image.jpg",
  "audio_local_path": "/path/to/audio/test_audio.mp3",
  "validity": {
    "text_valid": false,
    "image_valid": false,
    "audio_valid": false
  }
}
```
从智能助手或智能app处收到的json::
```json
{
  "test_id": "TEST_001",
  "status": "success or fail",
  "text_field": "这是一个测试文本字段",
  "image_local_path": "/path/to/images/test_image.jpg",
  "audio_local_path": "/path/to/audio/test_audio.mp3",
  "validity": {
    "text_valid": false,
    "image_valid": false,
    "audio_valid": false
  }
}
```

`实现方案提议`

考虑到适配代码的侵入性，适配工作量，接口定义灵活性，跨平台等因素，提议如下两种实现方式：
1. 基于 Broadcast/CommonEvent 的 Android 和 HarmonyOS Next 通信实现；
2. 基于AIDL/ArkTS IPC的Android和HarmonyOS Next 的通信实现；

其中，
方案1对厂商现有代码的侵入性较小，适配工作量较低，可以支持灵活的接口定义，跨平台（Android和HarmonyOS Next）有类似实现机制；
方案2需要对现有代码进行调整，我方需发布sdk，厂商需实现对应的service，有一定程度的代码侵入性，适配工作量适中，可以支持灵活的接口定义，跨平台（Android和HarmonyOS Next）有类似实现机制；

此外，考虑到测试app会自行计算ttft（首字延迟）及tps（每秒生成token数）指标，因此通信延迟是比较重要的因素，对于方案1，建议厂商把广播分为两个包，即首字包和其余字段包（厂商自行整理打包），减少通信次数，降低通信延迟。

下面对两种方案的相关实现细节做进一步展开描述，供厂商参考：

`基于 Broadcast/CommonEvent 的 Android 和 HarmonyOS Next 应用通信实现`

以下是一套基于 Broadcast（Android）和 CommonEvent（HarmonyOS Next）的完整实现方案，用于实现自研 App 与系统应用或其他普通应用的双向通信，支持发送和接收 JSON 数据（json in 和 json out），并满足流式传输 json out 中的 text_field 以及统计 TTFT（Time to First Token） 的需求。
实现包括 自研 App 侧 和 对端 App 侧（模拟系统应用或普通应用）的代码，涵盖 Android 和 HarmonyOS Next 的适配。

需求回顾

功能
1. 自研 App 发送 json in 数据给对端 App（系统应用或普通应用）。
2. 对端 App 处理后返回 json out 数据，其中 text_field 需流式传输。
3. 自研 App 统计 TTFT（从发送请求到接收 text_field 首个字符的时间）。

数据结构

json in:
```json
{
  "test_id": "TEST_001",
  "text_field": "这是一个测试文本字段",
  "image_local_path": "/path/to/images/test_image.jpg",
  "audio_local_path": "/path/to/audio/test_audio.mp3",
  "validity": {
    "text_valid": false,
    "image_valid": false,
    "audio_valid": false
  }
}
```
json out:
```json
{
  "test_id": "TEST_001",
  "status": "success or fail",
  "text_field": "这是一个测试文本字段",
  "image_local_path": "/path/to/images/test_image.jpg",
  "audio_local_path": "/path/to/audio/test_audio.mp3",
  "validity": {
    "text_valid": false,
    "image_valid": false,
    "audio_valid": false
  }
}
```
平台

兼容 Android 和 HarmonyOS Next。

通信方式
使用 Broadcast（Android）/CommonEvent（HarmonyOS Next）实现数据发送和流式接收。

实现方案概述
通信流程
自研 App 通过 Broadcast/CommonEvent 发送 json in 数据，触发对端 App。

对端 App 处理数据，生成 json out，通过 Broadcast/CommonEvent 流式返回 text_field 分块。

自研 App 接收流式数据，记录 TTFT，拼接完整 json out。

接口设计
发送：广播事件 com.example.ACTION_TEST，携带 json in。

接收：广播事件 com.example.TEXT_CHUNK（流式 text_field）和 com.example.RESULT（初始/结束消息）。

TTFT 统计
自研 App 在发送广播时记录 startTime。

在接收首个 text_chunk 时记录 firstTokenTime，计算 TTFT。


1. 自研 App 侧实现

Android 实现
发送广播（发送 json in）
```java
// MainActivity.java
package com.example.myapp;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.Bundle;
import android.util.Log;
import androidx.appcompat.app.AppCompatActivity;
import org.json.JSONObject;

public class MainActivity extends AppCompatActivity {
    private long startTime;
    private boolean firstChunk = true;
    private StringBuilder textField = new StringBuilder();
    private BroadcastReceiver receiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 注册广播接收器
        registerReceiver();

        // 发送 json in
        sendJsonIn();
    }

    private void sendJsonIn() {
        try {
            JSONObject jsonIn = new JSONObject();
            jsonIn.put("test_id", "TEST_001");
            jsonIn.put("text_field", "这是一个测试文本字段");
            jsonIn.put("image_local_path", "/sdcard/images/test_image.jpg");
            jsonIn.put("audio_local_path", "/sdcard/audio/test_audio.mp3");
            JSONObject validity = new JSONObject();
            validity.put("text_valid", false);
            validity.put("image_valid", false);
            validity.put("audio_valid", false);
            jsonIn.put("validity", validity);

            Intent intent = new Intent("com.example.ACTION_TEST");
            intent.putExtra("json_in", jsonIn.toString());
            startTime = System.nanoTime(); // 记录发送时间
            sendBroadcast(intent);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void registerReceiver() {
        receiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                try {
                    String type = intent.getStringExtra("type");
                    String testId = intent.getStringExtra("test_id");
                    if ("TEST_001".equals(testId)) {
                        if ("start".equals(type)) {
                            // 处理初始响应
                            String jsonOut = intent.getStringExtra("json_out");
                            JSONObject json = new JSONObject(jsonOut);
                            String status = json.getString("status");
                            // 处理 image_local_path, audio_local_path, validity
                        } else if ("text_chunk".equals(type)) {
                            // 处理流式 text_field
                            String chunk = intent.getStringExtra("text_field");
                            if (firstChunk) {
                                long firstTokenTime = System.nanoTime();
                                double ttftMs = (firstTokenTime - startTime) / 1_000_000.0;
                                Log.d("TTFT", "TTFT: " + ttftMs + " ms");
                                firstChunk = false;
                            }
                            textField.append(chunk);
                        } else if ("end".equals(type)) {
                            // 处理结束消息
                            Log.d("Result", "Complete text_field: " + textField.toString());
                            // 重置状态
                            firstChunk = true;
                            textField.setLength(0);
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        };
        IntentFilter filter = new IntentFilter();
        filter.addAction("com.example.RESULT");
        filter.addAction("com.example.TEXT_CHUNK");
        registerReceiver(receiver, filter);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (receiver != null) {
            unregisterReceiver(receiver);
        }
    }
}
```

AndroidManifest.xml 配置
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.myapp">
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <application
        android:allowBackup="true"
        android:label="@string/app_name">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

HarmonyOS Next 实现
发送 CommonEvent（发送 json in）
```java
// entry/src/main/ets/pages/Index.ets
import commonEvent from '@ohos.commonEvent';
import { BusinessError } from '@ohos.base';
import ability from '@ohos.app.ability';

@Entry
@Component
struct Index {
    private startTime: number = 0;
    private firstChunk: boolean = true;
    private textField: string = '';

    aboutToAppear() {
        // 订阅 CommonEvent
        this.subscribeCommonEvent();
        // 发送 json in
        this.sendJsonIn();
    }

    private sendJsonIn() {
        try {
            let jsonIn = {
                test_id: 'TEST_001',
                text_field: '这是一个测试文本字段',
                image_local_path: '/storage/emulated/0/images/test_image.jpg',
                audio_local_path: '/storage/emulated/0/audio/test_audio.mp3',
                validity: {
                    text_valid: false,
                    image_valid: false,
                    audio_valid: false
                }
            };

            let data = {
                parameters: { json_in: JSON.stringify(jsonIn) }
            };
            this.startTime = Date.now(); // 记录发送时间
            commonEvent.publish('com.example.ACTION_TEST', data, (err: BusinessError) => {
                if (err) {
                    console.error('Publish failed: ' + JSON.stringify(err));
                } else {
                    console.info('Publish success');
                }
            });
        } catch (e) {
            console.error('Error: ' + e);
        }
    }

    private subscribeCommonEvent() {
        commonEvent.subscribe({
            events: ['com.example.RESULT', 'com.example.TEXT_CHUNK']
        }, (err: BusinessError, data: commonEvent.CommonEventData) => {
            if (err) {
                console.error('Subscribe failed: ' + JSON.stringify(err));
                return;
            }
            try {
                let testId = data.parameters.test_id;
                let type = data.parameters.type;
                if (testId === 'TEST_001') {
                    if (type === 'start') {
                        // 处理初始响应
                        let jsonOut = JSON.parse(data.parameters.json_out);
                        let status = jsonOut.status;
                        // 处理 image_local_path, audio_local_path, validity
                        console.info('Status: ' + status);
                    } else if (type === 'text_chunk') {
                        // 处理流式 text_field
                        let chunk = data.parameters.text_field;
                        if (this.firstChunk) {
                            let firstTokenTime = Date.now();
                            let ttftMs = firstTokenTime - this.startTime;
                            console.info('TTFT: ' + ttftMs + ' ms');
                            this.firstChunk = false;
                        }
                        this.textField += chunk;
                    } else if (type === 'end') {
                        // 处理结束消息
                        console.info('Complete text_field: ' + this.textField);
                        // 重置状态
                        this.firstChunk = true;
                        this.textField = '';
                    }
                }
            } catch (e) {
                console.error('Error: ' + e);
            }
        });
    }
}
```
module.json 配置（HarmonyOS Next）
```json
{
  "module": {
    "package": "com.example.myapp",
    "name": ".MyApp",
    "type": "entry",
    "srcEntry": "./ets/pages/Index.ets",
    "description": "My App",
    "mainElement": "Index",
    "deviceTypes": ["phone"],
    "permissions": [
      {
        "name": "ohos.permission.READ_MEDIA",
        "reason": "Access media files"
      },
      {
        "name": "ohos.permission.WRITE_MEDIA",
        "reason": "Write media files"
      }
    ]
  }
}
```


2. 对端 App 侧实现（模拟系统应用或普通 application）

Android 实现
接收和处理广播
```java
// PeerActivity.java
package com.example.peerapp;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.Bundle;
import androidx.appcompat.app.AppCompatActivity;
import org.json.JSONObject;

public class PeerActivity extends AppCompatActivity {
    private BroadcastReceiver receiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 注册广播接收器
        registerReceiver();
    }

    private void registerReceiver() {
        receiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                if ("com.example.ACTION_TEST".equals(intent.getAction())) {
                    try {
                        String jsonIn = intent.getStringExtra("json_in");
                        JSONObject json = new JSONObject(jsonIn);
                        String testId = json.getString("test_id");
                        String textField = json.getString("text_field");

                        // 模拟处理
                        processData(testId, textField);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        IntentFilter filter = new IntentFilter("com.example.ACTION_TEST");
        registerReceiver(receiver, filter);
    }

    private void processData(String testId, String textField) {
        try {
            // 初始响应
            JSONObject jsonOut = new JSONObject();
            jsonOut.put("test_id", testId);
            jsonOut.put("status", "success");
            jsonOut.put("image_local_path", "/sdcard/images/result_image.jpg");
            jsonOut.put("audio_local_path", "/sdcard/audio/result_audio.mp3");
            JSONObject validity = new JSONObject();
            validity.put("text_valid", true);
            validity.put("image_valid", true);
            validity.put("audio_valid", true);
            jsonOut.put("validity", validity);

            Intent startIntent = new Intent("com.example.RESULT");
            startIntent.putExtra("test_id", testId);
            startIntent.putExtra("type", "start");
            startIntent.putExtra("json_out", jsonOut.toString());
            sendBroadcast(startIntent);

            // 流式返回 text_field
            String[] chunks = textField.split("(?<=\\G.{2})"); // 每 2 字符分块
            for (String chunk : chunks) {
                Intent chunkIntent = new Intent("com.example.TEXT_CHUNK");
                chunkIntent.putExtra("test_id", testId);
                chunkIntent.putExtra("type", "text_chunk");
                chunkIntent.putExtra("text_field", chunk);
                sendBroadcast(chunkIntent);
                Thread.sleep(100); // 模拟流式传输延迟
            }

            // 结束消息
            Intent endIntent = new Intent("com.example.RESULT");
            endIntent.putExtra("test_id", testId);
            endIntent.putExtra("type", "end");
            sendBroadcast(endIntent);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (receiver != null) {
            unregisterReceiver(receiver);
        }
    }
}
```

AndroidManifest.xml 配置
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.peerapp">
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <application
        android:allowBackup="true"
        android:label="@string/app_name">
        <activity android:name=".PeerActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <receiver android:name=".PeerActivity$BroadcastReceiver">
            <intent-filter>
                <action android:name="com.example.ACTION_TEST" />
            </intent-filter>
        </receiver>
    </application>
</manifest>
```
HarmonyOS Next 实现
接收和处理 CommonEvent
```java
// entry/src/main/ets/pages/PeerIndex.ets
import commonEvent from '@ohos.commonEvent';
import { BusinessError } from '@ohos.base';

@Entry
@Component
struct PeerIndex {
    aboutToAppear() {
        // 订阅 CommonEvent
        this.subscribeCommonEvent();
    }

    private subscribeCommonEvent() {
        commonEvent.subscribe({
            events: ['com.example.ACTION_TEST']
        }, (err: BusinessError, data: commonEvent.CommonEventData) => {
            if (err) {
                console.error('Subscribe failed: ' + JSON.stringify(err));
                return;
            }
            try {
                let jsonIn = JSON.parse(data.parameters.json_in);
                let testId = jsonIn.test_id;
                let textField = jsonIn.text_field;
                this.processData(testId, textField);
            } catch (e) {
                console.error('Error: ' + e);
            }
        });
    }

    private async processData(testId: string, textField: string) {
        try {
            // 初始响应
            let jsonOut = {
                test_id: testId,
                status: 'success',
                image_local_path: '/storage/emulated/0/images/result_image.jpg',
                audio_local_path: '/storage/emulated/0/audio/result_audio.mp3',
                validity: {
                    text_valid: true,
                    image_valid: true,
                    audio_valid: true
                }
            };

            let startData = {
                parameters: {
                    test_id: testId,
                    type: 'start',
                    json_out: JSON.stringify(jsonOut)
                }
            };
            commonEvent.publish('com.example.RESULT', startData, (err) => {
                if (err) console.error('Publish failed: ' + JSON.stringify(err));
            });

            // 流式返回 text_field
            let chunks = textField.match(/.{1,2}/g) || []; // 每 2 字符分块
            for (let chunk of chunks) {
                let chunkData = {
                    parameters: {
                        test_id: testId,
                        type: 'text_chunk',
                        text_field: chunk
                    }
                };
                commonEvent.publish('com.example.TEXT_CHUNK', chunkData, (err) => {
                    if (err) console.error('Publish failed: ' + JSON.stringify(err));
                });
                await new Promise(resolve => setTimeout(resolve, 100)); // 模拟延迟
            }

            // 结束消息
            let endData = {
                parameters: {
                    test_id: testId,
                    type: 'end'
                }
            };
            commonEvent.publish('com.example.RESULT', endData, (err) => {
                if (err) console.error('Publish failed: ' + JSON.stringify(err));
            });
        } catch (e) {
            console.error('Error: ' + e);
        }
    }
}
```
module.json 配置
```json
{
  "module": {
    "package": "com.example.peerapp",
    "name": ".PeerApp",
    "type": "entry",
    "srcEntry": "./ets/pages/PeerIndex.ets",
    "description": "Peer App",
    "mainElement": "PeerIndex",
    "deviceTypes": ["phone"],
    "permissions": [
      {
        "name": "ohos.permission.READ_MEDIA",
        "reason": "Access media files"
      },
      {
        "name": "ohos.permission.WRITE_MEDIA",
        "reason": "Write media files"
      }
    ],
    "skills": [
      {
        "actions": ["com.example.ACTION_TEST"]
      }
    ]
  }
}
```

3. 实现细节说明

通信流程
自研 App:
发送广播/CommonEvent（com.example.ACTION_TEST），携带 json in。

注册接收器，监听 com.example.RESULT（初始/结束消息）和 com.example.TEXT_CHUNK（流式 text_field）。

记录 TTFT，拼接 text_field。

对端 App:
接收 com.example.ACTION_TEST，解析 json in。

发送初始响应（com.example.RESULT，type=start），包含 status 和文件路径。

流式发送 text_field 分块（com.example.TEXT_CHUNK）。

发送结束消息（com.example.RESULT，type=end）。

TTFT 统计
自研 App 在发送广播时记录 startTime。

在接收首个 text_chunk 时记录 firstTokenTime，计算 TTFT
```java
double ttftMs = (firstTokenTime - startTime) / 1_000_000.0; // Android
let ttftMs = firstTokenTime - startTime; // HarmonyOS Next
```
文件处理
图像和音频文件通过路径（如 /sdcard/ 或 /storage/emulated/0/）传递。

确保双方 App 有存储权限（READ_EXTERNAL_STORAGE 和 WRITE_EXTERNAL_STORAGE）。

HarmonyOS Next 需适配其文件系统（如 /storage/emulated/0/）。

跨平台适配
Android: 使用 Intent 和 BroadcastReceiver，依赖 Android 的 IPC 机制。

HarmonyOS Next: 使用 CommonEvent 和 Want，适配分布式事件分发。

代码复用:
抽象 JSON 处理逻辑（如解析 json in 和 json out）。

定义统一的广播事件（如 com.example.ACTION_TEST）。

使用跨平台工具（如 TypeScript 的 JSON 解析）减少适配工作。


4. 注意事项

权限:
确保存储权限（Android 和 HarmonyOS Next）以访问文件路径。

HarmonyOS Next 需声明 ohos.permission.READ_MEDIA 和 WRITE_MEDIA。

广播可靠性:
Android 的广播可能因进程被杀死而丢失，建议对端 App 运行在前台或使用粘性广播。

HarmonyOS Next 的 CommonEvent 支持分布式分发，但需验证目标应用的接收能力。

数据量限制:
Broadcast/CommonEvent 的数据量有限，text_field 分块大小建议控制在 1KB 以内。

若 text_field 过大，可结合 Content Provider（Android）或 FileShare（HarmonyOS Next）传输。

系统应用支持:
验证目标系统应用（如智能助手）是否支持自定义广播。

若不支持，可通过 Intent/Want 触发系统动作（如 ACTION_SEND）。

调试:
使用 Android Studio 的 Logcat 查看广播日志。

使用 DevEco Studio 的日志工具监控 CommonEvent。


5. 扩展建议

错误处理:
检查 JSON 格式有效性，处理解析异常。

实现重试机制，应对广播丢失。

性能优化:
减少广播频率，合并小块 text_field（如每 500ms 发送一次）。

使用异步处理（Android 的 Handler 或 HarmonyOS Next 的 setTimeout）。

迁移工具:
使用 DevEco Studio 的迁移向导，将 Android 的 Broadcast 转换为 CommonEvent。

测试:
测试主流 Android 设备（小米、华为）和 HarmonyOS Next 设备。

验证 TTFT 统计的准确性（通过日志对比时间戳）。

总结

上述实现基于 Broadcast/CommonEvent，提供了自研 App 和对端 App 的完整代码，满足：
双向通信：发送 json in，接收 json out。

流式传输：text_field 通过分块广播实现流式返回。

TTFT 统计：精确记录首字符到达时间。

跨平台兼容：支持 Android 和 HarmonyOS Next，代码复用率约 80%。

该方案开发简单，适合快速实现与系统应用和普通应用的通信。若目标应用不支持广播，可考虑结合 Intent/Want 或 WebSocket 作为备选方案。

