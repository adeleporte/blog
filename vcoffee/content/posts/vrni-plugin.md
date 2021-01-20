---
title: "GET VRNI GRAPHS DIRECTLY FROM A KUBECTL VRNI COMMAND!"
date: 2020-08-24T17:07:15+01:00
draft: false
featured: true
tags: [
    "velocloud",
    "vrni",
    "go",
    "code"
]
---


### *DISCLAIMER: The views and opinions expressed on this blog are our own and may not reflect the views and opinions of our employer*

![kubectl vrni plugin](/vrni-plugin-cli.png)

Hi, today in this blog, I’m going to explain how to develop a small kubectl plugin. As you may know, VMware has a wonderful solution to collect and analyze network flows from many different sources, including Kubernetes: vRealize Network Insight (aka vRNI). My objective is to make this rich information easily and quickly available to a kube admin, thanks to a kubectl command.

What I want to achieve in this part 1 is a kubectl vrni flow IP command that will display some charts representing the flows for this specific IP (source or destination). This information can help to troubleshoot any networking issue. I will handle Kubernetes POD to IP translation in part 2.

Here are the different steps to code:
![Steps](/vrni-plugin-steps.png)

I’m going to develop this code in Go. Because we’ll have to deal with Kubernetes API in Part 2, Go is the best language to start with.

Let’s start by creating a client that we’ll use to speak to vRNI:

```go
package main
 
import (
    "bytes"
    "crypto/tls"
    "encoding/json"
    "errors"
    "fmt"
    "github.com/guptarohit/asciigraph"
    "io/ioutil"
    "net/http"
    "os"
        "net/http"
    "time"
)
 
const (
    // BaseURL ...
    BaseURL = "https://field-demo.vrni.cmbu.local/api/ni"
)
 
type Client struct {
    BaseURL    string
    token      string
    HTTPClient *http.Client
}
 
type errorResponse struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}
 
type successResponse struct {
    Code int         `json:"code"`
    Data interface{} `json:"data"`
}
 
type authResponse struct {
    Token  string `json:"token"`
    Expiry int64  `json:"expiry"`
}
 
// Domain ...
type Domain struct {
    DomainType string `json:"domain_type"`
    Value      string `json:"value,omitempty"`
}
 
// AuthBody ...
type AuthBody struct {
    Username string `json:"username"`
    Password string `json:"password"`
    Domain   Domain `json:"domain"`
}
 
// NewClient ...
func NewClient() *Client {
    return &amp;Client{
        BaseURL: BaseURL,
        token:   "",
        HTTPClient: &amp;http.Client{
            Timeout: time.Minute,
        },
    }
}
 
func (c *Client) sendRequest(req *http.Request) ([]byte, error) {
    req.Header.Set("Content-Type", "application/json; charset=utf-8")
    req.Header.Set("Accept", "application/json; charset=utf-8")
    req.Header.Set("Authorization", fmt.Sprintf("NetworkInsight %s", c.token))
 
    http.DefaultTransport.(*http.Transport).TLSClientConfig = &amp;tls.Config{InsecureSkipVerify: true}
    res, err := c.HTTPClient.Do(req)
    if err != nil {
        return nil, err
    }
 
    defer res.Body.Close()
 
    if res.StatusCode < http.StatusOK || res.StatusCode >= http.StatusBadRequest {
        var errRes errorResponse
        if err = json.NewDecoder(res.Body).Decode(&amp;errRes); err == nil {
            return nil, errors.New(errRes.Message)
        }
 
        return nil, fmt.Errorf("unknown error, status code: %d", res.StatusCode)
    }
 
    body, err := ioutil.ReadAll(res.Body)
    if err != nil {
        return nil, err
    }
 
    return body, nil
}
 
// Auth authenticates vRNI requests
func Auth(c *Client) string {
    var err error
 
    // Build request body
    authBody := &amp;AuthBody{
        Username: "**********",
        Password: "**********",
        Domain: Domain{
            DomainType: "LDAP",       // Otherwise = LOCAL
            Value:      "****", // Otherwise = none
        },
    }
    buf := new(bytes.Buffer)
    json.NewEncoder(buf).Encode(authBody)
 
    // Build the request
    req, err := http.NewRequest("POST", fmt.Sprintf("%s/auth/token", c.BaseURL), buf)
    if err != nil {
        fmt.Println(err.Error())
        return ""
    }
 
    // Send the request
    res, err := c.sendRequest(req)
    if err != nil {
        fmt.Println(err.Error())
        return ""
    }
 
    // Decode the response
    var fullResponse authResponse
    if err = json.Unmarshal(res, &amp;fullResponse); err != nil {
        fmt.Println("error decoding")
        fmt.Println(err)
        return ""
    }
 
    c.token = fullResponse.Token
    return fullResponse.Token
 
}
```

