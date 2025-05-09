**平台管理模块 API 文档**

本文档描述了草莓派SaaS平台的平台管理模块 (`ms-module-platform`) 提供的RESTful API接口。这些接口主要供平台管理员使用，用于管理租户、平台用户、平台角色和权限，以及查询操作日志。

**基础URL:** `/api/v1/platform` (示例，具体取决于网关配置)

**认证与授权:** 所有接口需要平台用户的有效认证Token，该Token通过平台登录接口获取。权限通过平台用户的角色及分配的平台权限进行控制。

**平台登录认证接口:** 平台用户登录认证接口通常位于独立的认证模块 (`ms-module-auth`)，例如 `/auth/platform/login`。成功登录后返回包含认证Token的响应。

---

## 1. 统一响应结构

所有API接口都将采用统一的响应格式，以确保一致性和易于处理。

### 1.1 成功响应

成功响应包含业务状态码、消息和业务数据。

```json
{
  "code": 200, // 业务状态码，200表示成功
  "message": "Success", // 响应消息
  "data": {
    // 业务数据，具体结构取决于接口
  }
}
```

特殊情况：创建租户接口（异步）返回状态码 `202 Accepted`。

### 1.2 错误响应

错误响应包含业务状态码（非200）、错误消息和可选的错误详情。

```json
{
  "code": 50000, // 业务错误码，例如参数校验失败、无权限等
  "message": "Parameter validation failed", // 错误消息
  "data": null, // 错误时通常为null
  "details": {
    // 可选的错误详情，如字段校验失败的具体信息
  }
}
```

---

## 2. 敏感数据处理说明

为了保障数据安全，平台管理模块对平台用户和租户信息中的敏感数据进行了特殊处理。

* **存储：** 租户数据库连接密码 (`dataSourceConfig` 中的 password)、租户联系人的手机号 (`contactPhone`) 和邮箱 (`contactEmail`) 在数据库中**必须进行加密存储**（采用您选择的对称加密）。平台用户密码 (`platform_user.password`) 必须进行**哈希加密**存储（如使用 BCrypt）。
* **API 返回 (普通平台管理员):**
  * 平台用户密码 (`platform_user.password`) **绝不返回**。
  * 在查询租户列表和详情时，租户的联系手机号 (`contactPhone`) 和联系邮箱 (`contactEmail`) 会进行**脱敏处理**后返回（例如：手机号 `138****1234`，邮箱 `user@***.com`）。
  * 租户的数据库连接配置 (`dataSourceConfig`) **不返回**给普通平台管理员。
* **API 返回 (平台超级管理员):**
  * 平台用户密码 (`platform_user.password`) **绝不返回**。
  * 在查询租户详情时，可以看到租户的**原始**联系信息（手机号、邮箱）。
  * 租户的数据库连接配置 (`dataSourceConfig`) 将返回，但其中的密码字段仍应是**加密存储后的密文**，不直接返回明文密码。
* **API 修改限制：**
  * 普通平台管理员在更新租户信息时，**无权修改**租户的联系方式和数据库连接配置。
  * 只有平台**超级管理员**才能通过更新租户信息 API 修改租户的联系方式和数据库连接配置（提交的明文密码后端需加密存储）。

---

## 3. 平台管理 (Platform Management - Tenants)

这些 API 接口用于管理租户。普通平台管理员和超级管理员都可以访问，但具体权限取决于分配的角色。

### 3.1 创建租户

触发异步租户数据库初始化过程。

* **URL:** `/tenants`

* **方法:** `POST`

* **描述:** 提交创建新租户的请求，系统异步进行数据库初始化。成功后通过消息队列发送通知。

* **权限:** 需要平台超级管理员或拥有 `platform:tenant:create` 权限的普通平台管理员。

* **请求体:** `application/json`

  ```json
  {
    "tenantCode": "string",  // 租户编码 (平台内唯一，必填)
    "tenantName": "string",  // 租户名称 (必填)
    "contactPerson": "string", // 联系人 (可选)
    "contactPhone": "string",  // 联系电话 (可选，入库需加密)
    "contactEmail": "string",  // 联系邮箱 (可选，入库需加密)
    "remark": "string"       // 备注 (可选)
    // 不包含数据库连接信息，由平台自动生成或后续配置
  }
  ```

