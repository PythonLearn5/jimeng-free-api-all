# 国际版 Seedance 接口设计文档

> 版本：v0.8.9 | 更新日期：2026-04-01

---

## 1. 概述

国际版 Seedance 是即梦 AI 免费 API 服务中的核心功能模块，基于 CapCut/Dreamina 国际平台（`mweb-api-sg.capcut.com`）提供多模态视频生成能力。与国内版不同，国际版通过纯算法签名（X-Bogus / X-Gnarly）绕过 Shark 反爬验证，无需 Playwright 浏览器代理。

### 1.1 核心特性

- 支持全球 **26 个区域** Token 接入（亚洲、欧洲、中东、南美等）
- 纯 TypeScript 算法签名绕过 Shark 反爬（零外部依赖）
- 同步 + 异步双模式视频生成
- 多模态素材混合上传（图片 / 视频 / 音频）
- `@1`、`@2` 占位符引用素材
- 4~15 秒可配置时长，720p/1080p 分辨率
- 服务端自动积分检查与扣减

### 1.2 与国内版对比

| 特性 | 国内版 (CN) | 国际版 (International) |
|------|------------|----------------------|
| 上游平台 | `jimeng.jianying.com` | `mweb-api-sg.capcut.com` |
| Shark 绕过 | Playwright + bdms SDK（浏览器代理） | X-Bogus + X-Gnarly（纯算法） |
| Token 格式 | 裸 sessionid | `{region}-` 前缀 + sessionid |
| Assistant ID | 513695 | 513641 |
| 素材上传 | ImageX / VOD（cn-north-1） | ImageX / VOD（ap-southeast-1） |
| 文件上传字段 | `files`（通用） | `image_file` / `video_file`（分类） |
| 外部依赖 | Chromium 浏览器 | 无 |

---

## 2. 系统架构

### 2.1 模块关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                        API 路由层 (routes/)                      │
│  /v1/videos/international/generations            [POST] 同步     │
│  /v1/videos/international/generations/async       [POST] 异步提交 │
│  /v1/videos/international/generations/async/:id   [GET]  异步查询 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                     控制器层 (controllers/)                      │
│  videos.ts ── generateInternationalSeedanceVideo()               │
│           ── _generateInternationalSeedanceVideoWithHistoryId()  │
│           ── submitInternationalAsyncVideoTask()                 │
│           ── queryAsyncVideoTask()                               │
│  core.ts   ── request() + X-Bogus/X-Gnarly 自动注入             │
│           ── parseRegionFromToken() 区域识别                     │
│           ─── uploadImageBufferForVideo() 素材上传               │
└───────┬──────────────────────────────────────┬──────────────────┘
        │                                      │
┌───────▼──────────┐  ┌────────────────────────▼─────────────────┐
│  签名模块 (lib/)  │  │           上游 API 调用                    │
│  x-bogus.ts      │  │  mweb-api-sg.capcut.com                   │
│  x-gnarly.ts     │  │  imagex-normal-sg.capcutapi.com           │
└──────────────────┘  └──────────────────────────────────────────┘
```

### 2.2 数据流

```
客户端请求
    │
    ▼
路由层 (videos.ts routes)
    │  验证模型、解析参数
    ▼
控制器层 (videos.ts controllers)
    │  1. parseRegionFromToken() → 解析区域
    │  2. getCredit() → 检查积分
    │  3. collectInternationalMaterialFields() → 收集素材
    │  4. uploadImageBufferForVideo() / uploadMediaForVideo() → 上传素材
    │  5. 构建 draft_content + generateBody
    ▼
core.ts request() 函数
    │  1. 构建 query string
    │  2. signXBogus() → URL 追加 X-Bogus 参数
    │  3. getXGnarly() → HTTP 头注入 X-Gnarly
    │  4. 发送到 mweb-api-sg.capcut.com
    ▼
上游返回 history_record_id
    │
    ▼
轮询 get_history_by_ids
    │  每 2~10 秒查询，最多 120 次
    ▼
