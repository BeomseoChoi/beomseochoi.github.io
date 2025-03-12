---
layout: post
title: "[Bug Report] Shallow copy"
date: 2025-03-12 20:30:00+0900
description: 
tags: formatting links
disqus_comments: true
categories: Develop
---

```c++
    template <typename T>
    inline void DynamicFixedSizeObjectBuffer<T>::Pull()
    {
        // std::memcpy(&buffer[0], &buffer[front_idx], GetFilledSize());
        std::copy(&buffer[front_idx], &buffer[front_idx] + GetFilledSize(), buffer.begin());

        rear_idx = GetFilledSize();
        front_idx = 0;
    }
    
    template <typename T>
    inline void DynamicFixedSizeObjectBuffer<T>::Expand(const uint32_t expand_object_count)
    {
        if (expand_object_count <= GetBufferSize())
        {
            return;
        }

        std::vector<T> new_buffer(expand_object_count);
        // std::memcpy(&new_buffer[0], &buffer[front_idx], GetFilledSize());
        std::copy(&buffer[front_idx], &buffer[front_idx] + GetFilledSize(), new_buffer.begin());
        buffer = std::move(new_buffer);
        rear_idx = GetFilledSize();
        front_idx = 0;
    }
```

## 요약

1. ```std::copy```는 원소단위 복사, ```std::memcpy```는 비트단위 복사.
2. 복사생성자를 명시하지 않으면 비트단위 복사가 발생.
3. 따라서 ```std::vector```의 원소를 복사할 때 복사생성자를 명시하고 ```std::copy```를 사용.

## 문제

두 가지 문제가 있다.
1. ```GetFilledSize()```는 원소 개수를 의미하기에 ```sizeof(T)```를 곱해줘야 한다.
2. ```std::memcpy```는 비트단위 복사라서 shallow copy가 발생한다.

2번 문제가 먼저 발생했고 그 과정에서 1번 문제를 발견했다. 2번 문제 때문에 double free가 발생했고, 1번 문제를 해결하더라도 동일하게 double free가 일어났다. 즉 double free는 ```std::memcpy```를 사용했기 때문이다.

## 원인

```std::memcpy```는 비트단위 복사를 수행하여 shallow copy가 일어난다. ```Pull``` 함수는 ```front_idx```부터 ```rear_idx```까지의 원소를 맨 앞으로 복사한다. 이때 ```std::memcpy```를 사용하여 shallow copy가 발생하고, 버퍼 내에 다른 원소가 같은 대상을 가리키게 된다. 그 두 원소 중 앞으로 땡겨진 원소를 ```[0]```, 기존 원소를 ```[i]```라고 부르겠다. new_buffer에 ```[0]```이 복사되고, buffer에는 여전히 ```[0]```과 ```[i]```가 존재한다. 이후 new_buffer의 내용을 buffer에 이동시키는데, 이 과정에서 buffer의 원소를 소멸시킨다. new_buffer의 ```[0]```은 deep copy를 수행했기 때문에 buffer의 원소가 소멸되도 문제가 없다. 하지만 buffer의 ```[0]```과 ```[i]```는 shallow copy 때문에 같은 대상을 가리킨다. 그래서 double free가 발생한다.