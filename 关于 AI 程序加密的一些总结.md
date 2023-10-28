AI 程序一般至少包含 AI 模型和推理逻辑。
如果涉及到加密场景，则还要实现模型加密、模型解密以及调用计数加密。
```
AI 程序 = AI 模型 + 推理逻辑
AI 程序 = 加密模型 + 解密模型 + 推理逻辑 + 计数逻辑
AI 程序 = 加密模型 + 加密“解密模型” + (加密)推理逻辑 + 加密计数逻辑
```

本文从软件工程的角度对常见 Python AI 程序加密的方法做一些代码攻防演示，并做出以下结论：
- 在 Python 代码内做软件加密意义不大，因为 Python Interpreter 很容易被拦截
- PyArmor  结合 C/C++ 是一个兼顾开发成本和保护力度的可行方案，适合早期的项目推动
- 软件加密不是万能的，任何加密都可以被破解，难度取决于运行时机器对 Hacker 的透明度
## 1. 破解程序的常见思路
破解程序只是手段，而不是最终目的。最终目的是：
- 拿到解密后的模型 (二次开发)
- 实现无限制的算法调用 (免费使用)
所以在破解程序和防破解程序的时候，要防止掉入思维陷阱。

破解程序可以在程序运行前(静态的)，也能在程序运行时(动态的)。
- 静态破解，逆向程序逻辑，从中获取解密逻辑(和解密密钥)
	- 相当于破解银行卡密码，自己从里面取钱
	- 常用手段有：
		- 代码分析
		- `so` 二进制文件逆向
- 动态破解，拦截关键逻辑，从而绕过加密逻辑的保护
	- 相当于伪装成骗子商户，拦截用户的付款
	- 常用手段有：
		- hook Python import 机制
		- 替换 Python Interpreter 或依赖包
		- 拦截系统动态链接
		- 修改或者 dump 内存
		- 伪造硬件信息/系统时间

程序无论如何加密，在运行时总是对机器透明的，所以运行时破解往往手段更多、效果更好，并且理论上使得**所有程序都可以被破解**。
破解难度只取决于运行时的机器对 Hacker 的透明度。
## 2. 应对破解的主要手段
程序在运行时对机器总是透明的。所以防止破解最有效的手段就是物理上隔离 Hacker 和运行时机器。

在私有化部署场景下，不得不把程序交付给客户端并且在客户端机器上运行时，任何防破解手段理论上就无法提供绝对保护。以下是一些常用手段：
- 应对静态破解
	- 代码混淆(降低可读性)
	- 代码编译(从明文代码编译为汇编指令)
	- 动态代码生成(在运行时再生成代码，不留下可分析的文件)
- 应对动态破解
	- 虚拟机保护
	- 独立实现核心逻辑，并进行加密

下面对各种 Python 程序加密手段进行分析。  
为了量化各种方案的利弊，首先从实现成本和保护力度两方面设立评价指标。 
### 2.1 实现成本的指标
| 成本 | 现有方案 | 额外投入 | 
| --- | --- | --- |
|✨ | 有成熟的方案可以实现 | 基本不再需要额外处理 |
|✨✨ | 有成熟的方案 | 需要额外处理一些**适配性**问题 |
|✨✨✨ | 有方案 | 需要对代码做一些**局部重构** |
|✨✨✨✨ | 有方案 | 需要对代码做**非常多的重构** |
|✨✨✨✨✨ | 有思路，理论上可行 | 需要投入非常多的资源进行底层定制 |

### 2.2 保护力度的指标
| 保护 | 获取源码 |  分析逻辑 | 拦截调用 |
| --- | --- | --- | --- |
| ✨ | 还是 Python 代码或者字节码 | 容易分析 | 容易被拦截 |
| ✨✨|  不可逆为 Python 代码 | 能被逆向分析汇编指令 | 容易被拦截 |
| ✨✨✨ | 不可逆为 Python 代码 | 不能被逆向分析汇编指令 | 容易被拦截 |
| ✨✨✨✨ | 不可逆为 Python 代码 | 能被逆向分析汇编指令 | 很难被拦截 |
| ✨✨✨✨✨ | 不可逆为 Python 代码 | 不能被逆向分析汇编指令 | 很难被拦截 |

 在这个评级中，“被拦截”的重要性比“被逆向分析汇编指令”高，因为：
 - 防拦截可以提高保护上限：防逆向/逆向对于 Developer/Hacker 来说是基操
 - 防拦截更容易成为程序漏洞：Hacker 可以拦截的情况下，经常不用逆向也能达到破解目的

