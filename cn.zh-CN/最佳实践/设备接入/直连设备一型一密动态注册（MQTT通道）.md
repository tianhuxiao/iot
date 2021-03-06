---
keyword: [IoT, 物联网, MQTT, 动态注册]
---

# 直连设备一型一密动态注册（MQTT通道）

本文提供基于MQTT通道的设备一型一密预注册示例代码。

物联网平台支持多种设备安全认证方式，具体认证方式，请参见[设备安全认证](/cn.zh-CN/设备接入/设备安全认证/概述.md)。

物联网平台支持基于MQTT通道实现[一型一密](/cn.zh-CN/设备接入/设备安全认证/一型一密.md)预注册、[一型一密](/cn.zh-CN/设备接入/设备安全认证/一型一密.md)免预注册认证。一型一密预注册中，设备通过动态注册获取连接云端所需的DeviceSecret。相关流程和参数说明，请参见[基于MQTT通道的设备动态注册](/cn.zh-CN/设备接入/使用开放协议自主接入/MQTT协议接入/基于MQTT通道的设备动态注册.md)。

本文提供基于Java语言的示例代码。

动态注册成功后，设备使用MQTT客户端直连接入物联网平台，示例代码请参见[Paho-MQTT Java接入示例](/cn.zh-CN/最佳实践/设备接入/使用Paho接入物联网平台/Paho-MQTT Java接入示例.md)。

## 前提条件

已完成[一型一密文档](/cn.zh-CN/设备接入/设备安全认证/一型一密.md)中的以下步骤：

1.  创建产品。
2.  开启动态注册。
3.  添加设备。
4.  产线烧录。

## pom.xml配置

在pom.xml文件中添加以下依赖，引入开发工具包。

```
<dependency>
  <groupId>org.eclipse.paho</groupId>
  <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
  <version>1.2.1</version>
</dependency>

<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>fastjson</artifactId>
  <version>1.2.61</version>
</dependency>
```

## 示例代码

**说明：**

-   当设备未激活时，动态注册可以反复进行，每次获取到的deviceSecret不一样。您需要保证只成功执行一次动态注册，并将获取的deviceSecret固化到本地。
-   已激活的设备如需再次动态注册，需要重置云端设备。您可调用[ResetThing](/cn.zh-CN/云端开发指南/云端API参考/设备管理/ResetThing.md) API进行重置。

