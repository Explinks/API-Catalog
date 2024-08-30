# 安全身份认证报文协议

为了更加安全的保护客户信息，在使用https的基础上，我们还提供了基于 `appId` 与 `appSecret` 的加密请求方式。

## 1.1 通讯协议

商户和本系统之间采用 **HTTPS** 通讯协议。

## 1.2 报文格式

通讯报文可采用标准的**JSON**数据格式封装请求参数，文本编码格式：**UTF-8**。

例如：
```text 
POST /v1/tools/person/idcard HTTP/1.1
x-mce-signature: mce-auth-v1/{appId}/{timeStamp}/{expire}/{sign}
Host: open.miitang.com
Content-Type: application/json;charset=utf-8

XXXXXXXXX加密后的请求参数XXXXXXXXX
```
其中大括号包裹的是需要替换的内容

appId：您的appId

timeStamp：时间戳，格式 `yyyy-MM-dd'T'HH:mm:ss'Z` 示例：2020-06-28T21:24:02Z

## 1.3 数据加密与签名说明

请求数据加密步骤如下：

1. 将请求参数转化为JSON字符串
2. 将JSON字符串转化为字节数组（utf-8编码）
3. 将字节数组使用AES加密

下方提供Java版本的加密实现示例：

```java
public String encrypt(byte[] data, String key) {
    if (key.length() < 16) {
        throw new RuntimeException("Invalid AES key length (must be 16 bytes)");
    } else if (key.length() > 16) {
        key = key.substring(0, 16);
    }
    try {
        Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");// 创建密码器
        SecretKeySpec secretKey = new SecretKeySpec(key.getBytes(CHARSET), "AES");
        IvParameterSpec ivspec = new IvParameterSpec("\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0".getBytes());
        cipher.init(Cipher.ENCRYPT_MODE, secretKey, ivspec);// 初始化
        byte[] result = cipher.doFinal(data);
        return Base64.encodeBase64String(result);
    } catch (Exception e){
        throw new RuntimeException("encrypt failed!");
    }
}
```

请求数据的签名步骤如下：

1. 按格式： mce-auth-v1/{appId}/{timeStamp}/{expire} 凭借成**签名前缀**
2. 使用 secretKey 对**签名前缀**进行 hmacHSA256 摘要，形成**派生秘钥**
3. 对请求参数按参数名进行升序排序（去除空值参数），排序后使用"="连接参数名与参数值，而后使用"&"连接各个参数对。
4. 对排序连接后的请求参数使用**派生秘钥**再次进行 hmacHSA256 摘要，形成最终的签名（{sign}）

签名过程的Java举例：

```java
/**
 * 签名工具类
 */
public class SignUtils {
    private static final String PREFIX = "mce-auth-v1";

    private static final String FORMAT = "yyyy-MM-dd'T'HH:mm:ss'Z'";

    /**
     * 签名：生成认证字符串 mce-auth-v1/{appId}/{timeStamp}/{expire}/{sign}
     * @param appId 用户id
     * @param secretKey 用户秘钥
     * @param datetime 签名时间
     * @param expire 签名过期时间
     * @param requestParam 请求参数
     * @return 签名认证字符串
     */
    public static String sign(String appId, String secretKey, Date datetime, int expire, Map<String,? extends Object> requestParam){
        // 生成时间戳
        String timeStamp = new SimpleDateFormat(FORMAT).format(datetime);
        //
        StringBuilder signStr = new StringBuilder(PREFIX).append("/")
                .append(appId).append("/")
                .append(timeStamp).append("/")
                .append(expire);
        // 生成派生秘钥
        String signKey = hmacHSA256(secretKey, signStr.toString());
        // 请求参数规范化
        SortedMap<String, Object> sortMap = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
        sortMap.putAll(requestParam);
        StringBuilder content = new StringBuilder();
        for (Map.Entry<String, Object> entry : sortMap.entrySet()) {
            String paramName = entry.getKey();
            Object paramValue = entry.getValue();
            if (paramValue == null || "".equals(paramValue.toString()) || "null".equals(paramValue.toString())) {
                continue;
            }
            if (content.length()>0) {
                content.append("&");
            }
            content.append(paramName).append("=").append(paramValue);
        }
        // 生成签名
        String sign = hmacHSA256(signKey, content.toString());
        //
        signStr.append("/").append(sign);
        return signStr.toString();
    }

    /**
     * HmacSHA256 摘要
     * @param key 秘钥
     * @param content 摘要内容
     * @return 摘要结果
     */
    public static String hmacHSA256(String key, String content){
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            SecretKeySpec secretKey = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
            mac.init(secretKey);
            byte[] bytes = mac.doFinal(content.getBytes(StandardCharsets.UTF_8));
            return hex(bytes);
        } catch (Exception e){
            throw new RuntimeException("签名生成错误！",e);
        }
    }

    /**
     * byte数组转16进制
     * @param bytes 数组
     * @return 16进制字符串
     */
    public static String hex(byte[] bytes) {
        StringBuilder result = new StringBuilder();
        for (byte aByte : bytes) {
            int decimal = (int) aByte & 0xff;
            // get last 8 bits
            String hex = Integer.toHexString(decimal);
            if (hex.length() % 2 == 1) {
                hex = "0" + hex;
            }
            result.append(hex);
        }
        return result.toString();
    }
}

```

## 1.3 响应格式

请求的响应内容为 **JSON** 格式。

例如：
```text
HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8

{
	"code":"FP00000",
	"message":"SUCCESS",
	...
}
```

## 1.4 网关地址

https://open.miitang.com

## 1.5 多种语言的对接工具

| 语言 | 工具地址 |
| --- | ------- |
| java | [https://static.miitang.com/saas/simple/MtUtils.java](https://static.miitang.com/saas/simple/MtUtils.java) <br/> [https://static.miitang.com/saas/simple/pom.xml](https://static.miitang.com/saas/simple/pom.xml) |
| php | [https://static.miitang.com/saas/simple/MtUtils_php.zip](https://static.miitang.com/saas/simple/MtUtils_php.zip) |
| C# | [https://static.miitang.com/saas/simple/MtUtils.cs](https://static.miitang.com/saas/simple/MtUtils.cs) |
| python | [https://static.miitang.com/saas/simple/mt_utils.py](https://static.miitang.com/saas/simple/mt_utils.py) |
