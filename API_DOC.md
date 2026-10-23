# AIGC 服务 API 接口文档

> 给 Claude Code / Codex 等客户端用的 MCP 工具服务见 [MCP.md](./MCP.md)；本文档是传统的 HTTP 业务接口。

本文档提供声音克隆、形象克隆、声音合成、视频合成、音色转换功能的完整 API 接口说明，包括 Java 和 Python 语言的调用示例。

## 目录

1. [概述](#概述)
2. [声音克隆 API](#声音克隆-api)
3. [形象克隆 API](#形象克隆-api)
4. [声音合成 API](#声音合成-api)
5. [视频合成 API](#视频合成-api)
6. [音色转换 API](#音色转换-api)
7. [通用查询 API](#通用查询-api)
8. [错误码说明](#错误码说明)

---

## 概述

### 基础信息
- **Base URL**: `http://your-server:port`
- **认证方式**: Header 中添加 `Authorization: Bearer {md5_token}`
- **数据格式**: JSON

### 通用响应格式

```json
{
  "code": "0",
  "msg": "success",
  "data": {
    // 具体业务数据
  }
}
```

### 成功响应示例
```json
{
  "code": "0",
  "msg": "success",
  "data": "voice_id_12345"
}
```

### 错误响应示例
```json
{
  "code": "500",
  "msg": "错误描述信息",
  "data": null
}
```

---

## 声音克隆 API

声音克隆功能支持通过一段音频样本来创建定制化的语音模型，可用于后续的语音合成。

### 1.1 添加声音

创建一个新的声音模型。

**接口**: `POST /voice_clone/add_voice`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| voiceName | String | 是 | 声音名称 |
| audioUrl | String | 是 | 参考音频 URL（建议时长 1-5 分钟，清晰无噪音） |
| venture | String | 否 | 服务商，固定为 manteng(默认) |

**请求示例**:
```json
{
  "voiceName": "我的声音",
  "audioUrl": "https://example.com/audio/sample.wav",
  "venture": "manteng"
}
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": "zero_voice_abc123_myvoice"
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.HashMap;
import java.util.Map;

public class VoiceCloneExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        // 构建请求头
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Authorization", "Bearer " + md5Token);
        
        // 构建请求体
        Map<String, String> requestBody = new HashMap<>();
        requestBody.put("voiceName", "我的声音");
        requestBody.put("audioUrl", "https://example.com/audio/sample.wav");
        requestBody.put("venture", "manteng");
        
        HttpEntity<Map<String, String>> entity = new HttpEntity<>(requestBody, headers);
        
        try {
            ResponseEntity<String> response = restTemplate.postForEntity(
                baseUrl + "/voice_clone/add_voice",
                entity,
                String.class
            );
            
            System.out.println("响应状态：" + response.getStatusCode());
            System.out.println("响应内容：" + response.getBody());
            
            // 解析响应
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            String voiceId = (String) result.get("data");
            System.out.println("声音 ID: " + voiceId);
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests
import json

def clone_voice():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    # 请求头
    headers = {
        "Authorization": f"Bearer {md5_token}",
        "Content-Type": "application/json"
    }
    
    # 请求体
    payload = {
        "voiceName": "我的声音",
        "audioUrl": "https://example.com/audio/sample.wav",
        "venture": "manteng"
    }
    
    try:
        response = requests.post(
            f"{base_url}/voice_clone/add_voice",
            headers=headers,
            data=json.dumps(payload)
        )
        
        print(f"响应状态：{response.status_code}")
        print(f"响应内容：{response.text}")
        
        # 解析响应
        result = response.json()
        if result.get("code") == "0":
            voice_id = result.get("data")
            print(f"声音 ID: {voice_id}")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    clone_voice()
```

---

### 1.2 获取声音详情

根据声音 ID 获取声音的详细信息。

**接口**: `GET /voice_clone/get_voice`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| voiceId | String | 是 | 声音 ID |

**请求示例**:
```
GET /voice_clone/get_voice?voiceId=zero_voice_abc123_myvoice
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "md5": "abc123def456",
    "type": "tts",
    "venture": "manteng",
    "voiceName": "我的声音",
    "voiceId": "zero_voice_abc123_myvoice",
    "isBuiltin": false,
    "isDeleted": false,
    "refAudio": "https://example.com/audio/sample.wav",
    "refText": "这是参考文本",
    "createDate": "2024-01-01 12:00:00",
    "updateDate": "2024-01-01 12:00:00"
  }
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;

public class GetVoiceExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        String voiceId = "zero_voice_abc123_myvoice";
        
        // 构建请求头
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + md5Token);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        try {
            ResponseEntity<String> response = restTemplate.exchange(
                baseUrl + "/voice_clone/get_voice?voiceId=" + voiceId,
                HttpMethod.GET,
                entity,
                String.class
            );
            
            // 解析响应
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            Map<String, Object> voiceData = (Map<String, Object>) result.get("data");
            
            System.out.println("声音名称：" + voiceData.get("voiceName"));
            System.out.println("声音 ID: " + voiceData.get("voiceId"));
            System.out.println("参考音频：" + voiceData.get("refAudio"));
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def get_voice_details():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    voice_id = "zero_voice_abc123_myvoice"
    
    # 请求头
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    # 请求参数
    params = {
        "voiceId": voice_id
    }
    
    try:
        response = requests.get(
            f"{base_url}/voice_clone/get_voice",
            headers=headers,
            params=params
        )
        
        result = response.json()
        if result.get("code") == "0":
            voice_data = result.get("data")
            print(f"声音名称：{voice_data.get('voiceName')}")
            print(f"声音 ID: {voice_data.get('voiceId')}")
            print(f"参考音频：{voice_data.get('refAudio')}")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    get_voice_details()
```

---

### 1.3 获取声音列表

获取当前用户的声音列表。

**接口**: `GET /voice_clone/get_voices`

**请求参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| type | String | 否 | tts | 声音类型：tts(语音合成), vc(音色转换) |
| venture | String | 否 | manteng | 服务商：manteng |
| builtin | Boolean | 否 | true | 是否包含内置声音 |

**请求示例**:
```
GET /voice_clone/get_voices?type=tts&venture=manteng&builtin=true
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": [
    {
      "md5": "abc123",
      "type": "tts",
      "venture": "manteng",
      "voiceName": "我的声音",
      "voiceId": "zero_voice_abc123_myvoice",
      "isBuiltin": false,
      "isDeleted": false,
      "refAudio": "https://example.com/audio.wav",
      "refText": "参考文本",
      "createDate": "2024-01-01 12:00:00"
    }
  ]
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.List;
import java.util.Map;

public class GetVoicesExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + md5Token);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        try {
            String url = baseUrl + "/voice_clone/get_voices?type=tts&venture=manteng&builtin=true";
            ResponseEntity<String> response = restTemplate.exchange(
                url,
                HttpMethod.GET,
                entity,
                String.class
            );
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            List<Map<String, Object>> voices = (List<Map<String, Object>>) result.get("data");
            
            System.out.println("共找到 " + voices.size() + " 个声音:");
            for (Map<String, Object> voice : voices) {
                System.out.println("- " + voice.get("voiceName") + 
                    " (ID: " + voice.get("voiceId") + ")");
            }
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def get_voice_list():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    params = {
        "type": "tts",
        "venture": "manteng",
        "builtin": True
    }
    
    try:
        response = requests.get(
            f"{base_url}/voice_clone/get_voices",
            headers=headers,
            params=params
        )
        
        result = response.json()
        if result.get("code") == "0":
            voices = result.get("data", [])
            print(f"共找到 {len(voices)} 个声音:")
            for voice in voices:
                print(f"- {voice.get('voiceName')} (ID: {voice.get('voiceId')})")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    get_voice_list()
```

---

### 1.4 删除声音

删除指定的声音模型。

**接口**: `POST /voice_clone/delete_voice`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | Integer | 是 | 记录 ID |

**请求示例**:
```
POST /voice_clone/delete_voice?id=123
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": null
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;

public class DeleteVoiceExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        int recordId = 123;
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + md5Token);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        try {
            String url = baseUrl + "/voice_clone/delete_voice?id=" + recordId;
            ResponseEntity<String> response = restTemplate.exchange(
                url,
                HttpMethod.POST,
                entity,
                String.class
            );
            
            System.out.println("删除成功：" + response.getBody());
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def delete_voice():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    record_id = 123
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    params = {
        "id": record_id
    }
    
    try:
        response = requests.post(
            f"{base_url}/voice_clone/delete_voice",
            headers=headers,
            params=params
        )
        
        result = response.json()
        if result.get("code") == "0":
            print("删除成功")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    delete_voice()
```

---

### 1.5 获取声音克隆记录

获取声音克隆任务的处理记录列表。

**接口**: `GET /voice_clone/get_voice_records`

**请求参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| serviceName | String | 是 | - | 服务类型名称，如：DOMESTIC_VOICE_CLONE, GLOBAL_VOICE_CLONE, ADVANCED_VOICE_CLONE |
| page | Integer | 否 | 1 | 页码 |
| pageSize | Integer | 否 | 100 | 每页数量 |

**请求示例**:
```
GET /voice_clone/get_voice_records?serviceName=DOMESTIC_VOICE_CLONE&page=1&pageSize=100
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": [
    {
      "id": 12345,
      "md5": "abc123",
      "serviceType": "DOMESTIC_VOICE_CLONE",
      "prompt": "{\"voiceName\":\"我的声音\",\"audioUrl\":\"...\"}",
      "resultUrl": "zero_voice_abc123_myvoice",
      "status": "COMPLETED",
      "quota": 1500,
      "errorMessage": null,
      "createDate": "2024-01-01 12:00:00",
      "updateDate": "2024-01-01 12:00:00"
    }
  ]
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;
import java.util.List;

public class GetVoiceRecordsExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + md5Token);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        try {
            String url = baseUrl + "/voice_clone/get_voice_records?serviceName=DOMESTIC_VOICE_CLONE&page=1&pageSize=100";
            ResponseEntity<String> response = restTemplate.exchange(
                url,
                HttpMethod.GET,
                entity,
                String.class
            );
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            List<Map<String, Object>> records = (List<Map<String, Object>>) result.get("data");
            
            System.out.println("共找到 " + records.size() + " 条记录");
            for (Map<String, Object> record : records) {
                System.out.println("- 任务 " + record.get("id") + ": " + record.get("status"));
                System.out.println("  声音 ID: " + record.get("resultUrl"));
            }
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def get_voice_records():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    params = {
        "serviceName": "DOMESTIC_VOICE_CLONE",
        "page": 1,
        "pageSize": 100
    }
    
    try:
        response = requests.get(
            f"{base_url}/voice_clone/get_voice_records",
            headers=headers,
            params=params
        )
        
        result = response.json()
        if result.get("code") == "0":
            records = result.get("data", [])
            print(f"共找到 {len(records)} 条记录")
            for record in records:
                print(f"- 任务 {record.get('id')}: {record.get('status')}")
                print(f"  声音 ID: {record.get('resultUrl')}")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    get_voice_records()
```

---

## 形象克隆 API

形象克隆功能支持通过视频样本来创建数字人形象模型。

### 2.1 添加数字人形象

创建一个新的数字人形象。

**接口**: `POST /avatar/add`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| avatarName | String | 是 | 形象名称 |
| videoUrl | String | 是 | 参考视频 URL（建议时长 1-5 分钟，正面清晰） |
| venture | String | 否 | 服务商，可选值：manteng(默认), chanjing |
| silenceFigure | Boolean | 否 | 是否静默形象，默认 false |

**请求示例**:
```json
{
  "avatarName": "我的数字人",
  "videoUrl": "https://example.com/video/avatar.mp4",
  "venture": "manteng",
  "silenceFigure": false
}
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": "C-7f82537c5f2543289353046452a81836"
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.HashMap;
import java.util.Map;

public class AvatarCloneExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Authorization", "Bearer " + md5Token);
        
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("avatarName", "我的数字人");
        requestBody.put("videoUrl", "https://example.com/video/avatar.mp4");
        requestBody.put("venture", "manteng");
        requestBody.put("silenceFigure", false);
        
        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(requestBody, headers);
        
        try {
            ResponseEntity<String> response = restTemplate.postForEntity(
                baseUrl + "/avatar/add",
                entity,
                String.class
            );
            
            System.out.println("响应：" + response.getBody());
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            String avatarId = (String) result.get("data");
            System.out.println("形象 ID: " + avatarId);
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests
import json

def clone_avatar():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}",
        "Content-Type": "application/json"
    }
    
    payload = {
        "avatarName": "我的数字人",
        "videoUrl": "https://example.com/video/avatar.mp4",
        "venture": "manteng",
        "silenceFigure": False
    }
    
    try:
        response = requests.post(
            f"{base_url}/avatar/add",
            headers=headers,
            data=json.dumps(payload)
        )
        
        result = response.json()
        if result.get("code") == "0":
            avatar_id = result.get("data")
            print(f"形象 ID: {avatar_id}")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    clone_avatar()
```

---

### 2.2 删除形象

删除指定的数字人形象。

**接口**: `POST /avatar/delete`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | Integer | 是 | 记录 ID |

**请求示例**:
```
POST /avatar/delete?id=123
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": null
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;

public class DeleteAvatarExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        int recordId = 123;
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + md5Token);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        try {
            String url = baseUrl + "/avatar/delete?id=" + recordId;
            ResponseEntity<String> response = restTemplate.exchange(
                url,
                HttpMethod.POST,
                entity,
                String.class
            );
            
            System.out.println("删除成功：" + response.getBody());
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def delete_avatar():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    record_id = 123
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    params = {
        "id": record_id
    }
    
    try:
        response = requests.post(
            f"{base_url}/avatar/delete",
            headers=headers,
            params=params
        )
        
        result = response.json()
        if result.get("code") == "0":
            print("删除成功")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    delete_avatar()
```

---

### 2.3 获取形象列表

获取当前用户的数字人形象列表。

**接口**: `GET /avatar/list`

**请求参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| share | Integer | 否 | 0 | 是否共享形象，0:私有, 1:共享 |
| venture | String | 否 | - | 服务商：manteng, chanjing |
| name | String | 否 | - | 形象名称（模糊搜索） |

**请求示例**:
```
GET /avatar/list?share=0&venture=manteng&name=我的数字人
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": [
    {
      "md5": "abc123",
      "avatar": "我的数字人",
      "avatarId": "C-7f82537c5f2543289353046452a81836",
      "coverImg": "https://example.com/img/cover.jpg",
      "srcUrl": "https://example.com/video/avatar.mp4",
      "share": 0,
      "venture": "manteng",
      "createDate": "2024-01-01 12:00:00"
    }
  ]
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.List;
import java.util.Map;

public class GetAvatarsExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + md5Token);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        try {
            String url = baseUrl + "/avatar/list?share=0&venture=manteng";
            ResponseEntity<String> response = restTemplate.exchange(
                url,
                HttpMethod.GET,
                entity,
                String.class
            );
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            List<Map<String, Object>> avatars = (List<Map<String, Object>>) result.get("data");
            
            System.out.println("共找到 " + avatars.size() + " 个形象:");
            for (Map<String, Object> avatar : avatars) {
                System.out.println("- " + avatar.get("avatar") + 
                    " (ID: " + avatar.get("avatarId") + ")");
            }
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def get_avatar_list():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    params = {
        "share": 0,
        "venture": "manteng"
    }
    
    try:
        response = requests.get(
            f"{base_url}/avatar/list",
            headers=headers,
            params=params
        )
        
        result = response.json()
        if result.get("code") == "0":
            avatars = result.get("data", [])
            print(f"共找到 {len(avatars)} 个形象:")
            for avatar in avatars:
                print(f"- {avatar.get('avatar')} (ID: {avatar.get('avatarId')})")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    get_avatar_list()
```

---

### 2.4 获取形象克隆记录

获取形象克隆任务的处理记录列表。

**接口**: `GET /avatar/records`

**请求参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| serviceName | String | 是 | - | 服务类型名称，如：SHU_ZI_REN_ADD, ADVANCED_SHU_ZI_REN_ADD |
| page | Integer | 否 | 1 | 页码 |
| pageSize | Integer | 否 | 100 | 每页数量 |

**请求示例**:
```
GET /avatar/records?serviceName=SHU_ZI_REN_ADD&page=1&pageSize=100
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": [
    {
      "id": 12345,
      "md5": "abc123",
      "serviceType": "SHU_ZI_REN_ADD",
      "prompt": "{\"avatarName\":\"我的数字人\",\"videoUrl\":\"...\"}",
      "resultUrl": "C-7f82537c5f2543289353046452a81836",
      "status": "COMPLETED",
      "quota": 1500,
      "errorMessage": null,
      "createDate": "2024-01-01 12:00:00",
      "updateDate": "2024-01-01 12:00:00"
    }
  ]
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;
import java.util.List;

public class GetAvatarRecordsExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + md5Token);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        try {
            String url = baseUrl + "/avatar/records?serviceName=SHU_ZI_REN_ADD&page=1&pageSize=100";
            ResponseEntity<String> response = restTemplate.exchange(
                url,
                HttpMethod.GET,
                entity,
                String.class
            );
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            List<Map<String, Object>> records = (List<Map<String, Object>>) result.get("data");
            
            System.out.println("共找到 " + records.size() + " 条记录");
            for (Map<String, Object> record : records) {
                System.out.println("- 任务 " + record.get("id") + ": " + record.get("status"));
                System.out.println("  形象 ID: " + record.get("resultUrl"));
            }
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def get_avatar_records():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    params = {
        "serviceName": "SHU_ZI_REN_ADD",
        "page": 1,
        "pageSize": 100
    }
    
    try:
        response = requests.get(
            f"{base_url}/avatar/records",
            headers=headers,
            params=params
        )
        
        result = response.json()
        if result.get("code") == "0":
            records = result.get("data", [])
            print(f"共找到 {len(records)} 条记录")
            for record in records:
                print(f"- 任务 {record.get('id')}: {record.get('status')}")
                print(f"  形象 ID: {record.get('resultUrl')}")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    get_avatar_records()
```

---

### 2.5 根据ID获取形象详情

根据记录 ID 获取形象克隆任务的详细信息。也可使用 [通用查询接口](#7-通用查询-api) `GET /common/get_record_by_id?id={recordId}` 获取记录详情。

**接口**: `GET /avatar/get_by_id`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | Integer | 是 | 记录 ID |

**请求示例**:
```
GET /avatar/get_by_id?id=12345
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "id": 12345,
    "md5": "abc123",
    "serviceType": "SHU_ZI_REN_ADD",
    "prompt": "{\"avatarName\":\"我的数字人\",\"videoUrl\":\"...\"}",
    "resultUrl": "C-7f82537c5f2543289353046452a81836",
    "status": "COMPLETED",
    "quota": 1500,
    "errorMessage": null,
    "createDate": "2024-01-01 12:00:00",
    "updateDate": "2024-01-01 12:00:00"
  }
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;

public class GetAvatarByIdExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        int recordId = 12345;
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + md5Token);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        try {
            String url = baseUrl + "/avatar/get_by_id?id=" + recordId;
            ResponseEntity<String> response = restTemplate.exchange(
                url,
                HttpMethod.GET,
                entity,
                String.class
            );
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            Map<String, Object> data = (Map<String, Object>) result.get("data");
            
            System.out.println("状态：" + data.get("status"));
            System.out.println("形象 ID: " + data.get("resultUrl"));
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def get_avatar_by_id():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    record_id = 12345
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    params = {
        "id": record_id
    }
    
    try:
        response = requests.get(
            f"{base_url}/avatar/get_by_id",
            headers=headers,
            params=params
        )
        
        result = response.json()
        if result.get("code") == "0":
            data = result.get("data")
            print(f"状态：{data.get('status')}")
            print(f"形象 ID: {data.get('resultUrl')}")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    get_avatar_by_id()
```

---

## 声音合成 API

声音合成（TTS）功能将文本转换为语音。

### 3.1 TTS 文字转语音

使用标准音库进行语音合成（TTS）。

**接口**: `POST /voice/tts`

**请求参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| voiceId | String | 是 | - | 声音 ID |
| text | String | 是 | - | 要合成的文本 |
| speed | Float | 否 | 1.0 | 语速 (0.5-2.0) |
| ref_audio_text | String | 否 | - | 参考音频文本，用于声音克隆时指定参考音频对应的文本内容 |

**请求示例**:
```json
{
  "voiceId": "standard_voice_001",
  "text": "你好，这是一个测试",
  "speed": 1.0,
  "ref_audio_text": "这是参考音频的文本内容"
}
```

**响应**：`202 Accepted`，响应头带 `Location: /voice/tts/{id}`，body 直接是资源：

```json
{
  "id": 12345,
  "serviceType": "DOMESTIC_VOICE_CLONE_CHARACTER",
  "resultUrl": null,
  "status": "QUEUE",
  "quota": 14,
  "text": "你好，这是一个测试",
  "errorMessage": null,
  "createDate": "2024-01-01 12:00:00"
}
```

- 不再返回 `{code,msg,data}` 信封；失败统一为 RFC 9457 Problem Details
- 任务进度查 `GET /voice/tts/{id}`，不要再走 `/common/get_record_by_id`

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.HashMap;
import java.util.Map;

public class TTSExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Authorization", "Bearer " + md5Token);
        
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("voiceId", "standard_voice_001");
        requestBody.put("text", "你好，这是一个测试");
        requestBody.put("speed", 1.0);
        requestBody.put("ref_audio_text", "这是参考音频的文本内容");
        
        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(requestBody, headers);
        
        try {
            ResponseEntity<String> response = restTemplate.postForEntity(
                baseUrl + "/voice/tts",
                entity,
                String.class
            );
            
            System.out.println("响应：" + response.getBody());
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> data = mapper.readValue(response.getBody(), Map.class);
            Integer recordId = (Integer) data.get("id");
            System.out.println("记录 ID: " + recordId);
            System.out.println("状态：" + data.get("status"));
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests
import json

def tts():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}",
        "Content-Type": "application/json"
    }
    
    payload = {
        "voiceId": "standard_voice_001",
        "text": "你好，这是一个测试",
        "speed": 1.0,
        "ref_audio_text": "这是参考音频的文本内容"
    }
    
    try:
        response = requests.post(
            f"{base_url}/voice/tts",
            headers=headers,
            data=json.dumps(payload)
        )
        
        if response.ok:
            data = response.json()
            record_id = data.get("id")
            print(f"记录 ID: {record_id}")
            print(f"状态：{data.get('status')}")
        else:
            print(f"错误：{response.text}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    tts()
```

---

---

### 3.2 TTS 任务查询

查询 TTS 任务的记录列表。

**接口**: `GET /voice/tts`

**请求参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| serviceName | String | 否 | DOMESTIC_VOICE_CLONE_CHARACTER | 服务类型名称 |
| page | Integer | 否 | 1 | 页码 |
| pageSize | Integer | 否 | 100 | 每页数量（最大 200） |
| status | String | 否 | - | 按状态过滤 |
| prompt | String | 否 | - | 按文本关键字过滤 |

**请求示例**:
```
GET /voice/tts?serviceName=DOMESTIC_VOICE_CLONE_CHARACTER&page=1&pageSize=100
```

**响应示例**（直接返回资源，不再有 `code/msg/data` 信封）:
```json
{
  "items": [
    {
      "id": 12345,
      "serviceType": "DOMESTIC_VOICE_CLONE_CHARACTER",
      "resultUrl": "https://example.com/audio/output.mp3",
      "status": "COMPLETED",
      "quota": 50,
      "text": "你好",
      "errorMessage": null,
      "createDate": "2024-01-01 12:00:00"
    }
  ],
  "total": 10,
  "page": 1,
  "pageSize": 100
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;
import java.util.List;

public class GetTtsRecordsExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + md5Token);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        try {
            String url = baseUrl + "/voice/tts?serviceName=DOMESTIC_VOICE_CLONE_CHARACTER&page=1&pageSize=100";
            ResponseEntity<String> response = restTemplate.exchange(
                url,
                HttpMethod.GET,
                entity,
                String.class
            );
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> data = mapper.readValue(response.getBody(), Map.class);
            List<Map<String, Object>> records = (List<Map<String, Object>>) data.get("items");
            
            System.out.println("共找到 " + data.get("total") + " 条记录");
            for (Map<String, Object> record : records) {
                System.out.println("- 任务 " + record.get("id") + ": " + record.get("status"));
                System.out.println("  音频 URL: " + record.get("resultUrl"));
            }
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def get_tts_records():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    params = {
        "serviceName": "DOMESTIC_VOICE_CLONE_CHARACTER",
        "page": 1,
        "pageSize": 100
    }
    
    try:
        response = requests.get(
            f"{base_url}/voice/tts",
            headers=headers,
            params=params
        )
        
        if response.ok:
            data = response.json()
            records = data.get("items", [])
            print(f"共找到 {data.get('total')} 条记录")
            for record in records:
                print(f"- 任务 {record.get('id')}: {record.get('status')}")
                print(f"  音频 URL: {record.get('resultUrl')}")
        else:
            print(f"错误：{response.text}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    get_tts_records()
```

---

---

## 视频合成 API

视频合成（唇形同步）功能将音频与视频中的人物口型进行同步。

### 4.1 提交唇形同步任务 V2

提交一个视频唇形同步任务（推荐使用 V2 版本）。

**接口**: `POST /lip_sync/submit_lip_sync_v2`

**请求参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| audioUrl | String | 否 | - | 音频 URL（与 text+voiceId 组合使用） |
| videoUrl | String | 否 | - | 视频 URL（形象克隆视频） |
| text | String | 否 | - | 合成文本（与 audioUrl 二选一） |
| venture | String | 否 | manteng | 服务商，随 prompt 原样记录，当前不影响计价与派发（只剩自研一档） |
| avatarId | String | 否 | - | 数字人形象 ID（与 videoUrl 二选一） |
| voiceId | String | 否 | - | 声音 ID（与 text 配合使用） |

**说明**：
- 方式 1：audioUrl + videoUrl = 唇形同步（使用现有音频和视频）
- 方式 2：text + voiceId + videoUrl = 文字转语音 + 唇形同步
- 方式 3：audioUrl + avatarId = 唇形同步（使用已有形象）
- 方式 4：text + voiceId + avatarId = 文字转语音 + 唇形同步（使用已有形象）

**请求示例 1**（音视频同步）:
```json
{
  "audioUrl": "https://example.com/audio/speech.mp3",
  "videoUrl": "https://example.com/video/avatar.mp4",
  "venture": "manteng"
}
```

**请求示例 2**（文字生成视频）:
```json
{
  "text": "你好，这是我的数字人视频",
  "voiceId": "zero_voice_abc123_myvoice",
  "avatarId": "C-7f82537c5f2543289353046452a81836",
  "venture": "chanjing"
}
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "id": 12345,
    "md5": "abc123",
    "serviceType": "LIP_SYNC",
    "prompt": "{\"audioUrl\":\"...\",\"videoUrl\":\"...\",\"venture\":\"manteng\"}",
    "resultUrl": "task_id_xxxxx",
    "status": "PROCESSING",
    "quota": 0,
    "errorMessage": null,
    "createDate": "2024-01-01 12:00:00"
  }
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.HashMap;
import java.util.Map;

public class LipSyncV2Example {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Authorization", "Bearer " + md5Token);
        
        // 方式 1：音视频同步
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("audioUrl", "https://example.com/audio/speech.mp3");
        requestBody.put("videoUrl", "https://example.com/video/avatar.mp4");
        requestBody.put("venture", "manteng");
        
        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(requestBody, headers);
        
        try {
            ResponseEntity<String> response = restTemplate.postForEntity(
                baseUrl + "/lip_sync/submit_lip_sync_v2",
                entity,
                String.class
            );
            
            System.out.println("响应：" + response.getBody());
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            Map<String, Object> data = (Map<String, Object>) result.get("data");
            String taskId = (String) data.get("resultUrl");
            System.out.println("任务 ID: " + taskId);
            System.out.println("状态：" + data.get("status"));
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests
import json

def submit_lip_sync_v2():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}",
        "Content-Type": "application/json"
    }
    
    # 方式 1：音视频同步
    payload = {
        "audioUrl": "https://example.com/audio/speech.mp3",
        "videoUrl": "https://example.com/video/avatar.mp4",
        "venture": "manteng"
    }
    
    try:
        response = requests.post(
            f"{base_url}/lip_sync/submit_lip_sync_v2",
            headers=headers,
            data=json.dumps(payload)
        )
        
        result = response.json()
        if result.get("code") == "0":
            data = result.get("data")
            task_id = data.get("resultUrl")
            print(f"任务 ID: {task_id}")
            print(f"状态：{data.get('status')}")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    submit_lip_sync_v2()
```

---

---

### 4.2 图片数字人

基于静态图片生成说话的视频。

**接口**: `POST /lip_sync/submit_img_lip_sync`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| audioUrl | String | 是 | 音频 URL |
| imgUrl | String | 是 | 图片 URL |

**请求示例**:
```json
{
  "audioUrl": "https://example.com/audio/speech.mp3",
  "imgUrl": "https://example.com/image/portrait.jpg"
}
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "id": 12346,
    "status": "PROCESSING",
    "resultUrl": "task_id_img_xxxxx"
  }
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import java.util.HashMap;
import java.util.Map;

public class ImageLipSyncExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Authorization", "Bearer " + md5Token);
        
        Map<String, String> requestBody = new HashMap<>();
        requestBody.put("audioUrl", "https://example.com/audio/speech.mp3");
        requestBody.put("imgUrl", "https://example.com/image/portrait.jpg");
        
        HttpEntity<Map<String, String>> entity = new HttpEntity<>(requestBody, headers);
        
        try {
            ResponseEntity<String> response = restTemplate.postForEntity(
                baseUrl + "/lip_sync/submit_img_lip_sync",
                entity,
                String.class
            );
            
            System.out.println("响应：" + response.getBody());
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests
import json

def submit_image_lip_sync():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}",
        "Content-Type": "application/json"
    }
    
    payload = {
        "audioUrl": "https://example.com/audio/speech.mp3",
        "imgUrl": "https://example.com/image/portrait.jpg"
    }
    
    try:
        response = requests.post(
            f"{base_url}/lip_sync/submit_img_lip_sync",
            headers=headers,
            data=json.dumps(payload)
        )
        
        result = response.json()
        if result.get("code") == "0":
            data = result.get("data")
            print(f"任务已提交")
            print(f"状态：{data.get('status')}")
            print(f"任务 ID: {data.get('resultUrl')}")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    submit_image_lip_sync()
```

---

### 4.3 数字人视频合成

使用数字人形象和文本/音频生成视频。

**接口**: `POST /lip_sync/shuziren`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| voiceId | String | 否 | 声音 ID（与 audioUrl 二选一） |
| text | String | 否 | 合成文本（与 audioUrl 二选一） |
| audioUrl | String | 否 | 音频 URL（与 voiceId+text 二选一） |
| avatarId | String | 是 | 数字人形象 ID |
| speed | Float | 否 | 1.0 | 语速 |

**请求示例**:
```json
{
  "voiceId": "zero_voice_abc123_myvoice",
  "text": "你好，这是我的数字人视频",
  "avatarId": "C-7f82537c5f2543289353046452a81836",
  "speed": 1.0
}
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "id": 12347,
    "status": "PROCESSING",
    "resultUrl": "job_id_xxxxx"
  }
}
```

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import java.util.HashMap;
import java.util.Map;

public class DigitalHumanVideoExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Authorization", "Bearer " + md5Token);
        
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("voiceId", "zero_voice_abc123_myvoice");
        requestBody.put("text", "你好，这是我的数字人视频");
        requestBody.put("avatarId", "C-7f82537c5f2543289353046452a81836");
        requestBody.put("speed", 1.0);
        
        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(requestBody, headers);
        
        try {
            ResponseEntity<String> response = restTemplate.postForEntity(
                baseUrl + "/lip_sync/shuziren",
                entity,
                String.class
            );
            
            System.out.println("响应：" + response.getBody());
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests
import json

def create_digital_human_video():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}",
        "Content-Type": "application/json"
    }
    
    payload = {
        "voiceId": "zero_voice_abc123_myvoice",
        "text": "你好，这是我的数字人视频",
        "avatarId": "C-7f82537c5f2543289353046452a81836",
        "speed": 1.0
    }
    
    try:
        response = requests.post(
            f"{base_url}/lip_sync/shuziren",
            headers=headers,
            data=json.dumps(payload)
        )
        
        result = response.json()
        if result.get("code") == "0":
            data = result.get("data")
            print(f"任务已提交")
            print(f"状态：{data.get('status')}")
            print(f"任务 ID: {data.get('resultUrl')}")
        else:
            print(f"错误：{result.get('msg')}")
            
    except Exception as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    create_digital_human_video()
```

---

## 音色转换 API

音色转换（Voice Change）功能将源音频的音色转换为目标音频的音色，支持语音模式和音乐模式。

### 6.1 提交音色转换

提交音色转换任务，将源音频的音色转换为目标音频的音色。

**接口**: `POST /audio/voice_change`

**Content-Type**: `application/x-www-form-urlencoded`

**请求参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| sourceAudioUrl | String | 是 | - | 源音频 URL（要转换的音频） |
| targetAudioUrl | String | 是 | - | 目标音频 URL（提供目标音色的参考音频） |
| conversionMode | String | 否 | speech | 转换模式：speech（语音模式）、music（音乐模式） |
| vocalSeparationEnabled | boolean | 否 | true | 是否启用人声分离 |

**请求示例**:
```
POST /audio/voice_change
Content-Type: application/x-www-form-urlencoded

sourceAudioUrl=https://example.com/source.wav&targetAudioUrl=https://example.com/target.wav&conversionMode=speech&vocalSeparationEnabled=true
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": 12345
}
```

- `data` 为提交成功的记录 ID，后续可通过查询接口获取任务状态和结果。

**说明**: 
- 接口为异步处理，提交成功后会返回记录 ID，可通过 `GET /common/get_record_by_id?id=xxx` 查询任务状态。
- `conversionMode` 为 `speech` 时适用于人声音色转换，为 `music` 时适用于音乐音色转换。
- 开启 `vocalSeparationEnabled` 后系统会先进行人声分离再转换，可获得更纯净的音色转换效果。

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import java.util.Map;

public class VoiceChangeExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        headers.set("Authorization", "Bearer " + md5Token);
        
        MultiValueMap<String, String> formData = new LinkedMultiValueMap<>();
        formData.add("sourceAudioUrl", "https://example.com/source.wav");
        formData.add("targetAudioUrl", "https://example.com/target.wav");
        formData.add("conversionMode", "speech");
        formData.add("vocalSeparationEnabled", "true");
        
        HttpEntity<MultiValueMap<String, String>> entity = new HttpEntity<>(formData, headers);
        
        try {
            ResponseEntity<String> response = restTemplate.postForEntity(
                baseUrl + "/audio/voice_change",
                entity,
                String.class
            );
            
            System.out.println("响应：" + response.getBody());
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            if ("0".equals(result.get("code"))) {
                Integer recordId = (Integer) result.get("data");
                System.out.println("音色转换任务提交成功，记录ID：" + recordId);
            } else {
                System.err.println("提交失败：" + result.get("msg"));
            }
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def voice_change():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}",
        "Content-Type": "application/x-www-form-urlencoded"
    }
    
    data = {
        "sourceAudioUrl": "https://example.com/source.wav",
        "targetAudioUrl": "https://example.com/target.wav",
        "conversionMode": "speech",
        "vocalSeparationEnabled": "true"
    }
    
    try:
        response = requests.post(
            f"{base_url}/audio/voice_change",
            headers=headers,
            data=data
        )
        
        result = response.json()
        if result.get("code") == "0":
            record_id = result.get("data")
            print(f"音色转换任务提交成功，记录ID：{record_id}")
        else:
            print(f"提交失败：{result.get('msg')}")
            
    except requests.exceptions.RequestException as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    voice_change()
```

---

---

## 通用查询 API

Common 模块提供统一的记录查询、删除和配额查询接口，适用于所有异步任务类型（音色转换、音乐生成、视频合成、TTS、唇形同步、声音克隆、形象克隆等）。

### 7.1 根据记录ID查询任务

查询指定记录 ID 的任务详情和状态。

**接口**: `GET /common/get_record_by_id`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | Integer | 是 | 提交任务时返回的记录 ID |

**请求示例**:
```
GET /common/get_record_by_id?id=12345
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "id": 12345,
    "md5": "abc123",
    "status": "COMPLETED",
    "resultUrl": "https://example.com/output.wav",
    "prompt": "{\"conversionMode\":\"speech\",\"sourceAudioUrl\":\"...\"}",
    "serviceType": "VOICE_CHANGE",
    "quota": 750,
    "errorMessage": null,
    "createDate": "2024-06-11 10:00:00",
    "updateDate": "2024-06-11 10:05:00"
  }
}
```

**响应字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer | 记录 ID |
| md5 | String | 用户标识 |
| status | String | 任务状态：PENDING（处理中）、PROCESSING（处理中）、COMPLETED（已完成）、FAILED（失败） |
| resultUrl | String | 任务结果 URL（状态为 COMPLETED 时有效） |
| serviceType | String | 服务类型 |
| prompt | String | 任务提交时的请求参数（JSON 字符串） |
| errorMessage | String | 失败原因（状态为 FAILED 时） |
| quota | Integer | 消耗的配额 |
| createDate | String | 任务创建时间 |
| updateDate | String | 任务更新时间 |

#### Java 调用示例

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;

public class QueryRecordExample {
    
    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        String baseUrl = "http://your-server:8080";
        String md5Token = "your_md5_token";
        Integer recordId = 12345;
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + md5Token);
        
        HttpEntity<Void> entity = new HttpEntity<>(headers);
        
        try {
            ResponseEntity<String> response = restTemplate.exchange(
                baseUrl + "/common/get_record_by_id?id=" + recordId,
                HttpMethod.GET,
                entity,
                String.class
            );
            
            ObjectMapper mapper = new ObjectMapper();
            Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
            if ("0".equals(result.get("code"))) {
                Map<String, Object> data = (Map<String, Object>) result.get("data");
                System.out.println("任务状态: " + data.get("status"));
                System.out.println("结果URL: " + data.get("resultUrl"));
            } else {
                System.err.println("查询失败：" + result.get("msg"));
            }
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Python 调用示例

```python
import requests

def query_record():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    record_id = 12345
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    try:
        response = requests.get(
            f"{base_url}/common/get_record_by_id",
            headers=headers,
            params={"id": record_id}
        )
        
        result = response.json()
        if result.get("code") == "0":
            data = result.get("data")
            print(f"任务状态: {data.get('status')}")
            print(f"结果URL: {data.get('resultUrl')}")
        else:
            print(f"查询失败：{result.get('msg')}")
            
    except requests.exceptions.RequestException as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    query_record()
```

---

### 7.2 查询记录列表

按服务类型分页查询记录列表。

**接口**: `GET /common/get_records`

**请求参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| serviceName | String | 是 | - | 服务类型名称 |
| page | Integer | 否 | 1 | 页码 |
| pageSize | Integer | 否 | 100 | 每页数量 |

**请求示例**:
```
GET /common/get_records?serviceName=VOICE_CHANGE&page=1&pageSize=100
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "records": [
      {
        "id": 12345,
        "md5": "abc123",
        "serviceType": "VOICE_CHANGE",
        "status": "COMPLETED",
        "resultUrl": "https://example.com/output.wav",
        "quota": 750,
        "createDate": "2024-06-11 10:00:00"
      }
    ],
    "total": 10,
    "page": 1,
    "pageSize": 100
  }
}
```

#### Python 调用示例

```python
import requests

def get_records():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    try:
        response = requests.get(
            f"{base_url}/common/get_records",
            headers=headers,
            params={
                "serviceName": "VOICE_CHANGE",
                "page": 1,
                "pageSize": 100
            }
        )
        
        result = response.json()
        if result.get("code") == "0":
            data = result.get("data")
            records = data.get("records", [])
            print(f"共 {data.get('total')} 条记录，当前页 {len(records)} 条")
            for r in records:
                print(f"  - 记录 {r['id']}: {r['status']}")
        else:
            print(f"查询失败：{result.get('msg')}")
            
    except requests.exceptions.RequestException as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    get_records()
```

---

### 7.3 删除记录

根据记录 ID 删除指定记录。

**接口**: `POST /common/delete_record_by_id`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | Integer | 是 | 记录 ID |

**请求示例**:
```
POST /common/delete_record_by_id?id=12345
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "id": 12345,
    "md5": "abc123",
    "status": "COMPLETED",
    "resultUrl": "https://example.com/output.wav"
  }
}
```

#### Python 调用示例

```python
import requests