```
/*   
 * Copyright © 2019 Alibaba. All rights reserved.
 */
package com.aliyun.iot.demo;

import java.nio.charset.StandardCharsets;
import java.util.Random;
import java.util.Set;
import java.util.SortedMap;
import java.util.TreeMap;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

import org.eclipse.paho.client.mqttv3.IMqttDeliveryToken;
import org.eclipse.paho.client.mqttv3.MqttCallback;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.MqttException;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;

import com.alibaba.fastjson.JSONObject;

/**
 * 设备动态注册。 
 */
public class DynamicRegisterByMqtt {

    // 地域ID，填写您的产品所在地域ID。
    private static String regionId = "cn-shanghai";

    // 定义加密方式。可选MAC算法：HmacMD5、HmacSHA1、HmacSHA256，需和signmethod取值一致。
    private static final String HMAC_ALGORITHM = "hmacsha1";

    // 接收物联网平台下发设备证书的Topic。无需创建，无需订阅，直接使用。
    private static final String REGISTER_TOPIC = "/ext/register";

    /**
     * 动态注册。
     * 
     * @param productKey 产品key
     * @param productSecret 产品密钥
     * @param deviceName 设备名称
     * @throws Exception
     */
    public void register(String productKey, String productSecret, String deviceName) throws Exception {

        // 接入域名，只能使用TLS。
        String broker = "ssl://" + productKey + ".iot-as-mqtt." + regionId + ".aliyuncs.com:1883";

        // 表示客户端ID，建议使用设备的MAC地址或SN码，64字符内。
        String clientId = productKey + "." + deviceName;

        // 获取随机值。
        Random r = new Random();
        int random = r.nextInt(1000000);

        // securemode只能为2表示只能使用TLS；signmethod指定签名算法。
        String clientOpts = "|securemode=2,authType=register,signmethod=" + HMAC_ALGORITHM + ",random=" + random + "|";

        // MQTT接入客户端ID。
        String mqttClientId = clientId + clientOpts;

        // MQTT接入用户名。
        String mqttUsername = deviceName + "&" + productKey;

        // MQTT接入密码，即签名。
        JSONObject params = new JSONObject();
        params.put("productKey", productKey);
        params.put("deviceName", deviceName);
        params.put("random", random);
        String mqttPassword = sign(params, productSecret);

        // 通过MQTT connect报文进行动态注册。
        connect(broker, mqttClientId, mqttUsername, mqttPassword);
    }

    /**
     * 通过MQTT connect报文发送动态注册信息。
     * 
     * @param serverURL 动态注册域名地址
     * @param clientId 客户端ID
     * @param username MQTT用户名
     * @param password MQTT密码
     */
    @SuppressWarnings("resource")
    private void connect(String serverURL, String clientId, String username, String password) {
        try {
            MemoryPersistence persistence = new MemoryPersistence();
            MqttClient sampleClient = new MqttClient(serverURL, clientId, persistence);
            MqttConnectOptions connOpts = new MqttConnectOptions();
            connOpts.setMqttVersion(4);// MQTT 3.1.1
            connOpts.setUserName(username);// 用户名
            connOpts.setPassword(password.toCharArray());// 密码
            connOpts.setAutomaticReconnect(false); // MQTT动态注册协议规定必须关闭自动重连。
            System.out.println("----- register params -----");
            System.out.print("server=" + serverURL + ",clientId=" + clientId);
            System.out.println(",username=" + username + ",password=" + password);
            sampleClient.setCallback(new MqttCallback() {
                @Override
                public void messageArrived(String topic, MqttMessage message) throws Exception {
                    // 仅处理动态注册返回消息。
                    if (REGISTER_TOPIC.equals(topic)) {
                        String payload = new String(message.getPayload(), StandardCharsets.UTF_8);
                        System.out.println("----- register result -----");
                        System.out.println(payload);
                        sampleClient.disconnect();
                    }
                }

                @Override
                public void deliveryComplete(IMqttDeliveryToken token) {
                }

                @Override
                public void connectionLost(Throwable cause) {
                }
            });
            sampleClient.connect(connOpts);
        } catch (MqttException e) {
            System.out.print("register failed: clientId=" + clientId);
            System.out.println(",username=" + username + ",password=" + password);
            System.out.println("reason " + e.getReasonCode());
            System.out.println("msg " + e.getMessage());
            System.out.println("loc " + e.getLocalizedMessage());
            System.out.println("cause " + e.getCause());
            System.out.println("excep " + e);
            e.printStackTrace();
        }
    }

    /**
     * 动态注册签名。
     * 
     * @param params 签名参数
     * @param productSecret 产品密钥
     * @return 签名十六进制字符串
     */
    private String sign(JSONObject params, String productSecret) {

        // 请求参数按字典顺序排序。
        Set<String> keys = getSortedKeys(params);

        // sign、signMethod除外。
        keys.remove("sign");
        keys.remove("signMethod");

        // 组装签名明文。
        StringBuffer content = new StringBuffer();
        for (String key : keys) {
            content.append(key);
            content.append(params.getString(key));
        }

        // 计算签名。
        String sign = encrypt(content.toString(), productSecret);
        System.out.println("sign content=" + content);
        System.out.println("sign result=" + sign);

        return sign;
    }

    /**
     * 获取JSON对象排序后的key集合。
     *
     * @param json 需要排序的JSON对象
     * @return 排序后的key集合
     */
    private Set<String> getSortedKeys(JSONObject json) {
        SortedMap<String, String> map = new TreeMap<String, String>();
        for (String key : json.keySet()) {
            String vlaue = json.getString(key);
            map.put(key, vlaue);
        }
        return map.keySet();
    }

    /**
     * 使用HMAC_ALGORITHM加密。
     * 
     * @param content 明文
     * @param secret 密钥
     * @return 密文
     */
    private String encrypt(String content, String secret) {
        try {
            byte[] text = content.getBytes(StandardCharsets.UTF_8);
            byte[] key = secret.getBytes(StandardCharsets.UTF_8);
            SecretKeySpec secretKey = new SecretKeySpec(key, HMAC_ALGORITHM);
            Mac mac = Mac.getInstance(secretKey.getAlgorithm());
            mac.init(secretKey);
            return byte2hex(mac.doFinal(text));
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    /**
     * 二进制转十六进制字符串。
     * 
     * @param b 二进制数组
     * @return 十六进制字符串
     */
    private String byte2hex(byte[] b) {
        StringBuffer sb = new StringBuffer();
        for (int n = 0; b != null && n < b.length; n++) {
            String stmp = Integer.toHexString(b[n] & 0XFF);
            if (stmp.length() == 1) {
                sb.append('0');
            }
            sb.append(stmp);
        }
        return sb.toString().toUpperCase();
    }

    public static void main(String[] args) throws Exception {

        String productKey = "您的设备productKey";
        String productSecret = "您的设备productSecret";
        String deviceName = "您的设备deviceName";

        // 进行动态注册。
        DynamicRegisterByMqtt client = new DynamicRegisterByMqtt();
        client.register(productKey, productSecret, deviceName);

        // 动态注册成功，需要在本地固化deviceSecret。
    }
}
```

