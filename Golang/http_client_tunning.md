# Go Http Client Library for Production

## Overview

HTTP(hypertext transfer protocol) is a communication protocol that transfer data between client and server.Http requests are very essential to access resources from the same or remote server. In Golang net Http package comes with the defaults settings that we need to adjust according to our requirement for the  high performance.

While working on the Golang projects, I came to know the if the HTTP is not configured properly, it can crash your server.

Problem:1 Default Http Client

The http client not contain the request timeout setting by default.
If you are using http.Get(url) or &Client{} that uses the http.DefaultClient. DefaultClient has not time out setting, it comes with `no timeout`

If the Rest API where you are making request is broken not sending response back that keeps the connection open. As More request came, open connection count will increase, So will increase the server resources utilization, that will result crashing you server when resource limits being reached.

Solution: Don't use default http client, always specify timeout in http.Client according to you use case
```
var httpClient = &http.Client{
  Timeout: time.Second * 10,
}
```

For the Rest API, it is recommended that timeout should not more than 10 second.
If the Requested resource is not responded in 10 second, the Http connection will be canceled with net/http: request canceled (Client.Timeout exceeded ...) error.


Problem:1 Default Http Transport
By default, Golang Http client perform the connection pooling. When the request completes that connection remain open  until the idle connection timeout(default is 90 second),  so if another request came, that uses the same established connection instead of creating new connection, after the idle connection time, connection will return to the pool.

By using the connection pooling, it will keep less connection open and more requests will be serve with low server resources

When we not defined transport in the http.Client, it use the default transport (https://golang.org/src/net/http/transport.go)
By Default Golang, 
```
var DefaultTransport RoundTripper = &Transport{
	...
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	...
}

// DefaultMaxIdleConnsPerHost is the default value of Transport's
// MaxIdleConnsPerHost.
const DefaultMaxIdleConnsPerHost = 2
```

MaxIdleConns is the connection pool size, this is the maximum connection can be open, its default value is 100 connection

There is problem with the default setting `DefaultMaxIdleConnsPerHost` with value of 2 connection, 
DefaultMaxIdleConnsPerHost is the number of connection can be allowed to open per host basic
Means for any particular host out of 100 connection from the  Connection pool only 2 connection will alloted to that host.

With the more request came, it will process only 2 request, other request will wait the connection to communicate wih the host server and go in `TIME_WAIT` state, As more request came, increase the connection in the `TIME_WAIT` state and increase the serve resource utilization, at the limit, server will crash

Solution: Don't use Default Transport and increase MaxIdleConnsPerHost

```
t := http.DefaultTransport.(*http.Transport).Clone()
t.MaxIdleConns = 100
t.MaxConnsPerHost = 100
t.MaxIdleConnsPerHost = 100
	
httpClient = &http.Client{
  Timeout:   10 * time.Second,
  Transport: t,
}
```

By increasing connection per host and the total number of idle connection, this will increase the performance and serve more request with less server resources.

Connection pool size and connection per host count can be increased as per server resources and requirement.



