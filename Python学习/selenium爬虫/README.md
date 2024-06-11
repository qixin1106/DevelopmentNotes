# selenium模拟浏览器操作

## 环境

python 3.10

## 安装库

`pip install selenium`

## 安装Chrome插件

https://chromedriver.storage.googleapis.com/index.html

> 插件安装时，需要找到你Chrome版本，找一个最接近的版本，按照你的系统下载对应的版本

如果是较新的版本，就别从上面链接中下载了，官网建议114版本以上从这里找：

https://googlechromelabs.github.io/chrome-for-testing/

因为我本地版本是`125.0.6422.142`，所以最近的一个版本是`125.0.6422.141`，就下载这个。

> 你需要根据你本地Chrome的版本来选择

#### 方案1：

将下载的`chromedriver`文件放置在你Python运行环境的bin文件夹中

> 你可以通过`which python`命令来查看当前环境的地址
> 
> 如：`/opt/homebrew/anaconda3/envs/Python_3_10_14_env/bin/python`
> 
> 那么你就将文件放在`/opt/homebrew/anaconda3/envs/Python_3_10_14_env/bin`下面

#### 方案2：

将`chromedriver`放在你的项目目录下。

>  建议将其放在项目目录下，原因如下：
> 
> - **项目独立性**:这样可以使每个项目都有自己专属的 `chromedriver` ，避免不同项目因依赖不同版本的 `chromedriver` 而产生冲突。如果放在 Python 解释器文件夹下，可能会影响到其他项目的运行。
> 
> - **可移植性**：当你将项目移动到其他位置或与他人共享时，不需要额外考虑 `chromedriver` 与 Python 解释器的关联问题，项目文件夹内包含了所有相关依赖，更易于管理和部署。

## 兼容问题

如果你的版本不匹配，会报这个错误。请按之前步骤仔细确认。

```
发生异常: SessionNotCreatedException
Message: session not created: This version of ChromeDriver only supports Chrome version 114
Current browser version is 125.0.6422.142 with binary path /Applications/Google Chrome.app/Contents/MacOS/Google Chrome
```

## 基本使用

打开一个网页，并获取网页title

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

# 设置 ChromeDriver 的路径
chromedriver_path = './selenium测试/chromedriver'  # 确保 chromedriver 文件位于同级目录中

# 创建一个 Chrome 浏览器实例
service = Service(chromedriver_path)
# 如果将chromedriver放在bin中，则不需要指定path
# service = Service()
driver = webdriver.Chrome(service=service)

# 打开 baidu.com
driver.get('https://www.baidu.com')

# 获取网页的Title
print(driver.title)
```

> 如果使用了方法1，将chromedriver放在bin中，则不需要指定path，直接service = Service()

## 获取元素

比如我们打开baidu之后，获取搜索框元素，并填充一个要搜索的值

```python
...
# 导入库
from selenium.webdriver.common.by import By

# 其他代码，略，请参考之前
...

# 打开 baidu.com
driver.get('https://www.baidu.com')

# 查找某个元素，这里使用Xpath语法解析
item = driver.find_element(By.XPATH, '//*[@id="kw"]')
print(item)
"""
<selenium.webdriver.remote.webelement.WebElement (session="79a412a07652f525fb08627d34bb9415", element="f.56CA51871077DC88EAF528738D44325B.d.C32949B669205A41D637985F3A79DD86.e.6")>
"""

# 获取多个元素列表
items = driver.find_elements(By.XPATH, '//*[@id="hotsearch-content-wrapper"]/li/a/span[2]')
print(items)
'''
[
    <selenium.webdriver.remote.webelement.WebElement (session="79a412a07652f525fb08627d34bb9415", element="f.56CA51871077DC88EAF528738D44325B.d.C32949B669205A41D637985F3A79DD86.e.16")>, 
    <selenium.webdriver.remote.webelement.WebElement (session="79a412a07652f525fb08627d34bb9415", element="f.56CA51871077DC88EAF528738D44325B.d.C32949B669205A41D637985F3A79DD86.e.17")>, 
    <selenium.webdriver.remote.webelement.WebElement (session="79a412a07652f525fb08627d34bb9415", element="f.56CA51871077DC88EAF528738D44325B.d.C32949B669205A41D637985F3A79DD86.e.18")>, 
    <selenium.webdriver.remote.webelement.WebElement (session="79a412a07652f525fb08627d34bb9415", element="f.56CA51871077DC88EAF528738D44325B.d.C32949B669205A41D637985F3A79DD86.e.19")>, 
    <selenium.webdriver.remote.webelement.WebElement (session="79a412a07652f525fb08627d34bb9415", element="f.56CA51871077DC88EAF528738D44325B.d.C32949B669205A41D637985F3A79DD86.e.20")>, 
    <selenium.webdriver.remote.webelement.WebElement (session="79a412a07652f525fb08627d34bb9415", element="f.56CA51871077DC88EAF528738D44325B.d.C32949B669205A41D637985F3A79DD86.e.21")>
]
'''
```

## 键盘事件

上面获取到了搜索框之后，可以对搜索框进行输入以及回车确认搜索

```python
...
# 导入库
from selenium.webdriver.common.keys import Keys

# 之前的操作
...

# 查找搜索框元素
item = driver.find_element(By.XPATH, '//*[@id="kw"]')

# 输入字符串，并回车操作，keys可以传入多个值，按顺序执行
# 输入->'python'->回车
item.send_keys('python', Keys.ENTER)

# 输入->'hello'->'world'->回车，相当于输入了helloworld
# item.send_keys('hello', 'world', Keys.ENTER)
```

## 获取当前页面html

页面刷新完之后你可以获取到新页面的html代码

```python
print(driver.page_source)
```

也可以继续在新页面中，执行`driver.find_element`等操作继续查找元素

## 无头浏览

如果不想每次都打开浏览器，可以不打开，需要进行一些配置工作

```python
...
# 导入库
from selenium.webdriver.chrome.options import Options

# 之前的操作
...

# 无头浏览器配置
opt = Options()
opt.add_argument("--headless")
opt.add_argument("--disable-gpu")
driver = webdriver.Chrome(service=service, options=opt)

# 接下来的操作如前
...
```