What we’ve done in this piece of code is:

* define a sendRequest function to be able to send the REST API call to vRNI
* call /auth/token method to invoke the creation of a token
* get the token and set it in the client, so all subsequent requests will add a NetworkInsight token header as an authentication mechanism

Then, we need to search the flows for a specific IP. We’re going to use /search method for that part.

```go
// SearchBody ...
type SearchBody struct {
    EntityType string `json:"entity_type"`
    Filter     string `json:"filter"`
}
 
// SearchResult ...
type SearchResult struct {
    EntityID   string `json:"entity_id"`
    EntityType string `json:"entity_type"`
    Time       int64  `json:"time"`
}
 
// SearchResults ...
type SearchResults struct {
    Results    []SearchResult `json:"results"`
    TotalCount int64          `json:"total_count"`
}
 
// Search is searching Flows...
func Search(c *Client, SearchRequest string) SearchResults {
    var err error
 
    // Build request body
    searchBody := &amp;SearchBody{
        EntityType: "Flow",
        Filter:     SearchRequest,
    }
    buf := new(bytes.Buffer)
    // json.NewEncoder(buf).Encode(authBody)
    json.NewEncoder(buf).Encode(searchBody)
 
    // Build the request
    req, err := http.NewRequest("POST", fmt.Sprintf("%s/search", c.BaseURL), buf)
    if err != nil {
        fmt.Println(err.Error())
        return SearchResults{}
    }
 
    // Send the request
    res, err := c.sendRequest(req)
    if err != nil {
        fmt.Println(err.Error())
        return SearchResults{}
    }
 
    // Decode the response
    var fullResponse SearchResults
    if err = json.Unmarshal(res, &amp;fullResponse); err != nil {
        fmt.Println("error decoding")
        fmt.Println(err)
        return SearchResults{}
    }
 
    return fullResponse
 
}
```

Here, we define:

* the search body request
* the expected search result

Then, the function Search takes a SearchRequest string as an argument. This enables me to use this function as a generic one. For example, if I call Search(c, `destination_ip.ip_address = ‘10.0.0.1’ or source_ip.ip_address = ‘10.0.0.1’`), I expect to to fill a SearchResults structure with flows matching source or destination IP 10.0.0.1.

Now, I need a function to get some information for each flow (at least the name):

```go
// EntityResult ...
type EntityResult struct {
    Name string `json:"name"`
}
 
// GetEntity is getting details about the flow...
func GetEntity(c *Client, ID string) EntityResult {
    var err error
 
    // Build the request
    req, err := http.NewRequest("GET", fmt.Sprintf("%s/entities/flows/%s", c.BaseURL, ID), nil)
    if err != nil {
        fmt.Println(err.Error())
        return EntityResult{}
    }
 
    // Send the request
    res, err := c.sendRequest(req)
    if err != nil {
        fmt.Println(err.Error())
        return EntityResult{}
    }
 
    // Decode the response
    var fullResponse EntityResult
    if err = json.Unmarshal(res, &amp;fullResponse); err != nil {
        fmt.Println("error decoding")
        fmt.Println(err)
        return EntityResult{}
    }
 
    return fullResponse
 
}
```

For that, I’m using a /entities/flow method to get some details. For now, I just want to retrieve the name.

Now that I have my flows and their name, I need to get their metrics:

