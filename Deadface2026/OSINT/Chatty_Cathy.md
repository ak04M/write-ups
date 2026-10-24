# Chatty Cathy — DEADFACE CTF

## Challenge Description

A DEADFACE member operating under the pseudonym **Alice** maintains a travel blog. The objective is to locate the blog through the GhostTown forum and retrieve information exposed by the associated website.

## Solution

### 1. Locate the Travel Blog

I started by searching the GhostTown forum at [ghosttown.deadface.io](https://ghosttown.deadface.io/), using the search term `travel` to find relevant posts.

Under the **OFF-GRID** category, I discovered a post titled **Travel Destinations**, published by `lamia415`. The post included a link to the travel blog hosted at:

```text
https://170.187.156.131:8443/
```

This identified the target web server for further investigation.

### 2. Retrieve the Flag

The challenge description specified that only the website running on port `8443` was relevant. I used `curl` to retrieve the website's `robots.txt` file directly from the command line:

```bash
curl -k https://170.187.156.131:8443/robots.txt
```

The `-k` option allows `curl` to proceed without verifying the server's TLS certificate, which is useful when the target uses a self-signed or otherwise untrusted certificate.

The response contained the flag:

```text
deadface{y0u_m3ddling_k!ds}
```

## Flag

`deadface{y0u_m3ddling_k!ds}`
