A flexible implementation of Django's [`cache_page`](https://docs.djangoproject.com/en/dev/topics/cache/#the-per-view-cache) decorator, supports building custom keys, cache groups, O(1) cache invalidation on redis, and more.

# Description

This package provides a customizable `cache_page` decorator that allows you to define your custom key generation functions with a few utilities and invalidate large cache groups on redis at no cost.

### The story behind this package:
we used django's default `cache_page` decorator for around a year, it was really easy and quick to set up and it served its purpose, but it was very limiting, it had no way of building custom keys for cached views and that was making it quite impossible to invalidate view caches properly, data inconsistencies appeared everywhere, and we had to build a customizable solution for caching. 


# Installation
 
```bash
pip install django-custom-cache-page
```


# Features:

The customizable cache_page supports the following:
- accepts a function to generate a key based on the request.
- has an option to have versioned groups in the cache key
    - this is really useful with **redis**, deleting a large number of caches (without storing all the keys!) on redis has a performance of O(n) and it blocks any other operation because redis is single-threaded. meaning that it will take a longer time as the size of your cache DBs grow, it's impossible to do this operation frequently at scale without choking redis. our solution to achieve O(1) invalidation is to have versioned groups in the cache key, where the group version is stored in redis separately, all you need to invalidate a group of caches is to increment the group version and all the old caches will become invalid and will expire gradually. (be careful, this won't help with keys that don't expire.)
- supports disabling cache for some requests by setting `do_not_cache` attribute on the request to True.

There are two default utilities to generate cache keys:
- generate_cache_key: similar to Django's default
- generate_query_params_cache_key: generate a key using query parameters only

but you can create your own key generation function and pass it to the decorator.


# Example

views.py:

```python
from django.http import HttpResponse

from custom_cache_page.cache import cache_page
from custom_cache_page.utils import generate_query_params_cache_key

@cache_page(60 * 60, generate_query_params_cache_key)
def my_view(request):
    return HttpResponse("okay")
```

using groups:
```python
from django.http import HttpResponse
from custom_cache_page.cache import cache_page

@cache_page(
        timeout=60 * 60,
        key_func=lambda r: r.path,
        versioned=False,
        group_func=lambda r: 'cached_views',
        prefix='prefix'
)
def my_view(request):
    return HttpResponse("okay")
```
to invalidate a group:
```python
from custom_cache_page.utils import invalidate_group_caches

invalidate_group_caches('cached_views')
```

## Hashed keys:

All keys generated by this package are hashed using md5 for performance reasons. if you wanted to delete the keys manually use the hashing utility first:
```python
from custom_cache_page.utils import hash_key
key = 'prefix:cached_views:0:/bo'
hashed_key =  hash_key(key)
```

---
## Running Tests
 
```bash
pytest
```


## Development installation

```bash
git clone https://github.com/zidsa/django-custom-cache-page.git
cd django-custom-cache-page
pip install --editable .
```
