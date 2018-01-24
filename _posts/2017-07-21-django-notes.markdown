---
published: true
layout: post
---
## A few notes about django so far
### Does django handle request by thread or process?
The correct answer is that django is a WSGI application, which is not a server itself but only interfaces with a server. How a process is servered (by process or by thread) is entirely based on the WSGI server django sits on top of (e.g. uWSGI, Gunicorn, mod_wsgi)
### Generator is kind of cool
...but it can not be used in a django template. In the template, the generator is always turned into a list before being iterated over. [reference](https://github.com/django/django/commit/6b730e1e92)
- consider a DFS travesal piece of code in python

```python
def traversal_with_path(node, path):
    path.append(node)

    children = list(get_children_nodes(node.id))

    if not children:
        yield path #important

    for child in get_children_nodes(node.id):
        yield from traversal_with_path(child, path)

    path.pop()

```
if now we are traversing through this generator like this
```python
all_paths = []
for path in traversal_with_path(node, []):
	all_paths.append(path)
```
apparently, the `all_paths` variable will contain all empty lists! 

Take a look at the line marked `important` above. Each time a path is yield, it refers to the same list. The list is being changed all the time, at each and every level of the tree. When we do `all_paths.append(path)` we are appending the same object reference over and over again, which got empty at the end of the DFS algorithm.

To fix this, just need to change the `#important` line into `yield path[:]` to return a copy of the path instead of returning the reference object.

### Dummy memory django cache from `django.core.cache.caches`
...has  really small default cache size (300 entries). When the number of entries exceeds this number, the old entries are removed. Better to just use memcached right away. In development environment, starting off with memcached is easy

```bash
docker run -d memcached
```
Get the ip of the docker box
```bash
docker inspect unruffled_jepsen | grep IPAddress | cut -d '"' -f 4
```
Then config django to use memcached
```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '172.17.0.2:11211',
    },
}
```

### In python, `mock` is not cleared easily as in java
Python mock persists through tests. If in one test a method of an object is mocked, the same mocked method will be used in other tests. 

```python
def test_method_1():
	utils.validate_data = mock.Mock(return_value=True)
    ...
    
def test_method_2():
	validation = utils.validate_data(content) # bingo, you are doomed!
```

There's also no `clear()` method for django mock. The way to go about it is to store the method reference to some variable, and the restore the reference after mock has done its job. Below is a a fix:

```python
def test_method_1():
	original_validate_data_fn = utils.validate_data
	utils.validate_data = mock.Mock(return_value=True)
    ...
    # after finish using the mocked function
    utils.validate_data = original_validate_data_fn
    
def test_method_2():
	validation = utils.validate_data(content) # you are safe here   
```

## Bonus: Some new cool tricks with python 3
### Dictionary comprehension
```python
dict_comprehension = {node['id']: node['content'] for node in node_list}
```
### Merging of dictionaries
```python
dict_merge = {**dict_a, **dict_b}
```

### Droping to a debugging shell when running
```python
import pdb; pdb.set_trace()
```
