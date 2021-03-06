summary: Discover how you can extend your data source plugin with a backend.
id: build-a-backend-plugin
categories: Plugins
tags: intermediate
status: Published
authors: Grafana Labs
Feedback Link: https://github.com/grafana/tutorials/issues/new

# Build a backend plugin

## Overview

Duration: 1

In this codelab, you'll learn how to build a backend plugin to support your data source plugin.

### Prerequisites

- Completed [Build data source plugins](/build-a-data-source-plugin)

### What you'll need

- Grafana version 6.4+
- NodeJS
- yarn
- Go

## Create a backend plugin

In the previous part of this guide, we looked at how to get started with writing data source plugins for Grafana. For many data sources, integrating a custom data source can be done completely in the Grafana browser client. For others, you might want the plugin to be able to continue running even after closing your browser window, such as alerting, or authentication.

Luckily, Grafana has support for _backend plugins_, which lets your data source plugin communicate with a process running on the server.

Last time, we started writing a data source plugin that would read CSV files. Let's see how a backend plugin lets you read a file on the server and return the data back to the client.

- Create another directory `backend` in your project root directory, containing a `main.go` file.

**main.go**

```go
package main

import (
    "context"
    "log"
    "os"

    sdk "github.com/grafana/grafana-plugin-sdk-go"
)

const pluginID = "myorg-custom-datasource"

type MyDataSource struct {
    logger *log.Logger
}

func (d *MyDataSource) Query(ctx context.Context, tr sdk.TimeRange, ds sdk.DataSourceInfo, queries []sdk.Query) ([]sdk.QueryResult, error) {
    return []sdk.QueryResult{}, nil
}

func main() {
    logger := log.New(os.Stderr, "", 0)

    srv := sdk.NewServer()

    srv.HandleDataSource(pluginID, &MyDataSource{
        logger: logger,
    })

    if err := srv.Serve(); err != nil {
        logger.Fatal(err)
    }
}
```

Positive
: The Grafana backend plugin system uses the [go-plugin](https://github.com/hashicorp/go-plugin) library from [Hashicorp](https://www.hashicorp.com/), and while communication happens over gRPC, here we'll use a Go client library that wraps the boilerplate for you.

- Build the plugin binary.

In order for Grafana to discover our plugin, we have to build a binary with the following suffixes, depending on the machine you're using to run Grafana:

```
_linux_amd64
_darwin_amd64
_windows_amd64.exe
```

I'm running Grafana on my MacBook, so I'll go ahead and build a Darwin binary:

```
go build -o ./dist/csv-datasource_darwin_amd64 ./backend
```

The binary needs to be bundled into `./dist` directory together with the frontend assets.

- Add a field `executable` to the `plugin.json` to make Grafana aware of our backend plugin.

```json
{
  "executable": "my-datasource"
}
```

The value should be the name of the binary, with the suffix removed.

- Restart Grafana and verify that your plugin is running:

```
ps aux | grep csv-datasource
```

## Add a backend handler

- Add two structs to represent the query and the options:

**main.go**

```go
type Query struct {
    RefID  string `json:"refId"`
    Values string `json:"values"`
}

type Options struct {
    Path string `json:"path"`
}
```

- Implement the `Query` method.

```go

func (d *MyDataSource) Query(ctx context.Context, tr sdk.TimeRange, ds sdk.DataSourceInfo, queries []sdk.Query) ([]sdk.QueryResult, error) {
    var opts Options
    if err := json.Unmarshal(ds.JsonData, &opts); err != nil {
        return nil, err
    }

    var res []sdk.QueryResult

    for _, q := range queries {
        var query Query
        if err := json.Unmarshal(q.ModelJson, &query); err != nil {
            return nil, err
        }

        var values []int
        for _, val := range strings.Split(query.values, ",") {
            num, _ := strconv.Atoi(val)
            values = append(values, num)
        }

        res = append(res, sdk.QueryResult{
            RefID: query.RefID,
            DataFrames: []sdk.DataFrame{dataframe.New("", dataframe.Labels{},
                dataframe.NewField("timestamp", dataframe.FieldTypeTime, []time.Time{}),
                dataframe.NewField("value", dataframe.FieldTypeNumber, values),
            )},
        })
    }

    return res, nil
}
```

## Use the backend plugin from the client

Let's make the `testDatasource` call our backend to make sure it's responding correctly.

- Install `@grafana/runtime`, by running `yarn add --dev @grafana/runtime`.
- In `DataSource.ts`, import the `getBackendSrv`. `getBackendSrv` is a function that returns runtime information about the Grafana backend.

**DataSource.ts**

```ts
import { getBackendSrv } from "@grafana/runtime";
```

- Define the backend request:

```ts
interface Request {
  queries: any[];
  from?: string;
  to?: string;
}
```

- In `DataSource.ts`, add the following code:

```ts
testDatasource() {
  const requestData: Request = {
    from: '5m',
    to: 'now',
    queries: [
      {
        datasourceId: this.id,
      },
    ],
  };

  return getBackendSrv()
    .post('/api/tsdb/query', requestData)
    .then((response: any) => {
      if (response.status === 200) {
        return { status: 'success', message: 'Data source is working', title: 'Success' };
      } else {
        return { status: 'failed', message: 'Data source is not working', title: 'Error' };
      }
    })
    .catch((error: any) => {
      return { status: 'failed', message: 'Data source is not working', title: 'Error' };
    });
}
```

- Confirm that the client is able to call our backend plugin by hitting **Save & Test** on your data source. It should give you a green message saying _Data source is working_.
