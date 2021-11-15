## 1. 并发更新map的最新值
```
    Long oldValue;
    do {
        oldValue = map.putIfAbsent(key, newValue);
    } while (oldValue != null && oldValue < newValue
            && !map.replace(key, oldValue, newValue));
```