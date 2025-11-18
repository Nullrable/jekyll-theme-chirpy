---
title: 解决Http Code 414 (Request—URI Too Large) 错误
author: nhsoft.lsd
date: 2022-08-03
categories: [Java,故障解决]
tags: [BugFix]
pin: false
---

# Fix 414 Too Large Request-URI

## Ingress:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
annotations:
nginx.ingress.kubernetes.io/server-snippet: |
client_header_buffer_size 512k;
large_client_header_buffers 4 512k;
```

## Spring Boot:
```
server:
max-http-header-size: 51200
```

## Spring Cloud Gateway:
```
@Component
public class NettyConfig implements WebServerFactoryCustomizer<NettyReactiveWebServerFactory> {

    /**
     * 10m
     */
    @Value("${server.max-initial-line-length:10485760}")
    private int maxInitialLingLength;

    @Override
    public void customize(final NettyReactiveWebServerFactory factory) {
        factory.addServerCustomizers(httpServer -> httpServer.httpRequestDecoder(httpRequestDecoderSpec -> {
            httpRequestDecoderSpec.maxHeaderSize(maxInitialLingLength) ;
            return httpRequestDecoderSpec;
            }
        ));
    }
}
```
