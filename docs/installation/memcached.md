[TOC]

---

Install Memcached:
```sh
root@cloudserver:~# apt-get update && apt-get install memcached
```

* Stop seafile service
* Stop seahub service

Append some lines to `/opt/Seafile/Server/conf/seahub_settings.py` to enable memcached.

```
...
        'HOST': '127.0.0.1',
        'PORT': '3306'
    }
}

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '127.0.0.1:11211',
    }
}
```