* **响应:** `application/json`

  * **状态码:** `202 Accepted` (表示请求已接受并正在异步处理)

  * **响应体:** (符合统一响应结构)

    ```json
    {
      "code": 202,
      "message": "Tenant creation request accepted and is being processed asynchronously.",
      "data": {
        "tenantId": 1234567890, // 新创建租户的ID
        "status": "PROVISIONING", // 初始化状态
        "receivedTime": "YYYY-MM-DDTHH:mm:ssZ" // 请求接收时间
      }
    }
    ```

  * **异步通知:** 最终结果通过消息队列通知，消息体包含 `tenantId`, `operationType`, `status`, `timestamp`, `errorMessage` 等。

### 3.2 查询租户列表

获取平台所有租户的信息列表，支持条件查询和分页。

* **URL:** `/tenants`

* **方法:** `GET`

* **描述:** 分页查询租户列表。普通平台管理员看到脱敏敏感信息，超级管理员看到原始信息。不返回租户用户详情数据。

* **权限:** 需要平台超级管理员或拥有 `platform:tenant:view` 权限的普通平台管理员。

* **请求参数:**

  * `page`: int (页码，默认1)
  * `size`: int (每页数量，默认10)
  * `tenantCode`: string (可选，按租户编码模糊查询)
  * `tenantName`: string (可选，按租户名称模糊查询)
  * `status`: string (可选，按状态过滤，如 ACTIVE, INACTIVE, EXPIRED, PROVISIONING)
  * `createTimeStart`: string (可选，创建时间范围起始，ISO 8601格式)
  * `createTimeEnd`: string (可选，创建时间范围结束，ISO 8601格式)

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构)

    ```json
    {
      "code": 200,
      "message": "Success",
      "data": {
        "total": 100, // 总条数
        "page": 1,
        "size": 10,
        "records": [
          {
            "tenantId": 1234567890,
            "tenantCode": "tenant_001",
            "tenantName": "演示租户公司A",
            "status": "ACTIVE",
            "createTime": "YYYY-MM-DDTHH:mm:ssZ",
            "updateTime": "YYYY-MM-DDTHH:mm:ssZ",
            "expireTime": "YYYY-MM-DDTHH:mm:ssZ", // 过期时间
            "contactPerson": "联系人A", // 普通管理员脱敏，超级管理员原始
            "contactPhone": "138****1234", // 普通管理员脱敏，超级管理员原始
            "contactEmail": "user@***.com", // 普通管理员脱敏，超级管理员原始
            "userCount": 50, // 租户下的用户数量 (只显示数量)
            "defaultLanguage": "zh_CN", // 固定配置字段
            "timeZone": "Asia/Shanghai", // 固定配置字段
            "remark": "备注信息"
            // 不包含 dataSourceConfig 和租户用户列表
          }
          // ... 更多租户记录
        ]
      }
    }
    ```

  * **敏感数据处理:** `contactPhone`, `contactEmail` 对普通平台管理员脱敏，超级管理员原始。`dataSourceConfig` 不返回给普通平台管理员。

### 3.3 查询租户详情

获取单个租户的详细信息。

* **URL:** `/tenants/{tenantId}`

* **方法:** `GET`

* **描述:** 获取指定租户的详细信息。敏感信息处理方式同列表查询。

* **权限:** 需要平台超级管理员或拥有 `platform:tenant:view` 权限的普通平台管理员。