返回视频 CDN URL → 客户端
```

---

## 3. API 接口定义

### 3.1 同步视频生成

```
POST /v1/videos/international/generations
```

**请求格式：** `multipart/form-data` 或 `application/json`

**参数：**

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| model | string | 否 | `seedance-2.0-fast` | 模型名称 |
| prompt | string | 是 | - | 视频描述，支持 `@1`、`@2` 占位符 |
| ratio | string | 否 | `4:3` | 宽高比：1:1, 4:3, 3:4, 16:9, 9:16 |
| resolution | string | 否 | `720p` | 分辨率：720p, 1080p |
| duration | number | 否 | 4 | 时长：4~15 秒 |
| image_file | file | 否* | - | 图片素材文件 |
| video_file | file | 否* | - | 视频素材文件 |
| image_file_1 ~ image_file_9 | file | 否* | - | 多图片素材 |
| video_file_1 ~ video_file_3 | file | 否* | - | 多视频素材 |
| file_paths / filePaths | string[] | 否* | - | 素材 URL 数组 |

> *至少提供一个素材来源（文件上传或 URL）

**请求头：**

```
Authorization: Bearer {region}-{sessionid}
Content-Type: multipart/form-data
```

**成功响应：**

```json
{
  "created": 1712000000,
  "data": [
    {
      "url": "https://v16-cc.capcut.com/.../video.mp4",
      "revised_prompt": "@1 中的人物开始微笑"
    }
  ]
}
```

**错误响应：**

```json
{
  "errmsg": "shark not pass reject",
  "code": -1
}
```

### 3.2 异步视频生成 — 提交任务

```
POST /v1/videos/international/generations/async
```

参数与同步接口完全一致。

**成功响应：**

```json
{
  "task_id": "async_xxxxxxxx",
  "status": "processing",
  "message": "任务已提交，请使用 task_id 查询结果"
}
```

### 3.3 异步视频生成 — 查询结果

```
GET /v1/videos/international/generations/async/{task_id}
```

**请求头：**

```
Authorization: Bearer {region}-{sessionid}
```

**处理中响应：**

```json
{
  "task_id": "async_xxxxxxxx",
  "status": "processing",
  "message": "任务处理中..."
}
```

**成功响应：**

```json
{
  "task_id": "async_xxxxxxxx",
  "status": "succeeded",
  "data": [
    {
      "url": "https://v16-cc.capcut.com/.../video.mp4",
      "revised_prompt": "@1 中的人物开始微笑"
    }
  ]
}
```

**失败响应：**

```json
{
  "task_id": "async_xxxxxxxx",
  "status": "failed",
  "error": "login error"
}
```

---

## 4. 区域识别与路由

### 4.1 Token 格式

```
{region_prefix}-{sessionid}
```

例如：`sg-abc123def456`、`hk-789ghi012jkl`、`il-mno345pqr678`

### 4.2 区域解析流程

```typescript
parseRegionFromToken(refreshToken: string): RegionInfo
```

1. 检测 `us-` 前缀 → `isUS=true, isInternational=true`
2. 提取前两个字符，查找 `INTERNATIONAL_REGION_MAP`
3. 匹配成功 → `isInternational=true, regionCode=<大写区域码>`
4. 未匹配 → `isCN=true, regionCode="CN"`

### 4.3 RegionInfo 接口

```typescript
interface RegionInfo {
  isUS: boolean;          // 是否为美国区域
  regionCode: string;     // 2 字母大写区域码（CN / SG / HK / ...）
  isInternational: boolean; // 是否为国际版
  isCN: boolean;          // 是否为国内版
}
```

### 4.4 支持的 26 个区域

| 前缀 | 区域 | 前缀 | 区域 | 前缀 | 区域 | 前缀 | 区域 |
|------|------|------|------|------|------|------|------|
| `sg-` | 新加坡 | `hk-` | 香港 | `jp-` | 日本 | `it-` | 意大利 |
| `al-` | 阿尔巴尼亚 | `az-` | 阿塞拜疆 | `bh-` | 巴林 | `ca-` | 加拿大 |
| `cl-` | 智利 | `de-` | 德国 | `gb-` | 英国 | `gy-` | 圭亚那 |
| `il-` | 以色列 | `iq-` | 伊拉克 | `jo-` | 约旦 | `kg-` | 吉尔吉斯 |
| `om-` | 阿曼 | `pk-` | 巴基斯坦 | `pt-` | 葡萄牙 | `sa-` | 沙特 |
| `se-` | 瑞典 | `tr-` | 土耳其 | `tz-` | 坦桑尼亚 | `uz-` | 乌兹别克 |
| `ve-` | 委内瑞拉 | `xk-` | 科索沃 | | | | |

> `INTERNATIONAL_REGION_MAP` 定义在 `src/api/controllers/core.ts` 第 73~100 行。

---

## 5. Shark 反爬签名

### 5.1 签名注入点

在 `core.ts` 的 `request()` 函数中（约第 314~333 行），对所有 `isInternational || isUS` 的请求自动注入：

```
原始请求 URL: https://mweb-api-sg.capcut.com/mweb/v1/aigc_draft/generate?param1=val1&param2=val2
                                              ↓ X-Bogus 签名
