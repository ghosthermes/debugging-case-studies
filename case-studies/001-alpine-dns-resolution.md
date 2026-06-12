### Case Study 001: The Alpine DNS Ghost

**Date:** March 14, 2024  
**Subject:** Python/Docker/Networking

#### The Symptom
A small scraping service built on `python:3.11-alpine` worked perfectly on my local machine. As soon as I pushed it to the staging cluster, every outgoing request to the internal API returned a `Name or service not known` error. 

#### Initial Theories
1. The API was down. (Verified: It was up.)
2. The staging VPC had a missing routing table entry. (Verified: Other services could reach it.)
3. I misspelled the environment variable. (Verified: I didn't.)

#### The Investigation
I launched a shell inside the running container to manually test the connection.

```bash
# Inside the container
ping internal-api.staging.svc.cluster.local
# Result: ping: bad address 'internal-api.staging.svc.cluster.local'
```

I checked `/etc/resolv.conf`. The search domains were correct. Then I remembered that Alpine Linux uses `musl libc` instead of `glibc`. 

I used `tcpdump` on the host machine to watch the DNS traffic. I noticed something strange: the container was sending DNS queries for the API address, getting a valid response from the cluster DNS, and then... ignoring it. It kept retrying until it timed out.

#### The Root Cause
The issue was the `ndots:5` configuration in Kubernetes combined with how `musl` handles DNS. In Alpine, if a DNS response is too large for a UDP packet, it doesn't always fail over to TCP correctly in certain older kernel environments. More importantly, `musl` does not support the same asynchronous DNS resolution logic as `glibc`. 

The internal API address was long enough that, combined with the search path suffixes, it triggered a specific edge case in how `musl` parses the resolver response.

#### The Solution
I switched the base image from `python:3.11-alpine` to `python:3.11-slim`. The `slim` image uses Debian and `glibc`. 

```dockerfile
# BEFORE
FROM python:3.11-alpine

# AFTER
FROM python:3.11-slim
```

The DNS resolution worked immediately. 

#### The Lesson
Alpine is great for small image sizes, but the `musl` vs `glibc` difference is a massive hidden variable. If a network issue makes no sense on Alpine, try a Debian-based image first. It saves hours of packet sniffing.