* **路径参数:**

  * `tenantId`: long (租户ID)

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构，结构类似列表查询的单条记录，增加更多详细字段)

    ```json
    {
      "code": 200,
      "message": "Success",
      "data": {
        "tenantId": 1234567890,
        "tenantCode": "tenant_001",
        "tenantName": "演示租户公司A",
        "status": "ACTIVE",
        "createTime": "YYYY-MM-DDTHH:mm:ssZ",
        "updateTime": "YYYY-MM-DDTHH:mm:ssZ",
        "expireTime": "YYYY-MM-DDTHH:mm:ssZ",
        "contactPerson": "联系人A", // 普通管理员脱敏，超级管理员原始
        "contactPhone": "138****1234", // 普通管理员脱敏，超级管理员原始
        "contactEmail": "user@***.com", // 普通管理员脱敏，超级管理员原始
        "userCount": 50, // 租户下的用户数量 (只显示数量)
        "defaultLanguage": "zh_CN", // 固定配置字段
        "timeZone": "Asia/Shanghai", // 固定配置字段
        "remark": "备注信息",
        "dataSourceConfig": { // 数据库连接信息 (敏感！)
             // 仅返回给超级管理员，且密码等敏感部分仍是加密存储后的密文
             "url": "jdbc:mysql://...",
             "username": "...",
             "password": "加密后的密文"
         }
        // 不包含租户用户列表
      }
    }
    ```

  * **敏感数据处理:** `contactPhone`, `contactEmail` 对普通平台管理员脱敏，超级管理员原始。`dataSourceConfig` 仅返回给超级管理员，且内部敏感信息（如密码）需处理。普通平台管理员无权查看或修改此字段。

### 3.4 更新租户信息

更新租户的基本信息和固定配置。

* **URL:** `/tenants/{tenantId}`

* **方法:** `PUT`

* **描述:** 更新指定租户的信息，包括默认语言和时区。普通平台管理员无权修改敏感信息（联系方式、数据源配置）。

* **权限:** 需要平台超级管理员或拥有 `platform:tenant:edit` 权限的普通平台管理员。

* **路径参数:**

  * `tenantId`: long (租户ID)

* **请求体:** `application/json`

  ```json
  {
    "tenantName": "string",  // 租户名称 (可选)
    "contactPerson": "string", // 联系人 (可选，普通管理员修改无效或报错)
    "contactPhone": "string",  // 联系电话 (可选，普通管理员修改无效或报错)
    "contactEmail": "string",  // 联系邮箱 (可选，普通管理员修改无效或报错)
    "defaultLanguage": "string", // 默认语言 (可选)
    "timeZone": "string", // 时区 (可选)
    "remark": "string",      // 备注 (可选)
    "dataSourceConfig": { // 数据库连接信息 (敏感！)
         // 仅超级管理员可在请求体中包含并修改此字段，普通管理员包含此字段将被忽略或报错
         "url": "string",
         "username": "string",
         "password": "string" // 明文传输，后端接收后需加密存储
     }
    // 不应包含 status 字段在此接口中修改
  }
  ```

* **响应:** `application/json`

  * **状态码:** `200 OK`
  * **响应体:** (符合统一响应结构，返回更新后租户的详细信息，结构类似查询详情)
  * **敏感数据处理:** 普通平台管理员提交的请求体中包含敏感字段（如联系方式、`dataSourceConfig`）将被忽略或返回权限错误。只有超级管理员的请求体中这些字段才会被处理。

### 3.5 启用/禁用租户

修改租户的状态（逻辑标记）。

* **URL:** `/tenants/{tenantId}/status`

* **方法:** `PUT`

* **描述:** 修改指定租户的逻辑状态（启用/禁用）。

* **权限:** 需要平台超级管理员或拥有 `platform:tenant:status_manage` 权限的普通平台管理员。

* **路径参数:**

  * `tenantId`: long (租户ID)

* **请求体:** `application/json`

  ```json
  {
    "status": "string" // 目标状态: "ACTIVE" 或 "INACTIVE"
    // 理论上也可以支持 EXPIRED, PROVISIONING 但通常不由API直接修改
  }
  ```

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构)

    ```json
    {
      "code": 200,
      "message": "Tenant status updated successfully.",
      "data": {
         "tenantId": 1234567890,
         "status": "ACTIVE" // 返回更新后的状态
      }
    }
    ```

### 3.6 删除租户

删除指定租户（逻辑或物理）。

* **URL:** `/tenants/{tenantId}`

* **方法:** `DELETE`

* **描述:** 删除指定租户。具体是逻辑删除（标记为已删除状态）还是物理删除（删除平台记录并删除租户数据库）需根据实际实现确定，物理删除风险高，通常需要高级权限和操作确认。

