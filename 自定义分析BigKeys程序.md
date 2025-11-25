```
import redis

def scan_bigkeys(host='localhost', port=6379, size_threshold=1024):
    r = redis.Redis(host=host, port=port)
    big_keys = []
    
    # 使用SCAN非阻塞遍历
    for key in r.scan_iter():
        key_type = r.type(key)
        size = 0
        
        try:
            if key_type == b'string':
                size = r.strlen(key)
            elif key_type == b'hash':
                size = r.hlen(key)
            elif key_type == b'list':
                size = r.llen(key)
            elif key_type == b'set':
                size = r.scard(key)
            elif key_type == b'zset':
                size = r.zcard(key)
                
            if size > size_threshold:
                big_keys.append({
                    'key': key.decode(), 
                    'type': key_type.decode(),
                    'size': size
                })
        except redis.RedisError as e:
            print(f"Error processing key {key}: {e}")
    
    return sorted(big_keys, key=lambda x: x['size'], reverse=True)

# 执行扫描
bigkeys = scan_bigkeys(size_threshold=10240)  # 10KB阈值
for item in bigkeys[:10]:  # 输出Top10大键
    print(f"Key: {item['key']}, Type: {item['type']}, Size: {item['size']}")
```