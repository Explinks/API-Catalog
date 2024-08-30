# 简单身份认证报文协议

## 1.1 通讯协议

商户和本系统之间采用 **HTTPS** 通讯协议。

## 1.3 数据加密说明

简单身份认证调用方式，只需要在请求头中携带自己的 AppCode 即可。

该请求头参数名为 `X-Mce-Signature` 值为: "AppCode" + "/" + "自己的AppCode"

例如：`X-Mce-Signature: AppCode/9ae2bf211459430e9cee594ff1d2a325`

## 1.2 报文格式
### 1.2.1 JSON方式

通讯报文可采用标准的**JSON**数据格式封装请求参数，文本编码格式：**UTF-8**。

例如：
```text 
POST /v1/tools/person/idcard HTTP/1.1
X-Mce-Signature: AppCode/9ae2bf211459430e9cee594ff1d2a325
Host: open.miitang.com
Content-Type: application/json;charset=utf-8

{
	"name":"米小泉",
	"idCardNo":"11010519001125523X"
}
```

### 1.2.2 表单方式

本系统也同时支持表单格式的是请求报文。

例如：
```text 
POST /v1/tools/person/idcard HTTP/1.1
X-Mce-Signature: AppCode/9ae2bf211459430e9cee594ff1d2a325
Host: open.miitang.com
Content-Type: application/x-www-form-urlencoded;charset=UTF-8

name=米小泉&idCardNo=11010519001125523X
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
