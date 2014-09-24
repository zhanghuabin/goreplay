[![Stories in Ready](https://badge.waffle.io/buger/gor.png?label=ready)](https://waffle.io/buger/gor)
[![Build Status](https://travis-ci.org/buger/gor.png?branch=master)](https://travis-ci.org/buger/gor)

## About

Gor is a simple http traffic replication tool written in Go.
Its main goal is to replay traffic from production servers to staging and dev environments.


Now you can test your code on real user sessions in an automated and repeatable fashion.
**No more falling down in production!**

Here is basic workflow: The listener server catches http traffic and sends it to the replay server or saves to file.The replay server forwards traffic to a given address.


![Diagram](http://i.imgur.com/9mqj2SK.png)


## Examples

### Capture traffic from port
```bash
# Run on servers where you want to catch traffic. You can run it on each `web` machine.
sudo gor --input-raw :80 --output-tcp replay.local:28020

# Replay server (replay.local).
gor --input-tcp replay.local:28020 --output-http http://staging.com
```

### Using 1 Gor instance for both listening and replaying
It's recommended to use separate server for replaying traffic, but if you have enough CPU resources you can use single Gor instance.

```
sudo gor --input-raw :80 --output-http "http://staging.com"
```

## Advanced use

### Rate limiting
Both replay and listener support rate limiting. It can be useful if you want
forward only part of production traffic and not overload your staging
environment. You can specify your desired requests per second using the
"|" operator after the server address:

#### Limiting replay
```
# staging.server will not get more than 10 requests per second
gor --input-tcp :28020 --output-http "http://staging.com|10"
```

#### Limiting listener
```
# replay server will not get more than 10 requests per second
# useful for high-load environments
gor --input-raw :80 --output-tcp "replay.local:28020|10"
```

#### Match on regexp of url
```
# only forward requests being sent to the api... domains
gor --input-raw :8080 --output-http staging.com --output-http-url-regexp ^www.
```

#### Filter based on regexp of header
```
# only forward requests with an api version of 1.0x
gor --input-raw :8080 --output-http staging.com --output-http-header-filter api-version:^1\.0\d
```

#### Filter based on hash of header
```
# send 1/32 of all users consistently to staging
gor --input-raw :8080 --output-http staging.com --output-http-header-hash-filter user-id:1/32
```

### Forward to multiple addresses

You can forward traffic to multiple endpoints. Just add multiple --output-* arguments.
```
gor --input-tcp :28020 --output-http "http://staging.com"  --output-http "http://dev.com"
```

#### Splitting traffic
By default it will send same traffic to all outputs, but you have options to equally split it:

```
gor --input-tcp :28020 --output-http "http://staging.com"  --output-http "http://dev.com" --split-output true
```

### Saving requests to file
You can save requests to file, and replay them later:
```
# write to file
gor --input-raw :80 --output-file requests.gor

# read from file
gor --input-file requests.gor --output-http "http://staging.com"
```

**Note:** Replay will preserve the original time differences between requests.

### Injecting headers

Additional headers can be injected/overwritten into requests during replay. This may be useful if you need to identify requests generated by Gor or enable feature flagged functionality in an application:

```
gor --input-raw :80 --output-http "http://staging.server" \
    --output-http-header "User-Agent: Replayed by Gor" \
    --output-http-header "Enable-Feature-X: true"
```

## Filtering HTTP methods

Requests not matching a specified whitelist can be filtered out. For example to strip non-nullipotent requests:

```
gor --input-raw :80 --output-http "http://staging.server" \
    --output-http-method GET \
    --output-http-method OPTIONS
```

### Basic Auth

If your development or staging environment is protected by Basic Authentication then those credentials can be injected in during the replay:

```
gor --input-raw :80 --output-http "http://user:pass@staging .com"
```

Note: This will overwrite any Authorization headers in the original request.

## Additional help

Feel free to ask question directly by email or by creating github issue.

## Latest releases (including binaries)

https://github.com/buger/gor/releases

## Command line reference
`gor -h` output:
```
  -cpuprofile="": write cpu profile to file
  -memprofile="": write memory profile to this file
  
  -input-dummy=[]: Used for testing outputs. Emits 'Get /' request every 1s

  -input-file=[]: Read requests from file: 
    gor --input-file ./requests.gor --output-http staging.com

  -input-raw=[]: Capture traffic from given port (use RAW sockets and require *sudo* access):
    # Capture traffic from 8080 port
    gor --input-raw :8080 --output-http staging.com

  -input-tcp=[]: Used for internal communication between Gor instances. Example: 
    # Receive requests from other Gor instances on 28020 port, and redirect output to staging
    gor --input-tcp :28020 --output-http staging.com

  -output-dummy=[]: Used for testing inputs. Just prints data coming from inputs.

  -output-file=[]: Write incoming requests to file: 
    gor --input-raw :80 --output-file ./requests.gor

  -output-http=[]: Forwards incoming requests to given http address.
    # Redirect all incoming requests to staging.com address 
    gor --input-raw :80 --output-http http://staging.com

  -output-http-header=[]: Inject additional headers to http reqest:
    gor --input-raw :8080 --output-http staging.com --output-http-header 'User-Agent: Gor'

  -output-http-header-filter=[]: A regexp to match a specific header against. Requests with non-matching headers will be dropped:
    gor --input-raw :8080 --output-http staging.com --output-http-header-filter api-version:^v1

  -output-http-header-hash-filter=[]: Takes a fraction of requests, consistently taking or rejecting a request based on the FNV32-1A hash of a specific header. The fraction must have a denominator that is a power of two:
    gor --input-raw :8080 --output-http staging.com --output-http-header-hash-filter user-id:1/4

  -output-http-url-regexp=: A regexp to match requests against. Anything else will be dropped:
    gor --input-raw :8080 --output-http staging.com --output-http-url-regexp ^www.
    
  -output-tcp=[]: Used for internal communication between Gor instances. Example: 
    # Listen for requests on 80 port and forward them to other Gor instance on 28020 port
    gor --input-raw :80 --output-tcp replay.local:28020
    
  -split-output=false: By default each output gets same traffic. If set to `true` it splits traffic equally among all outputs.
  
  -stats=false: If set to `true` it gives out queuing stats for the HTTP output and TCP listener every 5 seconds in the form latest,mean,max,count,count/second.
```

## Building from source
1. Setup standard Go environment http://golang.org/doc/code.html and ensure that $GOPATH environment variable properly set.
2. `go get github.com/buger/gor`.
3. `cd $GOPATH/src/github.com/buger/gor`
4. `go build` to get binary, or `go test` to run tests

## Development
Project contains Docker environment

1. Build container: `make dbuild`
2. Run tests: `make dtest`
3. Bash access to container: `make dbash`. Inside container you have python to run simple web server `python -m SimpleHTTPServer 8080` and `curl` to make http requests. 

## FAQ

### What OS are supported?
For now only Linux based. *BSD (including MacOS is not supported yet, check https://github.com/buger/gor/issues/22 for details)

### Why does the `--input-raw` requires sudo or root access?
Listener works by sniffing traffic from a given port. It's accessible
only by using sudo or root access.

### I'm getting 'too many open files' error
Typical linux shell has a small open files soft limit at 1024. You can easily raise that when you do this before starting your gor replay process:
  
  ulimit -n 64000

More about ulimit: http://blog.thecodingmachine.com/content/solving-too-many-open-files-exception-red5-or-any-other-application

## Tuning

To achieve the top most performance you should tune the source server system limits:

    net.ipv4.tcp_max_tw_buckets = 65536
    net.ipv4.tcp_tw_recycle = 1
    net.ipv4.tcp_tw_reuse = 0
    net.ipv4.tcp_max_syn_backlog = 131072
    net.ipv4.tcp_syn_retries = 3
    net.ipv4.tcp_synack_retries = 3
    net.ipv4.tcp_retries1 = 3
    net.ipv4.tcp_retries2 = 8
    net.ipv4.tcp_rmem = 16384 174760 349520
    net.ipv4.tcp_wmem = 16384 131072 262144
    net.ipv4.tcp_mem = 262144 524288 1048576
    net.ipv4.tcp_max_orphans = 65536
    net.ipv4.tcp_fin_timeout = 10
    net.ipv4.tcp_low_latency = 1
    net.ipv4.tcp_syncookies = 0


## Contributing

1. Fork it
2. Create your feature branch (git checkout -b my-new-feature)
3. Commit your changes (git commit -am 'Added some feature')
4. Push to the branch (git push origin my-new-feature)
5. Create new Pull Request

## Companies using Gor

* [Granify](http://granify.com)
* [GOV.UK](https://www.gov.uk) ([Government Digital Service](http://digital.cabinetoffice.gov.uk/))
* [theguardian.com](http://theguardian.com)
* [TomTom](http://www.tomtom.com/)
* To add your company drop me a line to github.com/buger or leonsbox@gmail.com
