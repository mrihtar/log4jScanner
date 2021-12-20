# log4jScanner

![image](https://user-images.githubusercontent.com/13978578/146378036-eb7ca332-81a1-48a4-ac42-4f320d252ba0.png)


## Goals

This tool provides you with the ability to scan internal (only) subnets for vulnerable log4j web services. 
It will attempt to send a JNDI payload to each discovered web service (via the [methods](#methods_used) outlined below) to a list of common HTTP/S ports. 
For every response it receives, it will log the responding host IP so we can get a list of the vulnerable servers.

If there is a "SUCCESS", this means that some web service has received the request, was vulnerable to the **log4j** exploit and sent a request to our TCP server.

The tool does not send any exploits to the vulnerable hosts, and is designed to be as passive as possible.

## Latest Release

| Platform | Binary   | Checksum |
|----------|----------|----------|
| Windows  |[log4jscanner-windows.zip](https://github.com/proferosec/log4jScanner/releases/download/latest/log4jscanner-windows.zip) | [SHA256](https://github.com/proferosec/log4jScanner/releases/download/latest/windows.sha256.txt) |
| Linux  |[log4jscanner-linux.zip](https://github.com/proferosec/log4jScanner/releases/download/latest/log4jscanner-linux.zip) | [SHA256](https://github.com/proferosec/log4jScanner/releases/download/latest/linux.sha256.txt) |
| MacOS  |[log4jscanner-darwin.zip](https://github.com/proferosec/log4jScanner/releases/download/latest/log4jscanner-darwin.zip) | [SHA256](https://github.com/proferosec/log4jScanner/releases/download/latest/darwin.sha256.txt) |

### ChangeLog

#### version 0.3.1

* moved from a TCP server to a minimal LDAP server, this allows us to have accurate match between request and callback
* added option to scan public IPs `--allow-public-ips`
* added an option to scan a port range `--ports=9000:10000`
* removed the option for slow scanning due to a bug
* we are now using a more comprehensive list of possible headers to trigger the vulnerability
* added a `--timeout` flag to control the timeout for the server shutdown
* added a "hint" to the triggering message to make it easier to determine that these requests came from our tool `Profero-log4jScanner-v0.3.1`
* various bug fixes


## Example

![example](https://github.com/proferosec/log4jScanner/blob/main/movie.gif)

In this example we run the tool against the `192.168.1.59/29` subnet (which contains a vulnerable server). 

The tools does the following:
1. Open a server on the default address (the local IP at port 5555)
2. POssibly, add the flag `--ports=top100` to adjust the scan to include the top 100 ports
3. The tool then tries all ports on each of the IP addresses in the subnet. If a remote server responds at one of the ports, the request is sent to it.
4. If the server is vulnerable, a callback is made to our server (created on step 1) and the IP address of the remote is logged
5. After all IP addresses in the subnet are scanned, the server waits for a default duration of 10s for any lingering connections and closes down
6. The tools displays the summary of the connections made:
   1. Requests sent to responding remote servers (and the status code they responded with)
   2. Any callback address made to our server

## Important Note about Assumptions

* If a callback happened, this means that a vulnerable server exists, the exploit worked and it initiated a callback. 
However.
* A good rule of thumb, if the callback IP address is not in the subnet scanned, the vulnerable server is behind a NAT 
(e.g. a docker container responds with its own IP address, not the host running the docker)
* The network traffic created by the tool might be classified as malicious by security products, or cause a lot of noise for monitoring services
* The server created by the tool assumes that it is open to receive inbound traffic. That means that opening a FW inbound rule on the host running the scan is needed.

## Basic usage

Download the tool for your specific platform (Windows, Linux or Mac), to run the tool, make sure port 5555 on the host is available (or change it via configuration), 
and specify the subnet to scan (it is possible to configure a separate server:port combination using the `--server` flag):

```bash
log4jScanner.exe scan --cidr 192.168.7.0/24
```

This will test the top 10 HTTP\S ports on the hosts in the subnet,  print any vulnerable hosts to the screen, 
and generate a log + summary CSV in the same location as the binary including all the attempts (both vulnerable and non-vulnerable).

In order to identify which hosts are vulnerable just look up the word `SUCCESS` in the log, you can grep the log for the keywork `SUCCESS` to get just the results.
Also, the tool generates a CSV file containing all the results, filter on `vulnerable` to get the vulnerable hosts.

### Additional usage options
You can use the tool to test for the top 100 HTTP\S ports using the `ports top100` flag, insert a single custom port or a range of ports (limited up to 1024 ports) .

```bash
log4jscanner.exe scan --cidr 192.168.7.0/24 --ports=top100
```

```bash
log4jscanner.exe scan --cidr 192.168.7.0/24 --ports=9000
```

```bash
log4jscanner.exe scan --cidr 192.168.7.0/24 --ports=9000:9005
```

it is possible to use a non-default configuration for the callback server
```bash
log4jscanner.exe scan --cidr 192.168.7.0/24 --server=192.168.1.100:5000
```

if you wish to disable the callback server, use `--noserver`

### Available flags

* `--nocolor` provide output without color
* `--ports` either top10 (default), top100 (list of the 100 most common web ports), a custom single port or a range of ports
* `--noserver` only scan, do not use a local callback server
* `--timeout=10` is setting the server shutdown timeout to 10 seconds

### Methods Used

Currently, the tool uses the following areas to try and send an exploit

### Test setup

In order to test your environment, you can use the included docker images to launch vulnerable applications.

Run the docker compose in [here](https://github.com/proferosec/log4jScanner/tree/main/docker):

`docker-compose up -d`

This will provide you with a container vulnerable on port 8080 for HTTP and port 8443 for HTTPS.

Alternatively, you can also run this:
1. Vuln. target: 
   1. `docker run --rm --name vulnerable-app -p 8080:8080 ghcr.io/christophetd/log4shell-vulnerable-app`
2. spin a server for incoming requests
   1. `log4jScanner scanip --cidr DOCKER-SUBNET`
3. send a request to the target, with the server details
   1. sends a request to the vuln. target, with the callback details of the sever
   2. once gets a callback, logs the ip of the calling request


# Contributions

We welcome contributions, please submit a PR or contact us via contact@profero.io