签名后 URL:   ...?param1=val1&param2=val2&X-Bogus=Dkdpgh...
                                              ↓ X-Gnarly 签名
HTTP 头:      X-Gnarly: u09tbS3UvgDE...
```

### 5.2 X-Bogus 签名算法

**实现文件：** `src/lib/x-bogus.ts`

**入口函数：**

```typescript
signXBogus(params: string, userAgent: string, data: string = ""): string
```

**算法流程：**

```
输入: queryString + userAgent + bodyData
          │
          ▼
    ┌─────────────┐
    │ 双重 MD5    │  md5(md5(data)), md5(md5(params))
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │ RC4 加密 UA │  密钥 [0, 1, 14]
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │ 自定义 B64  │  → MD5 编码
    └──────┬──────┘
           │
    ┌──────▼──────────────────────┐
    │ 构建 salt 数组 (22 字节)     │
    │ [magic, timestamp, 偏移量,  │
    │  md5 尾字节, 校验和, 0xFF]   │
    └──────┬──────────────────────┘
           │
    ┌──────▼──────┐
    │ filter 重排 │  提取 19 个特定索引
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │ scramble    │  交织前 10 与后 9 元素
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │ RC4 加密    │  密钥 [255]
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │ 前缀 + B64  │  \x02\xff + 自定义 Base64
    └──────┬──────┘
           │
           ▼
    输出: 28 字符 X-Bogus 值
```

**自定义 Base64 字母表：**

```
Dkdpgh4ZKsQB80/Mfvw36XI1R25-WUAlEi7NLboqYTOPuzmFjJnryx9HVGcaStCe
```

**关键参数：**
- 时间戳：`Math.floor(Date.now() / 1000)`（秒级）
- 魔数：`536919696`
- Salt 数组：22 字节（4 字节时间戳 + 4 字节魔数 + 校验和 + 边界标记）

### 5.3 X-Gnarly 签名算法

**实现文件：** `src/lib/x-gnarly.ts`

**入口函数：**

```typescript
getXGnarly(queryString: string, requestBody: string, userAgent: string): string
```

**算法流程：**

```
输入: queryString + requestBody + userAgent
          │
          ▼
    ┌───────────────────┐
    │ 初始化 ChaCha20   │  16 字状态 + 时间戳种子
    │ PRNG 状态          │
    └────────┬──────────┘
             │
    ┌────────▼────────────────────┐
    │ 构建 12 字段数据对象         │
    │ [版本号, MD5(query),         │
    │  MD5(body), MD5(UA),         │
    │  时间戳, 魔数 1938040196,    │
    │  SDK 版本, 校验和...]        │
    └────────┬────────────────────┘
             │
    ┌────────▼───────────┐
    │ 序列化为字节数组    │  字段计数 + 每字段: key + len + value
    └────────┬───────────┘
             │
    ┌────────▼──────────────────┐
    │ 生成 12 个随机密钥字       │  ChaCha20 PRNG
    │ 计算加密轮数 = Σ(low4) + 5 │  范围 5~20
    └────────┬──────────────────┘
             │
    ┌────────▼───────────┐
    │ ChaCha20 加密载荷   │  密钥字 + 轮数
    └────────┬───────────┘
             │
    ┌────────▼──────────────────┐
    │ 计算密钥插入位置            │  迭代 key 字节 + 密文字节
    │ 将密钥字节混入密文          │  mod (length + 1)
    └────────┬──────────────────┘
             │
    ┌────────▼───────────┐
    │ 控制字节 + 自定义   │  0x4B ('K') + 自定义 Base64
    │ Base64 编码         │
    └────────┬───────────┘
             │
             ▼
    输出: ~300 字符 X-Gnarly 头值