```go
// MetricsResults ...
type MetricsResults struct {
    Metric      string      `json:"metric"`
    DisplayName string      `json:"display_name"`
    Interval    int64       `json:"interval"`
    Unit        string      `json:"unit"`
    PointList   [][]float64 `json:"pointlist"`
    Start       int64       `json:"start"`
    End         int64       `json:"end"`
}
 
// GetMetrics is listing VMs...
func GetMetrics(c *Client, ID string) MetricsResults {
    var err error
    now := time.Now().Unix()
 
    // Build the request
    req, err := http.NewRequest("GET", fmt.Sprintf("%s/metrics?entity_id=%s&amp;start=%d&amp;end=%d&amp;interval=1800&amp;metric=flow.totalBytesRate.rate.average.bitsPerSecond", c.BaseURL, ID, now-86400, now), nil)
    if err != nil {
        fmt.Println(err.Error())
        return MetricsResults{}
    }
 
    // Send the request
    res, err := c.sendRequest(req)
    if err != nil {
        fmt.Println(err.Error())
        return MetricsResults{}
    }
 
    // Decode the response
    var fullResponse MetricsResults
    if err = json.Unmarshal(res, &amp;fullResponse); err != nil {
        fmt.Println("error decoding")
        fmt.Println(err)
        return MetricsResults{}
    }
 
    return fullResponse
}
```

I’m using the /metrics method for that. The interesting part of this call in the parameters. I need to get actual time translated in epochs. I use this function: now := time.Now().Unix(). I use now at the end metric period and now-86400 for the start metric period. 86400s = 24h, so these params will get the last 24-hours of data. The interval is 1800 (30minutes), so we should collect 48 items. The last part is the metric that we want to use: I chose flow.totalBytesRate.rate.average.bitsPerSecond in this post. (If you need to list all accepted metrics, you can use this method: /api/ni/schema/Flow/metrics)

Almost done! We just have to use all these functions in the right order and plot the charts! (For that, I’ll use this package: https://github.com/guptarohit/asciigraph)

Here is the main function:

```go
// InfoColor ...
const (
    InfoColor    = "\033[1;34m%s\033[0m\n"
    NoticeColor  = "\033[1;36m%s\033[0m\n"
    WarningColor = "\033[1;33m%s\033[0m\n"
    ErrorColor   = "\033[1;31m%s\033[0m\n"
    DebugColor   = "\033[0;36m%s\033[0m\n"
)
 
func main() {
 
    arg := os.Args[1:]
 
    if len(arg) <= 1 {
        fmt.Printf(ErrorColor, "Please specify at least one arg")
        return
    }
 
    client := NewClient()
 
    // Authentication
    token := Auth(client)
    if token == "" {
        fmt.Printf(ErrorColor, "Failed to authenticate")
        return
    }
 
 
    res := Search(client, fmt.Sprintf(`destination_ip.ip_address = '%s' or source_ip.ip_address = '%s'`, arg[1], arg[1]))
 
    if len(res.Results) == 0 {
        fmt.Printf(WarningColor, "no found flows")
        return
    }
 
    for i := range res.Results {
 
        metrics := GetMetrics(client, res.Results[i].EntityID)
        flowName := GetEntity(client, res.Results[i].EntityID)
 
        data := []float64{}
        for _, s := range metrics.PointList {
            data = append(data, s[1]/1000000)
        }
 
        // Display Charts
        fmt.Println("")
        fmt.Printf(NoticeColor, flowName.Name)
        fmt.Println("")
        fmt.Printf(InfoColor, "Average MBps (last 24Hours)")
        fmt.Println("")
        graph := asciigraph.Plot(data)
        fmt.Println(graph)
        fmt.Println("")
 
    }
 
}
```

The code is straightforward to understand, the only trick is s[1]/1000000. I decided to display charts in Mbps instead of bps, so I divide each point by 1M.

Making this code a kubectl pluging is more than easy. You just have to build the code with an output binary name starting by kubectl-, and move this executable in your system path.

```bash
adeleporte@adeleporte-a01 adeleporte % go build -o kubectl-vrni
adeleporte@adeleporte-a01 adeleporte % mv kubectl-vrni /usr/local/bin
adeleporte@adeleporte-a01 adeleporte % 
```

You should now be able to plot nice graphs by calling kubectl vrni flows and adding any IP:
![kubectl vrni plugin](/vrni-plugin-cli.png)