* **权限:** 需要平台超级管理员或拥有 `platform:tenant:delete` 权限的普通平台管理员（物理删除权限应更高）。

* **路径参数:**

  * `tenantId`: long (租户ID)

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构)

    ```json
    {
      "code": 200,
      "message": "Tenant deleted successfully.",
      "data": null // 或返回删除结果信息
    }
    ```

  * **建议：** 生产环境优先考虑逻辑删除，物理删除需谨慎并伴随数据备份。

### 3.7 批量删除租户

批量删除多个租户。

* **URL:** `/tenants/batch-delete`

* **方法:** `POST` (建议使用 POST 携带请求体，DELETE 方法带请求体兼容性可能不好)

* **描述:** 批量删除指定的多个租户。删除类型（逻辑/物理）应与单个删除一致或有更高要求。

* **权限:** 需要平台超级管理员或拥有 `platform:tenant:batch_delete` 权限的普通平台管理员。

* **请求体:** `application/json`

  ```json
  {
    "tenantIds": [1234567890, 1234567891, ...] // 租户ID列表 (必填)
  }
  ```

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构)

    ```json
    {
      "code": 200,
      "message": "Batch deletion request processed.",
      "data": {
         "successCount": 5, // 成功删除数量
         "failedCount": 1,  // 失败数量
         "failedTenantIds": [1234567892] // 失败的租户ID列表 (可选)
      }
    }
    ```

### 3.8 查询租户下的用户数量

查询指定租户的用户数量（不返回用户列表）。

* **URL:** `/tenants/{tenantId}/user-count`

* **方法:** `GET`

* **描述:** 查询指定租户在租户数据库中的用户总数。此接口会触发对租户数据库的查询（通过多租户数据源切换实现）。不返回租户用户详情。

* **权限:** 需要平台超级管理员或拥有 `platform:tenant:user_count:view` 权限的普通平台管理员。

* **路径参数:**

  * `tenantId`: long (租户ID)

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构)

    ```json
    {
      "code": 200,
      "message": "Success",
      "data": {
        "tenantId": 1234567890,
        "userCount": 50 // 租户下的用户总数量
      }
    }
    ```

---

## 4. 平台用户管理 (Platform User Management - Ordinary Admins)

这些 API 接口主要用于**平台超级管理员**管理**普通平台管理员**的账号。普通平台管理员**无权**访问这些接口。权限控制粒度较高。

* **基础URL:** `/api/v1/platform/ordinary-admins`

### 4.1 创建普通平台管理员

* **URL:** `/`

* **方法:** `POST`

* **描述:** 创建新的普通平台管理员账号。

* **权限:** **仅限平台超级管理员**，需要拥有 `platform:ordinary_admin:create` 权限。

* **请求体:** `application/json`

  ```json
  {
    "username": "string", // 用户名 (必填，平台内唯一)
    "password": "string", // 初始密码 (必填，明文传输，后端接收后需哈希加密存储)
    "nickname": "string", // 昵称 (必填)
    "email": "string", // 邮箱 (可选)
    "phone": "string", // 手机号 (可选)
    "status": "string", // 初始用户状态 (可选): "ENABLED", "DISABLED"，默认ENABLED
    "remark": "string", // 备注 (可选)
    "roleIds": [long] // 分配给该用户的平台角色ID列表 (必填，可以为空数组)
  }
  ```

* **响应:** `application/json` (符合统一响应结构，返回新创建用户的概览信息)

  ```json
  {
    "code": 200,
    "message": "Success",
    "data": {
      "userId": 1234567890, // 新用户ID
      "status": "ENABLED" // 初始化状态
    }
  }
  ```

### 4.2 查询普通平台管理员列表

* **URL:** `/`

* **方法:** `GET`

* **描述:** 查询普通平台管理员账号列表。支持条件查询和分页。

* **权限:** **仅限平台超级管理员**，需要拥有 `platform:ordinary_admin:view` 权限。