下面开始各种加密方案的详细分析。
## 3. AI 模型加密
对 AI 模型进行加密的方法，一般是使用对称加密算法或者非对称加密算法，对模型的二进制文件加密后再进行分发。在客户端进行推理前，先使用**密钥解密模型，再进行推理**。 

以下是加密模型的示例。我们可以在交付模型前先用这个逻辑把模型进行加密：
```python
"""
加密部分的逻辑
"""

import io
import os

import torch
from torchvision import models
from cryptography.fernet import Fernet

def encrypt_bytes(plain_bytes: bytes, enc_key: str):
    key = enc_key.encode("utf8")
    encrypted_bytes = Fernet(key).encrypt(plain_bytes)
    return encrypted_bytes


def save_encrypted(save_path: str):
    """
    使用 enc_key 把模型 state dict 加密，然后把加密后的二进制字节保存到 save_path。
    """
    enc_key = "k6oRK5yvag4mWTKrh_e3qNvpRYYLozThjK6V5yLhCmk="
    save_path = os.path.abspath(save_path)
    os.makedirs(os.path.dirname(save_path), exist_ok=True)

    model = models.vgg16(pretrained=True)
    io_bytes = io.BytesIO()
    torch.save(model.state_dict(), io_bytes)
    io_bytes.seek(0)

    plain_bytes = io_bytes.read()
    encrypted_bytes = encrypt_bytes(plain_bytes, enc_key)

    with open(save_path, "wb") as fp:
        fp.write(encrypted_bytes)

```

以下是使用加密模型的示例。我们把它放在一个 `protect` 模块中交付给客户。
```python
"""
protect/load.py

使用加密模型的示例。
模型先得解密再能使用。
如果解密部分的逻辑是用 Python 实现的(如本例)，那就相当于没有加密。
"""

import io
import os
import json
from typing import List

import torch
from torchvision import models
import torchvision.transforms as transforms
from cryptography.fernet import Fernet
from PIL import Image


def decrypt_bytes(encrypted_bytes:bytes, enc_key:str):
    key = enc_key.encode("utf8")
    decrypted_bytes = Fernet(key).decrypt(encrypted_bytes)
    return decrypted_bytes

def load_encrypted(model_path:str):
    """
    从 model_path 读取加密后的 state_dict 二进制字节，使用 enc_key 解密，
    再使用 pytorch 进行加载。
    """
    enc_key = "k6oRK5yvag4mWTKrh_e3qNvpRYYLozThjK6V5yLhCmk="
    
    model_path = os.path.abspath(model_path)
    with open(model_path, "rb") as fp:
        encrypted_bytes = fp.read()
    decrypted_bytes = decrypt_bytes(encrypted_bytes, enc_key)

    io_bytes = io.BytesIO(decrypted_bytes)
    io_bytes.seek(0)

    model = models.vgg16()
    state_dict = torch.load(io_bytes)
    model.load_state_dict(state_dict)

    return model

def predict(model, image_file:str, labels:List[str]):
    """
    使用解密并加载后的模型进行预测。
    """
    
    print(f"Predicting image: {image_file}")
    if not os.path.exists(image_file):
        print(f"Image file not found: {image_file}")
        return
    
    img = Image.open(image_file)
    img = Image.open(image_file).convert("RGB")

    # Define the image transformation
    preprocess = transforms.Compose([
        transforms.Resize(256),
        transforms.CenterCrop(224),
        transforms.ToTensor(),
        transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
    ])

    # Preprocess the image
    input_tensor = preprocess(img)
    input_batch = input_tensor.unsqueeze(0)

    # Check if a GPU is available and if not, use a CPU
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

    model = model.to(device)

    # Perform inference
    with torch.no_grad():
        input_batch = input_batch.to(device)
        output = model(input_batch)

    _, predicted_idx = torch.topk(output, 3)
    for idx in predicted_idx[0]:
        print(f"Predicted label: {labels[idx]}")

```