def delete_record():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    record_id = 12345
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    try:
        response = requests.post(
            f"{base_url}/common/delete_record_by_id",
            headers=headers,
            params={"id": record_id}
        )
        
        result = response.json()
        if result.get("code") == "0":
            print("删除成功")
        else:
            print(f"删除失败：{result.get('msg')}")
            
    except requests.exceptions.RequestException as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    delete_record()
```

---

### 7.4 查询全部配额

获取当前用户所有服务的配额信息。

**接口**: `GET /common/get_all_quotas`

**请求示例**:
```
GET /common/get_all_quotas
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "quotas": {
      "VOICE_CHANGE": { "total": 10000, "used": 750, "remaining": 9250 },
      "DOMESTIC_VOICE_CLONE": { "total": 50000, "used": 1500, "remaining": 48500 }
    }
  }
}
```

#### Python 调用示例

```python
import requests

def get_all_quotas():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    try:
        response = requests.get(
            f"{base_url}/common/get_all_quotas",
            headers=headers
        )
        
        result = response.json()
        if result.get("code") == "0":
            data = result.get("data")
            print("配额信息:", data)
        else:
            print(f"查询失败：{result.get('msg')}")
            
    except requests.exceptions.RequestException as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    get_all_quotas()
```

---

### 7.5 查询指定服务配额

查询指定服务的配额信息。

**接口**: `GET /common/get_quota`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| serviceName | String | 是 | 服务类型名称 |

**请求示例**:
```
GET /common/get_quota?serviceName=VOICE_CHANGE
```

**响应示例**:
```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "serviceName": "VOICE_CHANGE",
    "total": 10000,
    "used": 750,
    "remaining": 9250
  }
}
```

#### Python 调用示例

```python
import requests