* **请求参数:**

  * `page`: int (页码，默认1)
  * `size`: int (每页数量，默认10)
  * `username`: string (可选，按用户名模糊查询)
  * `nickname`: string (可选，按昵称模糊查询)
  * `status`: string (可选，按状态过滤，如 ENABLED, DISABLED)
  * `createTimeStart`: string (可选，创建时间范围起始，ISO 8601格式)
  * `createTimeEnd`: string (可选，创建时间范围结束，ISO 8601格式)
  * `roleId`: long (可选，按关联的平台角色ID过滤)

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构)

    ```json
    {
      "code": 200,
      "message": "Success",
      "data": {
        "total": 100, // 总条数
        "page": 1,
        "size": 10,
        "records": [
          {
            "id": 1234567890,
            "username": "user_001",
            "nickname": "用户A",
            "email": "userA@example.com",
            "phone": "13812345678",
            "status": "ENABLED",
            "createTime": "YYYY-MM-DDTHH:mm:ssZ",
            "updateTime": "YYYY-MM-DDTHH:mm:ssZ",
            "lastLoginTime": "YYYY-MM-DDTHH:mm:ssZ",
            "remark": "备注信息",
            "roleNames": ["角色1名称", "角色2名称"] // 关联的角色名称列表
            // 不包含 password 字段
          }
          // ... 更多用户记录
        ]
      }
    }
    ```

### 4.3 查询单个普通平台管理员详情

* **URL:** `/{userId}`

* **方法:** `GET`

* **描述:** 查询单个普通平台管理员账号详情。

* **权限:** **仅限平台超级管理员**，需要拥有 `platform:ordinary_admin:view` 权限。

* **路径参数:** `userId`: long (平台用户ID)

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构，结构与查询列表的单条记录**相同**)

    ```json
    {
      "code": 200,
      "message": "Success",
      "data": {
        "id": 1234567890,
        "username": "user_001",
        "nickname": "用户A",
        "email": "userA@example.com",
        "phone": "13812345678",
        "status": "ENABLED",
        "createTime": "YYYY-MM-DDTHH:mm:ssZ",
        "updateTime": "YYYY-MM-DDTHH:mm:ssZ",
        "lastLoginTime": "YYYY-MM-DDTHH:mm:ssZ",
        "remark": "备注信息",
        "roleNames": ["角色1名称", "角色2名称"] // 关联的角色名称列表
        // 不包含 password 字段
      }
    }
    ```

### 4.4 更新普通平台管理员信息

* **URL:** `/{userId}`

* **方法:** `PUT`

* **描述:** 更新普通平台管理员账号信息及其关联的平台角色。用户名和密码不在此接口修改。

* **权限:** **仅限平台超级管理员**，需要拥有 `platform:ordinary_admin:edit` 权限。

* **路径参数:** `userId`: long

* **请求体:** `application/json`

  ```json
  {
    "nickname": "string", // 昵称 (可选)
    "email": "string", // 邮箱 (可选)
    "phone": "string", // 手机号 (可选)
    "status": "string", // 用户状态 (可选): "ENABLED", "DISABLED"
    "remark": "string", // 备注 (可选)
    "addRoleIds": [long], // 要添加的角色ID列表 (必填，可以为空数组)
    "removeRoleIds": [long] // 要移除的角色ID列表 (必填，可以为空数组)
    // 不包含 username 和 password 在此接口修改
  }
  ```

* **响应:** `application/json` (符合统一响应结构，返回更新成功状态)

### 4.5 删除普通平台管理员

* **URL:** `/{userId}`
* **方法:** `DELETE`
* **描述:** **物理删除**普通平台管理员账号记录。**注意：** 不删除该用户相关的操作数据，应用逻辑需处理好其他表中引用该用户ID的**外键**（例如将引用设为 NULL 或指向一个“已删除用户”记录）。
* **权限:** **仅限平台超级管理员**，需要拥有 `platform:ordinary_admin:delete` 权限。
* **路径参数:** `userId`: long
* **响应:** `application/json` (符合统一响应结构，返回删除成功状态)

---

## 5. 平台用户个人信息管理 (Ordinary Admin Self-Service)