```

**自定义 Base64 字母表：**

```
u09tbS3UvgDEe6r-ZVMXzLpsAohTn7mdINQlW412GqBjfYiyk8JORCF5/xKHwacP=
```

**关键参数：**
- 时间戳：`Date.now()`（毫秒级）
- ChaCha20 常数：`[0x61707865, 0x3320646e, 0x79622d32, 0x6b206574]`
- 魔数：`1938040196`
- 加密轮数：5~20（动态计算）

---

## 6. 模型映射

### 6.1 模型名称映射

```typescript
const INTERNATIONAL_SEEDANCE_MODEL_MAP = {
  "jimeng-video-seedance-2.0":      "dreamina_seedance_40_pro",
  "seedance-2.0-pro":               "dreamina_seedance_40_pro",
  "jimeng-video-seedance-2.0-fast": "dreamina_seedance_40",
  "seedance-2.0-fast":              "dreamina_seedance_40",
};
```

### 6.2 权益类型映射

```typescript
const INTERNATIONAL_SEEDANCE_BENEFIT_TYPE_MAP = {
  "jimeng-video-seedance-2.0":      "seedance_20_pro_720p_output",
  "seedance-2.0-pro":               "seedance_20_pro_720p_output",
  "jimeng-video-seedance-2.0-fast": "seedance_20_fast_720p_output",
  "seedance-2.0-fast":              "seedance_20_fast_720p_output",
};
```

### 6.3 辅助 ID

| 参数 | 值 | 说明 |
|------|-----|------|
| Assistant ID | `513641` | 国际版专用 |
| Draft Version | `3.3.9` | 草稿版本号 |
| FPS | `24` | 固定帧率 |

---

## 7. 素材上传

### 7.1 素材字段收集

路由层通过 `collectInternationalMaterialFields(filesMap, body)` 收集素材：

**Multipart 文件字段命名规则：**

| 字段名 | 类型 | 数量限制 |
|--------|------|---------|
| `image_file` | 图片 | 1 |
| `video_file` | 视频 | 1 |
| `image_file_1` ~ `image_file_9` | 图片 | 最多 9 |
| `video_file_1` ~ `video_file_3` | 视频 | 最多 3 |
| `file_paths` / `filePaths` | URL 数组 | 不限类型 |

**约束条件：**
- 至少提供 1 个素材
- 图片最多 9 个，视频最多 3 个，总计最多 12 个

### 7.2 图片上传流程

```
本地文件 / URL
      │
      ▼
读取为 Buffer
      │
      ▼
get_upload_token(scene=2)     ← 获取上传凭证
      │
      ▼
ApplyImageUpload              ← 申请上传位
  → imagex-normal-sg.capcutapi.com
  → AWS Region: ap-southeast-1
      │
      ▼
Upload Binary                 ← 上传二进制数据
      │
      ▼
CommitImageUpload             ← 提交确认
      │
      ▼
返回 URI: tos-alisg-i-{service_id}/{uuid}
```

### 7.3 视频上传流程

```
本地文件 / URL
      │
      ▼
读取为 Buffer
      │
      ▼
get_upload_token(scene=1)     ← 获取上传凭证
      │
      ▼
ApplyUploadInner              ← 申请上传位
  → vod.bytedanceapi.com
      │
      ▼
Upload Binary                 ← 上传二进制数据
      │
      ▼
