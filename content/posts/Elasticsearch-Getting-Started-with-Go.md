---
author:
  name: ruifeng
  link: https://blog.luckypeak.xyz
  email: me@luckypeak.xyz
  avatar: /safari-pinned-tab.svg
title: 【译】Elasticsearch Go 入门
subtitle: ""
date: 2023-05-11
lastmod: 2023-05-11
categories: []
tags:
  - Go
  - Elasticsearch
draft: false
---

[原文](https://medium.com/@juliardifh49/elasticsearch-getting-started-with-go-520035d5b45f)

大家好，这是我的第一篇文章，分享了如何在 Go 中开始使用 Elasticsearch。它介绍了构建可以执行基本 CRUD 操作的简单 HTTP 服务器。看完这篇文章，我希望你能用 Elasticsearch x Go 开发很酷的项目。所以，事不宜迟，让我们开始行动吧！

## 什么是 Elasticsearch ？

Elasticsearch 是一个开源服务，用于构建自己的搜索引擎。它具有可扩展性和强大的功能，可用于处理高搜索流量。许多大型科技公司（如 Netflix，Microsoft 等）已经在许多用例中实施了 Elasticsearch。因此，如果你想建立一个强大的搜索引擎，你必须尝试这个。

## 通过 Docker 运行

有很多方法可以运行 Elasticsearch 服务器。使用 docker 是最简单的方法。通过删除其容器也很容易卸载。如果你还没有安装 docker，请不要担心。你可以按照以下 docker 文档中的安装步骤进行操作：

- [Install docker on Linux](https://docs.docker.com/engine/install/ubuntu/)
- [Install docker on Mac](https://docs.docker.com/desktop/install/mac-install/)
- [Install docker on Windows](https://docs.docker.com/desktop/install/windows-install/)

安装完成后，你可以键入以下命令来运行 Elasticsearch 服务器：

```shell
docker run -d --name es01 -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -it docker.elastic.co/elasticsearch/elasticsearch:7.14.0
```

在浏览器中打开此网址 http://localhost:9200。 如果你收到标语 “You Know, for Search” 的响应，那么恭喜你在本地计算机上成功运行了 Elasticsearch 服务器！

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/69f6970aaece9303cfd75a2221b349eb.png)

你可以通过运行下面的 docker 命令来关闭此服务器：

```shell
docker stop es01
```

## 设置客户端

我们需要准备一个 Go HTTP 客户端来与 Elasticsearch 进行通信。现在让我们使用健康检查函数启动客户端对象：

```go client.go
type Client struct {
	baseURL string
}

func NewClient(host string) *Client {
	return &Client{
		baseURL: host,
	}
}

func (c *Client) CheckHealth() error {
	resp, err := http.Get(c.baseURL)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	responseBody, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return fmt.Errorf("failed check ElasticSearch health: %v", err)
	}

	log.Println("debug health check response: ", string(responseBody))

	return nil
}
```

在我们的 Go 客户端准备就绪后，我们可以继续创建我们的第一个索引。Elasticsearch 上的索引（index）等效于 SQL 数据库上的模式（schema）。但是，在此之前，我们必须首先设计数据模型（data model）。让我们使用员工作为我们的模型：

```go employee.go
package main

type Employee struct {
	Id      int     `json:"id"`
	Name    string  `json:"name"`
	Address string  `json:"address"`
	Salary  float64 `json:"salary"`
}
```

现在，我们已准备好创建新索引。让我们通过 Elasticsearch API 访问 PUT /employee 来创建索引：

```shell
curl --location --request PUT 'http://localhost:9200/employee' \
--header 'Content-Type: application/json' \
--data-raw '{
  "mappings": {
    "properties": {
      "id": {
        "type": "integer"
      },
      "name": {
        "type": "text"
      },
      "address": {
        "type": "text"
      },
      "salary": {
        "type": "float"
      }
    }
  }
}'
```

此外，要检查员工索引是否存在，我们可以访问 http://localhost:9200/\_cat/indices。

```
green  open .geoip_databases 4B9nDUHMQpicsti15yI-jQ 1 0 43 0 41mb 41mb
yellow open employee         -x37Gq3UQzeEgBjfXVKIgw 1 1  0 0 208b 208b
```

最后，我们可以在客户端代码结构中添加 CreateIndex 函数：

```go client.go
func (c *Client) CreateIndex() error {
	body := `
	{
		"mappings": {
			"properties": {
				"id": {
					"type": "integer"
				},
				"name": {
					"type": "string"
				},
				"address": {
					"type": "string"
				},
				"salary": {
					"type": "float"
				}
			}
		}
	}
	`

	req, err := http.NewRequest("PUT", c.baseURL+"/employee", strings.NewReader(body))
	if err != nil {
		return fmt.Errorf("failed to make a create index request: %v", err)
	}

	httpClient := http.Client{}
	req.Header.Add("Content-Type", "application/json")
	response, err := httpClient.Do(req)
	if err != nil {
		return fmt.Errorf("failed to make a http call to create an index: %v", err)
	}
	defer response.Body.Close()

	responseBody, err := ioutil.ReadAll(response.Body)
	if err != nil {
		return fmt.Errorf("failed to read create index response: %v", err)
	}

	log.Println("debug create index response: ", string(responseBody))

	return nil
}
```

## 插入数据

创建员工索引后，我们可以开始插入一些数据。

API PUT employee/\_doc/1 的用法如下：

```shell
curl --location --request PUT 'http://localhost:9200/employee/_doc/1' \
--header 'Content-Type: application/json' \
--data-raw '{
  "id": 1,
  "name": "xiao",
  "address": "tai",
  "salary": 10000
}'
```

为了确保数据存在，我们可以访问 GET employee/\_doc/1 从 Elasticsearch 检索数据。

```
{"_index":"employee","_type":"_doc","_id":"1","_version":1,"_seq_no":0,"_primary_term":1,"found":true,"_source":{
"id": 1,
"name": "xiao",
"address": "tai",
"salary": 10000
}}
```

为了更真实，我们需要通过重复调用带有随机 id 的 `PUT employee/_doc/{id}` API 来插入几个新员工。让我们通过 `InsertData` 包装插入 API，通过 `SeedingData` 生成随机数据来简化这个过程。

```go client.go
func (c *Client) InsertData(e *Employee) error {
	body, _ := json.Marshal(e)

	id := strconv.Itoa(e.Id)
	req, err := http.NewRequest("PUT", c.baseURL+"/employee/_doc/"+id, bytes.NewBuffer(body))
	if err != nil {
		return fmt.Errorf("failed to make a insert data request: %v", err)
	}

	httpClient := http.Client{}
	req.Header.Add("Content-Type", "application/json")
	response, err := httpClient.Do(req)
	if err != nil {
		return fmt.Errorf("failed to make a http call to insert data: %v", err)
	}
	defer response.Body.Close()

	responseBody, err := ioutil.ReadAll(response.Body)
	if err != nil {
		return fmt.Errorf("failed to read insert data response: %v", err)
	}

	log.Println("debug insert data response: ", string(responseBody))

	return nil
}

func (c *Client) SeedingData(idStart, n int) error {
	for i := idStart; i < n; i++ {
		if err := c.InsertData(&Employee{
			Id:      i,
			Name:    "name_" + strconv.Itoa(i),
			Address: "address_" + strconv.Itoa(i),
			Salary:  float64(i * 100),
		}); err != nil {
			return fmt.Errorf("failed seeding data with id %d: %v", i, err)
		}
	}
	return nil
}
```

## 搜索数据

搜索操作是 Elasticsearch 的强项。我们可以像搜索引擎一样执行复杂的搜索。但是现在，我们只使用 `match` 做一个简单的搜索：

```shell
curl --location --request GET 'http://localhost:9200/employee/_search' \
--header 'Content-Type: application/json' \
--data-raw '{
  "query": {
    "match": {
      "name": "xiao"
    }
  }
}'
```

该查询的意思是 从给定的关键字名称中找到匹配的员工。它将输入拆分为单词，然后将它们与 Elasticsearch 中的数据进行匹配。

```shell
curl --location --request GET 'http://localhost:9200/employee/_search' \
--header 'Content-Type: application/json' \
--data-raw '{
  "query": {
    "match": {
      "name": "xiao"
    }
  }
}'
{
  "took":21,
  "timed_out":false,
  "_shards":{
    "total":1,
    "successful":1,
    "skipped":0,
    "failed":0
  },
  "hits":{
    "total":{
      "value":1,"relation":"eq"
      },
    "max_score":0.2876821,
    "hits":[
      {
        "_index":"employee",
        "_type":"_doc",
        "_id":"1",
        "_score":0.2876821,
        "_source":{
          "id": 1,
          "name": "xiao",
          "address": "tai",
          "salary": 10000
        }
      }
    ]
  }
}
```

然后，我们在 SearchData 函数中实现匹配查询：

```go client.go
type SearchHits struct {
	Hits struct {
		Hits []*struct {
			Source *Employee `json:"_source"`
		} `json:"hits"`
	} `json:"hits"`
}

func (c *Client) SearchData(keyword string) ([]*Employee, error) {
	query := fmt.Sprintf(`
	{
		"query": {
			"match": {
				"name": "%s"
			}
		}
	}
	`, keyword)

	req, err := http.NewRequest("GET", c.baseURL+"/employee/_search", strings.NewReader(query))
	if err != nil {
		return nil, fmt.Errorf("failed to make a search request: %v", err)
	}

	client := http.Client{}
	req.Header.Add("Content-Type", "application/json")
	resp, err := client.Do(req)
	if err != nil {
		return nil, fmt.Errorf("failed to make a http call to search data: %v", err)
	}
	defer resp.Body.Close()

	responseBody, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return nil, fmt.Errorf("failed to read search data response: %v", err)
	}

	log.Println("debug search data response: ", string(responseBody))

	var searchHits SearchHits
	if err := json.Unmarshal(responseBody, &searchHits); err != nil {
		return nil, fmt.Errorf("failed to unmarshal search data response: %v", err)
	}

	var employees []*Employee
	for _, hit := range searchHits.Hits.Hits {
		employees = append(employees, hit.Source)
	}

	return employees, nil
}
```

此外，搜索操作仅支持文本类型，因为文本类型是具有独特全文处理技术的特定类型。

## 更新数据

我们可以使用 POST `employee/_update/{id}` 来更新员工的数据。我们使用此端点进行部分更新。因此，我们只能写入要编辑的数据。在下面的代码中，我们只更新员工姓名：

```shell
curl --location --request POST 'http://localhost:9200/employee/_update/1' \
--header 'Content-Type: application/json' \
--data-raw '{
  "doc": {
    "name": "xiao lan"
  }
}'
```

为了确保数据成功更新，我们可以使用 GET `employee/_doc/{id}` 来检索最新数据。

```console
{
  "_index":"employee",
  "_type":"_doc",
  "_id":"1",
  "_version":2,
  "_seq_no":1,
  "_primary_term":1,
  "found":true,
  "_source":{
    "id": 1,
    "name": "xiao lan",
    "address": "tai",
    "salary": 10000
  }
}
```

然后，让我们将其包装到函数 UpdateData 下的客户端代码中：

```go client.go
func (c *Client) UpdateData(e *Employee) error {
	body, _ := json.Marshal(map[string]*Employee{
		"doc": e,
	})

	id := strconv.Itoa(e.Id)
	req, err := http.NewRequest("POST", c.baseURL+"/employee/_update/"+id, bytes.NewBuffer(body))
	if err != nil {
		return fmt.Errorf("failed to make a update data request: %v", err)
	}

	httpClient := http.Client{}
	req.Header.Add("Content-Type", "application/json")
	response, err := httpClient.Do(req)
	if err != nil {
		return fmt.Errorf("failed to make a http call to update data: %v", err)
	}
	defer response.Body.Close()

	responseBody, err := ioutil.ReadAll(response.Body)
	if err != nil {
		return fmt.Errorf("failed to read update data response: %v", err)
	}

	log.Println("debug update data response: ", string(responseBody))

	return nil
}
```

## 删除数据

如果我们想删除员工数据，我们可以调用 DELETE `employee/_doc/{id}`。

```shell
curl --location --request DELETE 'http://localhost:9200/employee/_doc/1'
```

然后，将其封装到我们的客户端代码的 `DeleteData` 函数下：

```go client.go
func (c *Client) DeleteData(id int) error {
	req, err := http.NewRequest("DELETE", c.baseURL+"/employee/_doc/"+strconv.Itoa(id), nil)
	if err != nil {
		return fmt.Errorf("failed to make a delete data request: %v", err)
	}

	httpClient := http.Client{}
	response, err := httpClient.Do(req)
	if err != nil {
		return fmt.Errorf("failed to make a http call to delete data: %v", err)
	}
	defer response.Body.Close()

	responseBody, err := ioutil.ReadAll(response.Body)
	if err != nil {
		return fmt.Errorf("failed to read delete data response: %v", err)
	}

	log.Println("debug delete data response: ", string(responseBody))

	return nil
}
```

## CRUD Server

最后，让我们将所有 CRUD 操作连接到我们的 HTTP 服务器：

```go server.go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"strconv"
)

// 外部用户通过访问 CRUD server 来访问 Elasticsearch 服务
// server 对外提供 RESTful API
// 对内调用 client 访问 Elasticsearch 服务

type Server struct {
	client *Client
}

func NewServer(clientURI string) *Server {
	client := NewClient(clientURI)
	if err := client.CheckHealth(); err != nil {
		log.Fatal("failed to check health: ", err)
	}
	if err := client.CreateIndex(); err != nil {
		log.Fatal("failed to create index: ", err)
	}

	return &Server{
		client: client,
	}
}

func (s *Server) Start(port string) {
	http.HandleFunc("/insert", s.InsertDataHandler)
	http.HandleFunc("/search", s.SearchDataHandler)
	http.HandleFunc("/update", s.UpdateDataHandler)
	http.HandleFunc("/delete", s.DeleteDataHandler)
	http.HandleFunc("/health", s.CheckHealthHandler)
	log.Println("listening server on port:", port)
	log.Fatal(http.ListenAndServe(port, nil))
}

func (s *Server) InsertDataHandler(w http.ResponseWriter, r *http.Request) {
	var employee *Employee
	json.NewDecoder(r.Body).Decode(&employee)
	if err := s.client.InsertData(employee); err != nil {
		writeResponseInternalError(w, err)
		return
	}
	writeResponseOK(w, employee)
}

func (s *Server) SearchDataHandler(w http.ResponseWriter, r *http.Request) {
	keyword := r.FormValue("keyword")
	employees, err := s.client.SearchData(keyword)
	if err != nil {
		writeResponseInternalError(w, err)
		return
	}
	writeResponseOK(w, employees)
}

func (s *Server) UpdateDataHandler(w http.ResponseWriter, r *http.Request) {
	var employee *Employee
	json.NewDecoder(r.Body).Decode(&employee)
	if err := s.client.UpdateData(employee); err != nil {
		writeResponseInternalError(w, err)
		return
	}
	writeResponseOK(w, employee)
}

func (s *Server) DeleteDataHandler(w http.ResponseWriter, r *http.Request) {
	id, _ := strconv.Atoi(r.FormValue("id"))
	if err := s.client.DeleteData(id); err != nil {
		writeResponseInternalError(w, err)
		return
	}
	writeResponseOK(w, Employee{Id: id})
}

func (s *Server) CheckHealthHandler(w http.ResponseWriter, r *http.Request) {
	if err := s.client.CheckHealth(); err != nil {
		writeResponseInternalError(w, err)
		return
	}
	writeResponseOK(w, map[string]string{
		"status": "OK",
	})
}

func writeResponseInternalError(w http.ResponseWriter, err error) {
	w.Header().Add("Content-Type", "application/json")
	w.WriteHeader(http.StatusInternalServerError)
	writeResponse(w, map[string]interface{}{
		"error": err,
	})
}

func writeResponse(w http.ResponseWriter, response interface{}) {
	json.NewEncoder(w).Encode(response)
}

func writeResponseOK(w http.ResponseWriter, response interface{}) {
	w.Header().Add("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	writeResponse(w, response)
}
```

```go main.go
package main

func main() {
	server := NewServer("http://localhost:9200")
	server.Start(":8080")
}
```