客户这样来使用我们的模型做预测：
```python
image = "data/dog.png"
labels = "data/labels.json"
plain_model = "data/vgg16.pth"
encrypted_model = "data/vgg16_encrypted.pth"

with open(labels, "r") as fp:
    labels = json.load(fp)

model = load_encrypted(encrypted_model)
predict(model, image, labels)
```

客户端在进行推理之前，客户端必须解密模型。但是解密模型的 key 和逻辑，也必须以某种形式交付给客户端进行部署。  

这是致命的漏洞。如果把模型用最先进的技术加密，但是用明文的方式写上了解密步骤，就和“**把钱存到银行，但是把密码写到银行卡背面**”是一样的。  

所以加密 AI 程序**虽然起点在模型加密，但难点在程序加密**。
## 4. AI 程序加密

AI 程序中至少有两个部分需要加密：
- 解密模型的逻辑
- 调用计数的逻辑
### 4.1 使用代码混淆
基本只是文本上的混淆，输出的仍是 Python 代码或者 `pyc`(Python 解释器的字节码) 文件，保护程度相当少。  
唯一的优势是实现简单。
成本：✨
保护：✨

这个太简单了，不详细展开。  
```
顺带说一下，经过测试，Python 3.8 及以下版本可以使用常用的 uncompyle6 进行逆向。
Python 3.9 及以上能用 pycdc(https://github.com/zrax/pycdc) 工具进行逆向。
```

### 2.2 使用 PyArmor 加密 Python 代码
Pyarmor 的基本原理是把 Python 代码混淆后加密为不可逆的二进制数据，在调用时解密这些二进制再交给 Python 去运行，运行完后从内存立即删除解密后的数据。  

这种方案下，编译后的 Python 代码几乎不可能进行分析，但是在运行时仍然依赖 Python 、PyArmor 和其它系统库，且有一定性能损耗，不能防止非法调用。

以下面的加载模型的脚本 `protect/load.py` 为例。
```python
import io
import os

import torch
from cryptography.fernet import Fernet
from torchvision import models


def load_plain(model_path: str):
    model = models.vgg16()
    state_dict = torch.load(model_path)
    model.load_state_dict(state_dict)

    return model


def decrypt_bytes(encrypted_bytes: bytes, enc_key: str):
    key = enc_key.encode("utf8")
    decrypted_bytes = Fernet(key).decrypt(encrypted_bytes)
    return decrypted_bytes


def load_encrypted(model_path: str):
    enc_key = "k6oRK5yvag4mWTKrh_e3qNvpRYYLozThjK6V5yLhCmk="

    model_path = os.path.abspath(model_path)
    with open(model_path, "rb") as fp:
        encrypted_bytes = fp.read()
    decrypted_bytes = decrypt_bytes(encrypted_bytes, enc_key)

    io_bytes = io.BytesIO(decrypted_bytes)
    io_bytes.seek(0)

    model = models.vgg16()
    state_dict = torch.load(io_bytes)
    model.load_state_dict(state_dict)

    return model

```

我们可以使用 PyArmor 加密整个 `protect` 模块，再交付给客户。
```
pyarmor gen protect -i \
      --enable-jit \
      --assert-call \
      --assert-import \
      --restrict
```

加密后我们得到加密后的 `dist/protect` 模块，用它来代替原本的 `protect` 模块后再交付给客户。
```
$ tree -L 3 dist
dist
└── protect
    ├── __init__.py
    ├── load.py
    └── pyarmor_runtime_000000
        ├── __init__.py
        └── pyarmor_runtime.so


$ head dist/protect/load.py -c 200
# Pyarmor 8.4.0 (trial), 000000, non-profits, 2023-10-24T15:19:01.047851
from .pyarmor_runtime_000000 import __pyarmor__
__pyarmor__(__name__, __file__, b'PY000000\x00\x03\t\x00a\r\r\n\x80\x00\x01\x00% 
```

