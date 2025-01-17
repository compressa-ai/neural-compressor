# Neural Solution API

Welcome to Neural Solution OaaS API documentation. This API documentation provides a detailed description of all the endpoints available in Neural Solution OaaS API.

## Base URL

The base URL for Neural Solution OaaS API is `{host_ip}:port`

## Endpoints

### GET /

#### Description

This is the welcome interface for Neural Solution OaaS.

#### Usage
```bash
curl -X GET {host_ip}:port/description
```

#### Responses

| Status Code | Description                                       |
| ----------- | ------------------------------------------------- |
| 200         | Welcome to Neural Solution OaaS!                  |


### POST /task/submit

#### Description

Submit a new task to Neural Solution OaaS.

#### Parameters

Refer to [task_request_description.md](./template/task_request_description.md).

#### Usage
```bash
curl -X POST -H "Content-Type: application/json" -d @task_request.json {host_ip}:port/task/submit
```
#### Responses

| Status Code | Description             | Content               |
| ----------- | -----------------------| ---------------------|
| 200         | Submitted successfully.| `status`: "Successfully.", `task_id`: Hashed key, `msg` : "Task submitted successfully"|
| 500         | Submitted failed.       | `status`: "Failed."  |

### GET /task/status/{task_id}

#### Description

Get the status of a submitted task.

#### Parameters

- `task_id` - The hashed key of the submitted task.

#### Usage
```bash
curl -X GET {host_ip}:port/task/status/{task_id}
```

#### Responses

| Status Code | Description      | Content       |
| ----------- | ---------------- | ------------- |
| 200         | The status of task .  | `status`: "running"/"done"/"pending"/"failed"</br> `tuning_info`: tuning information</br> `optimization_result`: optimization time, Accuracy, Duration, result_path|

### GET /task/log/{task_id}

#### Description

Get the log of a submitted task.

#### Usage
```bash
curl -X GET {host_ip}:port/task/log/{task_id}
```

#### Parameters

- `task_id` - The hashed key of the submitted task.

#### Responses

| Status Code | Description | Content      |
| ----------- | ----------- | ------------ |
| 200         | Task log.   | Task log.    |

### WebSocket /task/screen/{task_id}

#### Description

Get real-time log of a submitted task.

#### Parameters

- `task_id` - The hashed key of the submitted task.

#### Responses

| Status Code | Description                              | Content               |
| ----------- | ---------------------------------------- | ---------------------|
| 101         | Get real-time task log.                  | Real-time task log.   |
| 1000        | Normal Closure.                          | Connection was closed successfully.|
| 404         | Task not found.                          | `status`: "Failed."  |

### GET /ping

#### Description

Check the health status of Neural Solution.

#### Usage
```bash
curl -X GET {host_ip}:port/task/log/{task_id}
```

#### Responses

| Status Code | Description  | Content                                |
| ----------- | ------------ | -------------------------------------- |
| 200         | The health status. | `status`: "Healthy", `msg`: "Neural Solution is running." |
| 500         | Ping fail! & error message. | `status`: "Failed.", `msg`: Error message. |


### GET /cluster

#### Description

Get the running status of Neural Solution cluster.

#### Usage
```bash
curl -X GET {host_ip}:port/cluster
```

#### Responses

| Status Code | Description | Content                                    |
| ----------- | ------------| ------------------------------------------ |
| 200         | Cluster information. |  `msg`: "Cluster information." |

### GET /download/{task_id}

#### Description

Download optimized result locally.

#### Usage
```bash
curl -X GET {host_ip}:port/download/{task_id} --output quantized_model.zip
```

#### Responses

| Status Code | Description | Content          |
| ----------- | ----------- | ---------------- |
| 200         | Download optimized model. | zip file |
| 400         | No quantized model when task failed. | `msg`: "Please check URL." |
| 404         | Download optimized model. | `msg`: "Task failed, file not found" |


### GET /description

#### Description

Get user-facing API.

#### Usage
```bash
curl -X GET {host_ip}:port/description
```

#### Responses

| Status Code | Description | Content          |
| ----------- | ----------- | ---------------- |
| 200         | User-facing API. | `msg`: The user-facing API. |