这些 API 接口供**当前登录的普通平台管理员**修改自己的个人信息。

* **基础URL:** `/api/v1/platform/users/me`

### 5.1 修改当前登录平台用户的个人信息

* **URL:** `/` (或者 `/profile`)

* **方法:** `PUT`

* **描述:** 当前登录的平台用户更新自己的个人信息。

* **权限:** 需要平台用户的有效登录Token，且拥有 `platform:user:update_profile` 或类似权限（通常赋予所有普通平台管理员）。

* **请求体:** `application/json`

  ```json
  {
    "email": "string", // 邮箱 (可选)
    "phone": "string", // 手机号 (可选)
    "remark": "string" // 备注 (可选)
    // 不能包含 userId, username, nickname, status, roleIds 等字段
  }
  ```

* **响应:** `application/json` (符合统一响应结构，返回更新成功状态)

### 5.2 修改当前登录平台用户的密码

* **URL:** `/password`

* **方法:** `PUT` (或 `POST`)

* **描述:** 当前登录的普通平台管理员修改自己的登录密码。需要提供原密码进行验证。

* **权限:** 需要平台用户的有效登录Token，且拥有 `platform:user:change_password` 或类似权限。

* **请求体:** `application/json`

  ```json
  {
    "oldPassword": "string", // 原密码 (必填)
    "newPassword": "string", // 新密码 (必填)
    "confirmPassword": "string" // 确认新密码 (必填)
    // 密码一致性和复杂度校验主要由前端负责，后端进行基本安全检查
  }
  ```

* **响应:** `application/json` (符合统一响应结构，返回修改成功状态)

---

## 6. 平台超级管理员密码管理 (Super Admin Self-Service)

这些 API 接口供**当前登录的平台超级管理员**管理自己的密码，采用邮箱验证的异步两步流程。

* **基础URL:** `/api/v1/platform/super-admins`

### 6.1 请求发送修改密码邮箱验证码 (Super Admin)

* **URL:** `/email-verification-code`

* **方法:** `POST`

* **描述:** 请求系统向当前登录的超级管理员的绑定邮箱发送修改密码验证码。

* **权限:** 需要平台超级管理员的有效登录Token。

* **请求体:** `application/json` (通常为空，或者包含标识当前超级管理员的字段如果URL不足够)

  ```json
  {} // 或 { "userId": 1 }
  ```

* **响应:** `application/json` (符合统一响应结构，返回发送成功状态)

  ```json
  {
    "code": 200,
    "message": "Verification code sent to your email.",
    "data": null // 或包含一个任务ID，或冷却时间提示
  }
  ```

### 6.2 提交新密码和验证码 (Super Admin)

* **URL:** `/password`

* **方法:** `PUT` (或 `POST`)

* **描述:** 当前登录的超级管理员提交新密码和收到的邮箱验证码以完成密码修改。

* **权限:** 需要平台超级管理员的有效登录Token。

* **请求体:** `application/json`

  ```json
  {
    "newPassword": "string", // 新密码 (必填)
    "confirmPassword": "string", // 确认新密码 (必填)
    "verificationCode": "string" // 接收到的邮箱验证码 (必填)
    // 密码一致性和复杂度校验主要由前端负责，后端进行基本安全检查
  }
  ```

* **响应:** `application/json` (符合统一响应结构，返回修改成功状态)

---

## 7. 平台角色管理 (Platform Role Management)

这些 API 接口用于管理平台级别的角色，用于组织平台用户的权限。这些接口可由平台超级管理员或拥有相应权限的普通平台管理员访问。

* **基础URL:** `/api/v1/platform/roles`

### 7.1 查询平台角色列表

* **URL:** `/`

* **方法:** `GET`

* **描述:** 查询平台角色列表。支持条件查询和分页。

* **权限:** 需要平台超级管理员或拥有 `platform:role:view` 权限的普通平台管理员。