CommitUploadInner             ← 提交确认
      │
      ▼
返回: { vid, width, height, duration, fps }
```

### 7.4 素材注册表

上传完成后，每个素材注册到 `materialRegistry` Map：

```typescript
materialRegistry.set(fieldKey, {
  material_type: "image" | "video",
  uri?: string,          // 图片 URI
  vid?: string,          // 视频 VID
  width: number,
  height: number,
  duration?: number,
  fps?: number,
});
```

---

## 8. Prompt 解析

### 8.1 占位符规则

`parseOmniPrompt()` 函数解析 prompt 中的素材引用：

| 占位符 | 含义 |
|--------|------|
| `@1` | 引用第 1 个素材 |
| `@2` | 引用第 2 个素材 |
| `@图1` | 引用第 1 张图片 |
| `@image1` | 引用第 1 张图片 |

### 8.2 meta_list 结构

解析后生成 `meta_list` 数组，交替排列文本和素材引用：

```json
[
  { "meta_type": "text", "text": "让" },
  { "meta_type": "image", "material_ref": { "uri": "tos-alisg-i-..." } },
  { "meta_type": "text", "text": "中的人物开始微笑" }
]
```

**隐式引用：** 若 prompt 中无 `@` 引用，所有素材自动插入到 meta_list 开头。

---

## 9. 视频生成流程

### 9.1 生成请求体结构

```json
{
  "submit_id": "uuid",
  "extend": {
    "root_model": "dreamina_seedance_40",
    "m_video_commerce_info": {
      "benefit_type": "seedance_20_fast_720p_output"
    }
  },
  "draft_content": "{\"component_list\":[{\"abilities\":{\"gen_video\":{\"text_to_video_params\":{\"video_gen_inputs\":[{\"unified_edit_input\":{\"material_list\":[...],\"meta_list\":[...]},\"video_aspect_ratio\":\"4:3\",\"seed\":12345,\"model_req_key\":\"dreamina_seedance_40\",\"duration_ms\":4000,\"fps\":24}]}}}]}"
}
```

### 9.2 轮询机制

生成请求返回 `history_record_id` 后，通过 `pollHistoryForVideoUrl()` 轮询结果：

```
请求: POST /mweb/v1/get_history_by_ids
      ↓
状态码映射:
  status == 20  →  处理中，继续等待
  status == 10  →  完成，提取视频 URL
  status == 30  →  失败（服务端错误）
  status == 3   →  失败（fail_code=2038 内容过滤）
      ↓
等待策略: 2000 * min(retryCount+1, 5) ms
      ↓
最多重试: 120 次
初始延迟: 5 秒
```

### 9.3 视频 URL 提取

完成后的视频 URL 提取策略（按优先级）：

1. `/mweb/v1/get_local_item_list` 高质量链接
2. 5 种正则策略从响应中提取 CDN URL
3. `item_list[0]` 字段直接提取

---

## 10. 异步任务系统

### 10.1 架构

```
submitInternationalAsyncVideoTask()
         │
         ├── 创建 AsyncTask 对象
         │   { task_id, status: "processing", _promise, _resolve }
         │
         ├── 持久化到磁盘
         │   tmp/async-tasks/{task_id}.json
         │
         └── 启动异步 IIFE
              │
              ▼
    _generateInternationalSeedanceVideoWithHistoryId()
              │
              ├── onHistoryId 回调 → 保存 historyId 到任务文件
              │                    （崩溃恢复用）
              │
              ├── 成功 → task.status = "succeeded"
              │          task.result = { url, revised_prompt }
              │
              ├── 超时 → task.status 保持 "processing"
              │          可后续通过 historyId 重新查询
              │
              └── 失败 → task.status = "failed"
                         task.error = errorMessage
