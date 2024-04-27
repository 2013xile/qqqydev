---
title: Implement infinite scrolling using IntersectionObserver in React
date: "2024-03-21"
categories:
  - Frontend
  - React
---

> Documenting a method for implementing infinite scrolling using `IntersectionObserver` in React.

# Introduction

In web development, it is a common requirement to automatically load the next page of a list when the scroll bar touches the bottom. There are typically two approaches to achieve this. One is to compare the distance of the scroll bar from the top of the container with the height of the container itself. The other is to use the <a href="https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API" target="_blank">`IntersectionObserver`</a> API.

> The Intersection Observer API provides a way to asynchronously observe changes in the intersection of a target element with an ancestor element or with a top-level document's viewport.

# Solution

With the help of `IntersectionObserver`, we can monitor the last item of the current list. When it appears in the viewport, we load the next page and then move the observer to the last item of the next page.

# Implemention

Here's a simple implementation idea, including some pseudo code. You'll need to modify it according to your needs to make it runnable.

```ts
import React, { useCallback, useEffect, useRef, useState } from "react";

export const useLoadMoreObserver = ({
  loadMore,
}: {
  loadMore: () => void; // callback
}) => {
  const [lastItem, setLastItem] = useState<React.RefObject<any>>(null);
  const observer = useRef<IntersectionObserver | null>(null);
  const observeExposure = useCallback(
    (lastItem: React.RefObject<any>) => {
      if (!lastItem) {
        return;
      }
      observer.current && observer.current.disconnect();
      observer.current = new IntersectionObserver((entries) => {
        entries.forEach((entry) => {
          if (entry.target !== lastItem.current) {
            return;
          }
          if (entry.isIntersecting) {
            observer.current && observer.current.unobserve(lastItem.current);
            loadMore();
          }
        });
      });

      lastItem.current &&
        observer.current &&
        observer.current.observe(lastItem.current);
    },
    [loadMore],
  );

  useEffect(() => {
    observeExposure(lastItem);

    return () => {
      observer.current && observer.current.disconnect();
    };
  }, [lastItem, observeExposure]);

  return {
    lastItem,
    setLastItem,
  };
};
```

```ts
export const List: React.FC = () => {
  const [data, setData] = useState(null);
  const [list, setList] = useState([]);
  const [loading, setLoading] = useState(false);

  const loadMore = useCallback(async () => {
    const meta = data?.meta;
    if (!meta || meta.page >= meta.totalPage) {
      return;
    }
    setLoading(true);
    // request next page
    const res = await axios.request({
      url,
      params: {
        page: meta.page + 1,
        pageSize: meta.pageSize,
      }
    });
    setData(data);
    setLoading(false);
  }, []);
  const { lastItem, setLastItem } = useLoadMoreObserver({ loadMore });

  useEffect(() => {
    if (!data?.data?.length) {
      return;
    }

    const ref = React.createRef<any>();
    setLastItem(ref);
    setList((prev) => prev.concat(data.data));
  }, [data, setLastItem]);

  const items = useMemo(
    () =>
      list.map((item: any, index: number) => ({
        key: item.name,
        label:
          index === roles.length - 1 ? (
            <div ref={lastItem}>
            {item}
            </div>
          ) : item,
      })),
    [list, lastItem],
  );

  return (
    <>
      {roles.length ? (
        <Menu items={items} />
      ) : 'No data'}
    </>
  );
};
```