可以看到加密后的 `load.py` 文件中已经被替换成了加密的字节码。这个字节码必须在运行时依赖 `pyarmor_runtime.so` 进行解密才能运行。  

在收费版中，PyArmor 据称使用了完全不可逆的方式来加密 Python 代码。如果这是可信的话，那相比起编译 Python 代码为常规的 `so`，PyArmor 的加密是更牢固的。因为 `so` 的逆向已经有非常多的配套工具和经验。 

尝试运行程序后，我们发现加密后的模块**仍然生成了 `pyc` 文件**，不过用 `pycdc` 逆向后看到其中仍然是加密后的字节码。
```shell
$ tree -L 3 dist
dist
└── protect
    ├── __init__.py
    ├── load.py
    ├── pyarmor_runtime_000000
    │   ├── __init__.py
    │   ├── pyarmor_runtime.so
    │   └── __pycache__
    └── __pycache__
        ├── __init__.cpython-39.pyc
        └── load.cpython-39.pyc


pycdas dist/__pycache__/load.cpython-3.9.pyc >> load.pycdas
cat load.pycdas
```

```python
load.cpython-39.pyc (Python 3.9)
[Code]
    File Name: .../dist/protect/load.py
    Object Name: <module>
    Arg Count: 0
    Pos Only Arg Count: 0
    KW Only Arg Count: 0
    Locals: 0
    Stack Size: 4
    Flags: 0x00000040 (CO_NOFREE)
    [Names]
        'pyarmor_runtime_000000'
        '__pyarmor__'
        '__name__'
        '__file__'
    [Var Names]
    [Free Vars]
    [Cell Vars]
    [Constants]
        1
        (
            '__pyarmor__'
        )
        b'PY000000\x00\x03\t\x00a\r\r\n\x80\x00\x01\x00\x08\x...'
        None
    [Disassembly]
        0       LOAD_CONST                    0: 1
        2       LOAD_CONST                    1: ('__pyarmor__',)
        4       IMPORT_NAME                   0: pyarmor_runtime_000000
        6       IMPORT_FROM                   1: __pyarmor__
        8       STORE_NAME                    1: __pyarmor__
        10      POP_TOP                       
        12      LOAD_NAME                     1: __pyarmor__
        14      LOAD_NAME                     2: __name__
        16      LOAD_NAME                     3: __file__
        18      LOAD_CONST                    2: b'PY000000\x00\x03\t\x00a\r\r\n\x80\x00\x01\x00\x08\...'
        20      CALL_FUNCTION                 3
        22      POP_TOP                       
        24      LOAD_CONST                    3: None
        26      RETURN_VALUE                  
```

但是正如之前所说的，破解程序不是目的，只是手段。  
很多时候**不需要破解**程序代码或者逻辑，**只需要拦截**一些关键 API 的调用就可以了。

比如说大部分加密/解密程序都用到了 `cryptography` 依赖包，大部分 AI 程序都用到了 `torch`，那只要拦截他们的关键 `API` 即可。  

举个例子：
```python
def hack_crypto():
    from cryptography import fernet

    old_Fernet = fernet.Fernet

    class Fernet(old_Fernet):
        def __init__(self, *args, **kwargs):
            super(Fernet, self).__init__(*args, **kwargs)
            print(f"Initializing Fernet, args={args}, kwargs={kwargs}")
            print("You are hacked!!!")

    fernet.Fernet = Fernet
    print("Hijack fernet.Fernet success!")


hack_crypto()
```

Hacker 可以把上面的程序注入到程序启动时。
虽然我们在调用时可以强制 `reload` 模块，但是 Hacker 也可以直接替换 `cryptography` 的源代码文件，导致 `reload` 后也是它的逻辑。

综上，PyArmor 只能起到保护 Python 代码自身的作用，但是仍然依赖 Python 运行时，很容易被拦截。
成本: ✨✨，有成熟的方案，需要额外处理一些适配性问题
保护: ✨✨✨，不可逆为 Python 代码，不能被逆向分析汇编指令，容易被拦截