```

### 10.2 并发控制

```typescript
const MAX_ASYNC_CONCURRENCY = 10;
```

超过上限时返回 HTTP 429 错误。

### 10.3 任务持久化与恢复

**持久化：**
- 任务创建时写入 `tmp/async-tasks/{taskId}.json`
- 获得历史 ID 后更新文件（`historyId` 字段）
- 完成时更新文件（`status` + `result` / `error`）

**恢复（服务重启）：**
```typescript
restoreTasksFromFiles()
  → 读取 tmp/async-tasks/ 目录所有 JSON
  → 筛选 status === "processing" 的任务
  → 若有 historyId → 启动 on-demand 轮询
  → 若超过 24 小时 → 标记为过期并清理
```

### 10.4 查询接口逻辑

```typescript
queryAsyncVideoTask(taskId: string)
  → 内存查找 → 文件查找
  → 有活跃 Promise → await 返回
  → status=processing + historyId → on-demand 查询
  → status=succeeded → 直接返回 result
  → status=failed → 直接返回 error
```

---

## 11. 错误处理

### 11.1 错误码映射

| 错误信息 | 含义 | HTTP 状态码 |
|----------|------|------------|
| `shark not pass reject` | X-Bogus/X-Gnarly 签名验证失败 | 200 (上游返回) |
| `login error` | sessionid 无效或过期 | 200 (上游返回) |
| `Not enough credits` / `INSUFFICIENT_POINTS` | 积分不足 | 200 (上游返回) |
| `fail_code: 2038` | 内容安全过滤触发 | 200 (上游返回) |
| 国际 Seedance 接口仅接受国际 token | Token 前缀不在区域映射表中 | 200 |

### 11.2 重试策略

`core.ts` 的 `request()` 函数内置重试：

```
最大重试次数: 3
单次超时: 45 秒
退避策略: 固定 5 秒延迟
重试条件: HTTP 错误 或 网络超时
```

---

## 12. 关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `src/api/routes/videos.ts` | 国际版路由定义（3 个端点） |
| `src/api/controllers/videos.ts` | 视频生成核心逻辑（同步/异步/轮询） |
| `src/api/controllers/core.ts` | 区域识别、签名注入、请求封装、素材上传 |
| `src/lib/x-bogus.ts` | X-Bogus 签名算法（MD5 + RC4 + Base64） |
| `src/lib/x-gnarly.ts` | X-Gnarly 签名算法（ChaCha20 + Base64） |
| `src/lib/browser-service.ts` | 浏览器代理服务（仅国内版使用） |

---

## 13. 调用示例

### 13.1 同步生成（JSON + URL）

```bash
curl -X POST http://localhost:8000/v1/videos/international/generations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sg-your_sessionid" \
  -d '{
    "model": "seedance-2.0-fast",
    "prompt": "@1 中的人物开始微笑",
    "ratio": "4:3",
    "resolution": "720p",
    "duration": 4,
    "file_paths": ["https://example.com/photo.jpg"]
  }'
```

### 13.2 同步生成（Multipart 文件上传）

```bash
curl -X POST http://localhost:8000/v1/videos/international/generations \
  -H "Authorization: Bearer hk-your_sessionid" \
  -F "model=seedance-2.0-fast" \
  -F "prompt=@1 中的人物开始微笑" \
  -F "ratio=4:3" \
  -F "duration=4" \
  -F "image_file=@/path/to/image.jpg"
```

### 13.3 异步生成

```bash
# 提交任务
curl -X POST http://localhost:8000/v1/videos/international/generations/async \
  -H "Authorization: Bearer il-your_sessionid" \
  -F "model=seedance-2.0-fast" \
  -F "prompt=@1 中的人物开始微笑" \
  -F "image_file=@/path/to/image.jpg"

# 查询结果
curl http://localhost:8000/v1/videos/international/generations/async/async_xxxxxxxx \
  -H "Authorization: Bearer il-your_sessionid"
```

### 13.4 多素材混合

```bash
curl -X POST http://localhost:8000/v1/videos/international/generations \
  -H "Authorization: Bearer tr-your_sessionid" \
  -F "model=seedance-2.0-pro" \
  -F "prompt=@1 和 @2 两人开始跳舞" \
  -F "ratio=16:9" \
  -F "duration=5" \
  -F "image_file_1=@/path/to/person1.jpg" \
  -F "image_file_2=@/path/to/person2.jpg"
```
