# Ajax API

通常希望与 Azkaban 交互而无需使用 Web UI。  Azkaban 有一些暴露的 ajax 调用，可以通过 curl 或其他一些 HTTP 请求客户端访问。  所有 API 调用都首先需要正确的身份验证。 

## 一、认证 (Authenticate)

- **Method:** POST  

- **Request URL:** /?action=login  

- **Parameter Location:** Request Query String  

此 API 有助于验证用户并提供 `session.id` 作为回应。 

一旦 `session.id` 已返回，直到会话过期，此 ID 可用于执行任何已授予适当权限的 API 请求。   如果您注销、更改机器、浏览器或位置，如果 Azkaban 重新启动或会话过期，会话就会过期。  默认会话超时为 24 小时（一天）。无论会话是否过期，您都可以重新登录。对于同一用户，新会话将始终覆盖旧会话。 

**重要的，**  `session.id` 应该为几乎所有 API 调用提供（认证除外）。 `session.id` 可以简单地作为请求参数之一附加，或通过 cookie 设置： `azkaban.browser.session.id`.  下面的两个 HTTP 请求是等效的： 

```sh
# sherlock @ mbp in ~ [11:29:20]
$ curl -k -X POST --data "action=login&username=azkaban&password=azkaban" https://10.0.15.131:8443
{
  "session.id" : "9aafa158-d1c4-45ee-9892-3416130ca525",
  "status" : "success"
}%
```

## 二、获取项目流 (Fetch Flows of a Project) 

给定项目名称，此 API 调用会获取该项目的所有流 ID。

- **Method:** GET  

- **Request URL:** /manager?ajax=fetchprojectflows  
- **Parameter Location:** Request Query String  

#####  **Request Parameters** 

| Parameter              | Description                                                  |
| ---------------------- | ------------------------------------------------------------ |
| session.id             | The user session id.                                         |
| ajax=fetchprojectflows | The fixed parameter indicating the fetchProjectFlows action. |
| project                | The project name to be fetched.                              |

 **Response Object** 

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| project   | The project name.                                            |
| projectId | The numerical id of the project.                             |
| flows     | A list of flow ids.                 **Example values:**        [{"flowId": "aaa"}, {"flowId": "bbb"}] |

```sh
# sherlock @ mbp in ~ [12:05:39]
$ curl -k --get --data "session.id=9aafa158-d1c4-45ee-9892-3416130ca525&ajax=fetchprojectflows&project=dwd_algorithm" https://10.0.15.131:8443/manager
{
  "flows" : [ {
    "flowId" : "dwd_prodcut_sale_d"
  } ],
  "project" : "dwd_algorithm",
  "projectId" : 249
}%
```

## 三、获取流的作业 (Fetch Jobs of a Flow)

对于给定的项目和流 ID，此 API 调用会获取属于此流的所有作业。它还返回这些作业的相应图形结构。 

- ​    **Method:** GET  
- ​    **Request URL:** /manager?ajax=fetchflowgraph  
- ​    **Parameter Location:** Request Query String  

#####  **Request Parameters** 

| Parameter           | Description                                                  |
| ------------------- | ------------------------------------------------------------ |
| session.id          | The user session id.                                         |
| ajax=fetchflowgraph | The fixed parameter indicating the fetchProjectFlows action. |
| project             | The project name to be fetched.                              |
| flow                | The project id to be fetched.                                |

#####  **Response Object** 

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| project   | The project name.                                            |
| projectId | The numerical id of the project.                             |
| flow      | The flow id fetched.                                         |
| nodes     | A list of job nodes belonging to this flow.                                   **Structure:** `{  "id": "job.id"  "type": "job.type"  "in": ["job.ids that this job is directly depending upon.  Indirect ancestors is not included in this list"] }                  `                                                 **Example values:** [{"id": "first_job", "type": "java"}, {"id": "second_job", "type": "command", "in":["first_job"]}] |

```sh
# sherlock @ mbp in ~ [12:54:53]
$ curl -k --get --data "session.id=9aafa158-d1c4-45ee-9892-3416130ca525&ajax=fetchflowgraph&project=dwd_algorithm&flow=dwd_prodcut_sale_d" https://10.0.15.131:8443/manager
{
  "nodes" : [ {
    "id" : "dwd_order_label",
    "type" : "command"
  }, {
    "id" : "dwd_prodcut_sale_d",
    "type" : "command"
  } ],
  "project" : "dwd_algorithm",
  "projectId" : 249,
  "flow" : "dwd_prodcut_sale_d"
}%
```

## 四、获取流的执行 (Fetch Executions of a Flow)