### 2.3 编译 Python 为二进制 `so`
使用 Cython 能够比较快地把 Python 程序专为动态链接库 `so` 文件。  

还是以 `protect` 模块为例，可以用 `setup.py` 比较快地把一个模块打包为动态链接库：

```python
"""
setup.py 用来打包如下结果的 module:

$ tree -L 2 protect
protect
├── __init__.py
├── load.py
├── predict.py
└── utils.py
"""

from setuptools import setup, find_packages
from Cython.Build import cythonize
import glob

setup(
    name="protect",
    packages=find_packages(),
    ext_modules=cythonize(
        glob.glob("protect/*.py"),
        build_dir="cython_build",
    )
)
```

不过，这种方案同样不能防止非法调用。相比起 `PyArmor`, 它的保护力度更小，但是可以通过 C/C++ 实现关键逻辑来摆脱对 Python 的依赖，从而更好地防止拦截。这可以提高保护上限。

成本：✨✨，有成熟的方案，需要额外处理一些适配性问题
保护：✨✨，不可逆为 Python 代码，可以被逆向分析汇编指令，容易被拦截

### 2.4 使用 C/C++ 重写关键逻辑
上面的两种方案能在某种程度上保护 Python 代码，但是不能防止拦截关键调用。

当代码中使用了第三方库来做一些加密/解密操作的时候，Hacker 可以预先 Hook 这些模块，捕获输入和输出，从而知道你的输入和输出。如果依赖的是 Python 的库，那这样的拦截更简单。

举个比上面 `hack_crypto` 更有效的例子。绝大部分 AI 程序都用到了 `torch` 框架。
不管 AI 程序是用什么算法加密的，也不管解密逻辑做了多少保护，只要还在调用 `torch.load` 方法，Hacker 就能用类似  `hack_crypto` 的方法拦截对这个函数的调用，把它的返回值 dump 出来一份再返回给调用者，从而获取到了解密后的模型。  

```python
# 直接修改 site-packages/torch/nn/modules/module.py 文件
# 把原来的 Module class 重命名为 _Module，然后自己实现一个 Module 做封装

class Module(_Module):
    def load_state_dict(self, *args, **kwargs):
        ret = super().load_state_dict(*args, **kwargs)
        torch.save(ret, "hacked_load.bin" )
        print("Hacked load_state_dict! Result is saved to hacked_load.bin")
        return ret
```

所以对于一些关键算法，要避免使用第三方库，而是自己用 C++ 实现。
但是根据需要自行实现的库的复杂度，实现成本也会相应增长。

比如说，为了摆脱对 `torch.load` 方法的依赖，我们可以在 C++ 里面实现 load 逻辑。  
```C++
#include <torch/script.h>
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include <sstream>
#include <iostream>

namespace py = pybind11;

// 在此处实现一个加载 bytes 到 model 的逻辑，再返回给 python
// 这里只是一个简单说明，我们可以在这里面再实现 bytes 的解密逻辑
// 当然，我们调用这个函数的代码，也需要保护起来，但那可以用 Cython 或者 Pyarmor 即可
torch::jit::script::Module load_bytes(const py::bytes& model_bytes) {
    std::string model_data = model_bytes;
    std::istringstream model_stream(model_data);
    model_stream.seekg(0, model_stream.beg);
    if (!model_stream.good()) {
        std::cerr << "Stream is not good!" << std::endl;
    }

    torch::jit::script::Module module;
    try {
        module = torch::jit::load(model_stream);
        std::cout << "Load model successfully and safely~" << std::endl;
    } catch (const c10::Error& e) {
        std::cerr << "Error loading the model: " << e.what() << std::endl;
        exit(1);
    }

    return module;
}

PYBIND11_MODULE(load, m) {
    m.def("load_bytes", &load_bytes, "Load TorchScript model");
}

```

编译成 `so` 后并非没有被拦截的可能，操作系统本身就有各种 `interception` 操作。  
但至少它提到了难度：
- 我们的 `so` 是私有的，不是一些公共知名的链接库，没有明显的范式可以参考
- 我们可以把保护区域划大一点，比如把模型加载和推理的逻辑全部封装起来