* **请求参数:**

  * `page`: int (页码，默认1)
  * `size`: int (每页数量，默认10)
  * `roleCode`: string (可选，按角色编码模糊查询)
  * `roleName`: string (可选，按角色名称模糊查询)
  * `sortBy`: string (可选，排序字段，如 roleCode, createTime)
  * `sortOrder`: string (可选，排序方式，如 asc, desc)

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构，返回平台角色列表)

    ```json
    {
      "code": 200,
      "message": "Success",
      "data": {
        "total": 100,
        "page": 1,
        "size": 10,
        "records": [
          {
            "id": 1234567890, // 角色ID
            "roleCode": "PLATFORM_ADMIN", // 角色编码
            "roleName": "平台管理员", // 角色名称
            "description": "拥有平台所有管理权限", // 描述
            "createTime": "YYYY-MM-DDTHH:mm:ssZ",
            "updateTime": "YYYY-MM-DDTHH:mm:ssZ",
            "permissionCount": 50 // 关联的权限数量 (可选添加)
          }
          // ... 更多角色记录
        ]
      }
    }
    ```

### 7.2 查询单个平台角色详情 (含权限)

* **URL:** `/{roleId}`

* **方法:** `GET`

* **描述:** 查询单个平台角色的详细信息，包括关联的权限列表。

* **权限:** 需要平台超级管理员或拥有 `platform:role:view` 权限的普通平台管理员。

* **路径参数:** `roleId`: long

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构，返回角色详情及关联的平台权限列表)

    ```json
    {
      "code": 200,
      "message": "Success",
      "data": {
        "id": 1234567890,
        "roleCode": "PLATFORM_ADMIN",
        "roleName": "平台管理员",
        "description": "拥有平台所有管理权限",
        "remark": "内部使用",
        "createTime": "YYYY-MM-DDTHH:mm:ssZ",
        "updateTime": "YYYY-MM-DDTHH:mm:ssZ",
        "permissions": [ // 关联的平台权限列表
          {
            "permissionId": 1,
            "permissionCode": "platform:tenant:create",
            "permissionName": "创建租户"
          },
           {
            "permissionId": 2,
            "permissionCode": "platform:tenant:view",
            "permissionName": "查看租户"
          }
          // ... 更多权限
        ]
      }
    }
    ```

### 7.3 创建平台角色

* **URL:** `/`

* **方法:** `POST`

* **描述:** 创建新的平台角色。

* **权限:** 需要平台超级管理员或拥有 `platform:role:create` 权限的普通平台管理员。

* **请求体:** `application/json`

  ```json
  {
    "roleCode": "string", // 角色编码 (必填，平台内唯一)
    "roleName": "string", // 角色名称 (必填)
    "description": "string", // 描述 (可选)
    "remark": "string", // 备注 (可选)
    "permissionIds": [long] // 关联的平台权限ID列表 (必填，可以为空数组)
  }
  ```

* **响应:** `application/json` (符合统一响应结构，返回新创建角色的基本信息)

### 7.4 更新平台角色 (含权限分配)

* **URL:** `/{roleId}`

* **方法:** `PUT`

* **描述:** 更新平台角色的信息及其关联的权限。角色编码通常不可修改。

* **权限:** 需要平台超级管理员或拥有 `platform:role:edit` 权限的普通平台管理员。

* **路径参数:** `roleId`: long

* **请求体:** `application/json`

  ```json
  {
    "roleName": "string", // 角色名称 (可选)
    "description": "string", // 描述 (可选)
    "remark": "string", // 备注 (可选)
    "permissionIds": [long] // 完整的平台权限ID列表 (必填，会覆盖原有权限)
    // roleCode 通常不可修改
  }
  ```

* **响应:** `application/json` (符合统一响应结构，返回更新成功状态)

### 7.5 删除平台角色

* **URL:** `/{roleId}`
* **方法:** `DELETE`
* **描述:** 删除平台角色。如果该角色有关联用户，阻止删除，并提示用户先解除用户与角色的关联。
* **权限:** 需要平台超级管理员或拥有 `platform:role:delete` 权限的普通平台管理员。
* **路径参数:** `roleId`: long
* **响应:** `application/json` (符合统一响应结构，返回删除成功状态)

---

## 8. 平台权限管理 (Platform Permission Management)

