---
title: 使用Python标准库上传文件
date: 2017-09-01 14:40:51
categories:
- 技术
tags: 
- python
- imhuwq
---

废话不多说啦, 三个点:  
1. `Header` 里面设置 `Content-Type`;  
2. `Header` 里面设置 `Content-Length`;  
3. 把文件内容读出来, 按格式要求写到 Body 里面.  
<!-- more -->
##  一、Content-Type
因为我们是通过 Post Form 的形式上传文件, 所以 `Content-Type` 是:  `multipart/form-data`. 
```python
# boundary 是用来分割 body 中各 field 的内容的, 推荐使用标准库来自动生成
boundary = mimetools.choose_boundary()
content_type = 'multipart/form-data; boundary=%s' % boundary
```

## 二、Body 格式
`Content-Length` 很简单， 只要构建好了 Body， Length 就是 `len(Body)`
怎么构建 Post 的 Body 部分呢？先创建一个 binary 对象: `binary = io.BytesIO()` 然后再往里面写东西就好啦. 不过写的时候注意按以下格式.  

**2.1 先创建一个分割线**
```python
part_boundary = '--' + boundary  
# 就是 Header 里面指定的 `boundary`, 把它前面再加两个减号. 
```
**2.2 构建 Form 表单每一个常规 Field**  
常规 Field 只有 `name` 和 `value`.  
```python
regular_blocks = []
for name, value in regular_fields.items():
	block = [part_boundary,
			'Content-Disposition: form-data; name="%s"' % name,
			'',
			value
			]
	block = '\r\n'.join(block)
	block = codecs.encode(block)
	regular_blocks.append(block)
regular_blocks = '\r\n'.join(regular_blocks)
```
**2.3  构建 Form 表单每一个文件 Field**  
文件 Field 有 `field_name`, `file_name`, `file_type` 和 `file_data`.  
```python
# 先搞到文件类型
file_type = mimetypes.guess_type(filename)[0] or 'application/octet-stream' 
# 再搞到文件内容
with codecs.open(file_path, 'rb') as f:
    file_data = f.read()
...
# 然后是写入文件 block
file_blocks = []
for field_name, file_name, file_type, file_data in file_fields:
    block = [part_boundary,
             # 注意这里, 相对于常规 field, 文件 field 多了一个 filename
             'Content-Disposition: form-data; name="%s"; filename="%s"' % (field_name, file_name),
             # 注意这里, 相对于常规 field, 文件 field 多了一个 Content-Type
             'Content-Type: %s' % file_type,
             '',
             file_data
             ]
    block = '\r\n'.join(block)
    file_blocks.append(block)
file_blocks = '\r\n'.join(file_blocks)
```
**2.4 把这些 blocks 写入 binary**
```python
all_field_blocks = '\r\n'.join(regular_blocks, file_blocks)
binary.write(all_field_blocks)
binary.write('\r\n--' + boundary + '--\r\n')
```
**2.5 得到 Body**
```python
body = str(binary.getvalue())
```

## 三、Content-Length
Content-Length 就是 `len(body)`

## 四. 发送数据
```python
from urllib2 import Request
from urllib2 import urlopen
from urllib2 import HTTPError
... 
# 假设你已经按照格式得到了 body
headers = {'Content-Length': len(body)),
           'Content-Type': `multipart/form-data; boundary=%s` % mimetools.choose_boundary()}
upload_req = Request(url=upload_url, data=body, headers=headers)
upload_rep = urlopen(upload_req).read()
print(upload_rep)
```