同样也有一些额外措施可以提高逆向难度：
- 去除 `so` 中的所有符号：`strip -s $file.so`  
- 减少常量字符串的使用，尤其是全局常量字符串存储密钥
- 减少关键路径中的 log，提高 Hacker 的调试成本
- 提高关键路径中的判断复杂度
	- 避免一次变量判断就通过所有检测

成本：✨✨✨✨
保护：✨✨✨✨

### 2.5 综合 Cython 和 PyArmor
PyArmor 加密不可逆、不可分析汇编指令(✨✨✨保护)，但是不能防止拦截。
C++ 编译不可逆、可以分析汇编指令的(✨✨保护)，并且可以提高拦截的门槛。

对于一些关键逻辑，我们可以用 C++ 来实现，并在 PyArmor 加密的关键入口中进行调用。  
这样一来，Hacker 很难分析关键入口的代码逻辑，也很难拦截我们的关键 API 调用。
并且我们可以保持代码仓整体最小的改动，尽可能降低实现成本。

不独立实现核心库的情况下：
成本：✨✨
保护：✨✨(整体代码) + ✨✨✨(关键入口)

使用 C/C++ 独立实现核心库的情况下：
成本：✨✨✨✨
保护：✨✨✨✨(整体代码) + ✨✨✨✨✨(关键入口)

### 2.6 软件加密方案的总结
上面分析了 5 种方案，并且给出了对应评级。

| 方案 | 成本 | 保护 |
| --- | --- | --- |
| 代码混淆 | ✨ | ✨ |
| PyArmor | ✨✨ | ✨✨✨ |
| Cython | ✨✨ | ✨✨ |
| C/C++ | ✨✨✨✨ | ✨✨✨✨ |
| PyArmor + C/C++ (公共库) | ✨✨ | ✨✨✨ |
| PyArmor + C/C++ (私有库) | ✨✨✨✨ | ✨✨✨✨✨ |


虽然星级分 1 到 5 等，但是不管是保护“ AI 模型”还是保护“调用计数”，**至少要达到四颗星**的保护(能够防止拦截调用)，才能起到真正意义上的保护，**否则就是徒劳无功**。

上面多次提到编译私有 `so` 动态链接库可以提高拦截的门槛。但其实  `so` 的逆向并非完全不可行。之所以这么强调 `so` 的安全性，主要想表达的意思是：**在 Python 内做程序加密意义不大，还是得依靠编译**。  

如果把 Developer 和 Hacker 的攻防来打比方的话：
- 在 Python 体系内做安全加密，就像是小孩和大人打架，Developer 是小孩；
- 回到编译型领域，虽然不是一劳永逸，至少是两个成年人之间的公平对决。

基于此再来做选择的话，**综合 Cython 和 PyArmor** 是比较合适的、渐进的方案。
它起步简单，并且可以随着项目推进而逐步加固。
## 4. 整体总结
- Python 加密意义不大，还是得靠编译
	- Python 本身不能防止拦截，轻松就能修改依赖库
	- 编译后的动态链接库 `so` 有更高的逆向和拦截成本
- 软件加密总是能被破解的
	- 加密程序对运行机器总是透明的
	- 破解难度 = 运行机器对 Hacker 的透明度
	- 破解可能性 = 破解收益 - 破解难度 
- 软件加密和项目合作的平衡
	- PyArmor + C/C++ 能够应对非专业黑客，可以确保项目早期能够推动执行
	- VMProtect 能够应对独立专业黑客，成本更高，只能对编译型程序生效
		- [VMProtect](https://vmpsoft.com/products/vmprotect)
	- 机密计算芯片能够保护较高价值的程序，成本更高，需要接触部署机房硬件
		- [Nvidia H100](https://developer.nvidia.com/zh-cn/blog/protecting-sensitive-data-and-ai-models-with-confidential-computing/)
		- [IDEA SPU](https://www.idea.edu.cn/news/5647.html)
	- 不考虑把超高价值的程序完整离线交付给客户