def get_quota():
    base_url = "http://your-server:8080"
    md5_token = "your_md5_token"
    
    headers = {
        "Authorization": f"Bearer {md5_token}"
    }
    
    try:
        response = requests.get(
            f"{base_url}/common/get_quota",
            headers=headers,
            params={"serviceName": "VOICE_CHANGE"}
        )
        
        result = response.json()
        if result.get("code") == "0":
            data = result.get("data")
            print(f"服务: {data.get('serviceName')}")
            print(f"总量: {data.get('total')}, 已用: {data.get('used')}, 剩余: {data.get('remaining')}")
        else:
            print(f"查询失败：{result.get('msg')}")
            
    except requests.exceptions.RequestException as e:
        print(f"请求失败：{str(e)}")

if __name__ == "__main__":
    get_quota()
```

---

## 错误码说明

| 错误码 | 说明 |
|--------|------|
| 0 | 成功 |
| 500 | 服务器内部错误 |
| 400 | 请求参数错误 |
| 401 | 未授权访问 |
| 403 | 权限不足 |
| 404 | 资源不存在 |
| 500 | 请求处理失败 |
| 500 | 积分不足 |
| 500 | 请求参数缺失 |
| 500 | 记录未找到 |

### 常见错误处理

#### Java 错误处理示例

```
try {
    ResponseEntity<String> response = restTemplate.postForEntity(
        baseUrl + "/voice_clone/add_voice",
        entity,
        String.class
    );
    
    ObjectMapper mapper = new ObjectMapper();
    Map<String, Object> result = mapper.readValue(response.getBody(), Map.class);
    
    if (!"0".equals(result.get("code"))) {
        System.err.println("请求失败：" + result.get("msg"));
        // 根据错误码进行相应处理
    }
    
} catch (Exception e) {
    e.printStackTrace();
    // 网络异常或其他错误处理
}
```

#### Python 错误处理示例

```
try:
    response = requests.post(
        f"{base_url}/voice_clone/add_voice",
        headers=headers,
        data=json.dumps(payload)
    )
    
    result = response.json()
    if result.get("code") != "0":
        print(f"请求失败：{result.get('msg')}")
        # 根据错误码进行相应处理
    else:
        print("操作成功")
        
except requests.exceptions.RequestException as e:
    print(f"网络请求失败：{str(e)}")
except Exception as e:
    print(f"发生错误：{str(e)}")
```

---

## 最佳实践

### 1. 音频要求
- 格式：WAV 或 MP3
- 采样率：44.1kHz 或以上
- 时长：1-5 分钟（声音克隆）
- 质量：清晰无噪音，语速适中

### 2. 视频要求
- 格式：MP4
- 分辨率：720p 或以上
- 时长：1-10 分钟（形象克隆）
- 画面：人物正面，光线充足，表情自然

### 3. 性能优化
- 批量处理时使用异步任务
- 大文件先上传到 CDN/OSS
- 合理设置重试机制
- 监控任务状态并及时处理

### 4. 成本控制
- 根据业务需求选择合适的模型
- 定期清理不需要的声音/形象
- 利用缓存减少重复合成
- 监控积分消耗情况

---

## 技术支持

如有问题，请查阅：
- 项目 GitHub Issues
- 官方文档
- 技术支持群

---

*最后更新时间：2026-06-12*
