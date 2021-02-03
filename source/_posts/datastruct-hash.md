---
title: 哈希表： 散列查找
date: 2021-02-02 10:55:33
tags:
- 数据结构与算法
categories:
- 数据结构与算法
---

```go
package hash_map

import (
	"github.com/OneOfOne/xxhash"
	"math"
	"sync"
)

const expandFactor = 0.75

type hashTable struct {
	items        []*hashMap
	len          int
	capacity     int
	capacityMask int

	lock sync.Mutex
}

type hashMap struct {
	key   string
	value interface{}
	next  *hashMap
}

func NewHashTable(capacity int) *hashTable {
	defaultCapacity := 1 << 4

	if capacity <= defaultCapacity {
		capacity = defaultCapacity
	} else {
		capacity = 1 << int(math.Ceil(math.Log2(float64(capacity))))
	}

	hTable := new(hashTable)
	hMap := make([]*hashMap, capacity, capacity)
	hTable.items = hMap
	hTable.capacity = capacity
	hTable.capacityMask = capacity - 1

	return hTable
}

func (c *hashTable) hashIndex(key string) int {
	hash := xxHash([]byte(key))

	return int(hash & uint64(c.capacityMask))
}

func (c *hashTable) Add(key string, value interface{}) {
	c.lock.Lock()
	defer c.lock.Unlock()

	index := c.hashIndex(key)
	element := c.items[index]
	if element == nil {
		c.items[index] = &hashMap{
			key:   key,
			value: value,
		}
	} else {
		var lastElement *hashMap
		for element != nil {
			if element.key == key {
				element.value = value
				return
			}

			lastElement = element
			element = element.next
		}

		lastElement.next = &hashMap{
			key:   key,
			value: value,
		}

		newLen := c.len + 1
		if float64(newLen)/float64(c.capacity) >= expandFactor {
			newHashMap := new(hashTable)
			newHashMap.items = make([]*hashMap, 2*c.capacity, 2*c.capacity)
			newHashMap.capacity = 2 * c.capacity
			newHashMap.capacityMask = 2*c.capacity - 1
			for _, item := range c.items {
				for item != nil {
					newHashMap.Add(item.key, item.value)
					item = item.next
				}
			}

			c.items = newHashMap.items
			c.capacity = newHashMap.capacity
			c.capacityMask = newHashMap.capacityMask
		}

		c.len = newLen
	}
}

func (c *hashTable) Get(key string) interface{} {
	c.lock.Lock()
	defer c.lock.Unlock()

	index := c.hashIndex(key)
	element := c.items[index]

	for element != nil {
		if element.key == key {
			return element.value
		}

		element = element.next
	}

	return nil
}

func (c *hashTable) Delete(key string) {
	c.lock.Lock()
	defer c.lock.Unlock()

	index := c.hashIndex(key)
	element := c.items[index]
	if element == nil {
		return
	}

	if element.key == key {
		c.items[index] = element.next
		c.len = c.len - 1
		return
	}

	nextElement := element.next
	for nextElement != nil {
		if nextElement.key == key {
			element.next = nextElement.next
			c.len = c.len - 1
			return
		}

		element = nextElement
		nextElement = nextElement.next
	}
}

func (c *hashTable) Range() map[string]interface{} {
	c.lock.Lock()
	defer c.lock.Unlock()

	hashMaps := make(map[string]interface{}, c.len)

	for _, item := range c.items {
		for item != nil {
			hashMaps[item.key] = item.value
			item = item.next
		}
	}

	return hashMaps
}

func xxHash(key []byte) uint64 {
	h := xxhash.New64()
	h.Write(key)
	return h.Sum64()
}
```



### 参考文档

- https://goa.lenggirl.com/algorithm/search/hash_find.html