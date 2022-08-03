---
title: 解决Http Code 413(Request Entity Too Large) 错误
author: nhsoft.lsd
date: 2022-08-03
categories: [故障解决,IO,TCP]
tags: [BugFix]
pin: true
---

# Fix 413 Request Entity Too Large

## Ingress:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 10m
```

## Spring Boot:
```
server:
  servlet:
    multipart:
      enabled: true
      max-file-size: 50MB
      max-request-size: 300MB
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
             httpRequestDecoderSpec.maxInitialLineLength(maxInitialLingLength);
            return httpRequestDecoderSpec;
            }
        ));
    }
}
```
