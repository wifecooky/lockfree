# Lockfree

## 1. Introduction
### 1.1. Design Purpose
A high-performance lock-free queue implementation in Go that outperforms native channels by 3-8 times. Achieves 20 million ops/sec on 8-core cloud servers.

### 1.2. Channel Fundamentals
#### 1.2.2. Channel Write Flow
![write.jpg](images/write.jpg)

#### 1.2.3. Channel Read Flow
![read.jpg](images/read.jpg)

## 2. Core Principles
### 2.2. Optimizations
#### 4) Generics Acceleration
Using compile-time type determination through generics to reduce runtime type conversion overhead.

## 3. Usage
### 3.1. Import
```go
go get github.com/bruceshao/lockfree
```

## 4. Performance
### 4.2. Test Environment
- CPU: 8-core Intel Xeon
- Memory: 8GB
- OS: CentOS 7.2
- Go version: 1.18+