这些 API 接口用于查询平台级别的权限列表，通常与平台角色管理结合使用，以便为角色分配权限。不提供对权限本身的增删改接口。

* **基础URL:** `/api/v1/platform/permissions`

### 8.1 查询平台权限列表

* **URL:** `/`

* **方法:** `GET`

* **描述:** 查询平台所有可用的权限列表。这些权限定义了平台管理员可以执行的各项操作。

* **权限:** 需要平台超级管理员或拥有 `platform:permission:view` 权限的普通平台管理员（通常与角色管理权限关联）。

* **请求参数:**

  * `page`: int (页码，默认1)
  * `size`: int (每页数量，默认10)
  * `permissionCode`: string (可选，按权限编码模糊查询)
  * `permissionName`: string (可选，按权限名称模糊查询)
  * `sortBy`: string (可选，排序字段，如 permissionCode, permissionName)
  * `sortOrder`: string (可选，排序方式，如 asc, desc)
  * `moduleCode`: string (可选，按所属模块过滤，如果权限有模块划分)
  * `category`: string (可选，按类别过滤，如果权限有分类)

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构)

    ```json
    {
      "code": 200,
      "message": "Success",
      "data": {
        "total": 50,
        "page": 1,
        "size": 10,
        "records": [
          {
            "id": 1, // 权限ID
            "permissionCode": "platform:tenant:create", // 权限编码
            "permissionName": "创建租户", // 权限名称
            "description": "允许用户创建新的租户记录和初始化数据库" // 权限描述
          },
          {
            "id": 2,
            "permissionCode": "platform:tenant:view",
            "permissionName": "查看租户",
            "description": "允许用户查看租户列表和详情"
          }
          // ... 更多权限记录
        ]
      }
    }
    ```

---

## 9. 平台操作日志管理 (Platform Operation Log Management)

这些 API 接口用于查询平台管理员对租户等平台资源的操作日志。

* **基础URL:** `/api/v1/platform/operation-logs`

### 9.1 查询平台操作日志列表

* **URL:** `/`

* **方法:** `GET`

* **描述:** 查询平台操作日志列表。支持按操作人、租户、操作类型、时间范围过滤。日志按操作时间默认倒序排列。

* **权限:** 需要平台超级管理员或拥有 `platform:operation_log:view` 权限的普通平台管理员。

* **请求参数:**

  * `page`: int (页码，默认1)
  * `size`: int (每页数量，默认10)
  * `operatorUserId`: long (可选，按操作人用户ID过滤)
  * `operatorNickname`: string (可选，按操作人昵称模糊查询)
  * `operationType`: string (可选，按操作类型过滤)
  * `targetTenantId`: long (可选，按被操作租户ID过滤)
  * `targetTenantName`: string (可选，按被操作租户名称模糊查询)
  * `startTime`: string (可选，操作时间范围起始，ISO 8601格式)
  * `endTime`: string (可选，操作时间范围结束，ISO 8601格式)
  * **注意：** 不支持排序参数，默认按 `operation_time` 倒序排列。

* **响应:** `application/json`

  * **状态码:** `200 OK`

  * **响应体:** (符合统一响应结构)

    ```json
    {
      "code": 200,
      "message": "Success",
      "data": {
        "total": 1000,
        "page": 1,
        "size": 10,
        "records": [
          {
            "id": 10001, // 日志ID
            "operatorUserId": 1, // 操作人用户ID
            "operatorNickname": "admin", // 操作人用户名
            "operationType": "CREATE_TENANT", // 操作类型
            "targetTenantId": 1234567890, // 被操作租户ID
            "targetTenantName": "演示租户公司A", // 被操作租户名称
            "operationTime": "YYYY-MM-DDTHH:mm:ssZ" // 操作时间
          },
           {
            "id": 10002,
            "operatorUserId": 5,
            "operatorUsername": "ordinary_admin_001",
            "operationType": "DISABLE_TENANT",
            "targetTenantId": 1234567891,
            "targetTenantName": "演示租户公司B",
            "operationTime": "YYYY-MM-DDTHH:mm:ssZ"
          }
          // ... 更多日志记录，按时间倒序
        ]
      }
    }
    ```

---

