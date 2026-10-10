[REPORT-checkpoint.md](https://github.com/user-attachments/files/33282907/REPORT-checkpoint.md)
# Lab 1 实验报告

## 学生信息

- **姓名：田昕媛**
- **学号：112404260111**
- **班级：数据1241**
- **GitHub 用户名：tian11543**
- **仓库地址：https://github.com/tian11543/bigdata-collection-labs-112404260111.git**

---

# 任务 1：判断数据加载方式

## 1.1 Device：判断动态加载并找到 API

### 判断结果

- 页面：`/devices`
- 数据加载方式：`动态加载`
- 判断依据：源代码找不到设备名称, 在“查看网页源代码”中不能搜索到页面上的设备名称，在 Network 的 Fetch/XHR 中能找到返回设备数据的请求

### API 信息

- Request URL：https://115.159.159.82/api/devices?page=1&page_size=50
- Request Method：GET
- Query 参数：page=1&page_size=50

### 【截图 1】Device API 发现证据
文件名：![Device API 发现证据](images/01_device_api.png)

---

## 1.2 Company：判断静态加载

- 页面：`/companies`
- 数据加载方式：`静态加载`
- 判断依据：使用“右键 → 查看网页源代码”后，能搜索到页面中真实显示的企业名称。



### 【截图 2】Company 静态加载证据

文件名：![Company 静态加载证据](images/02_company_source.png)

---

# 任务 2：Device API 分页采集

## 2.1 API 分页结构

填写你实际从 JSON 中观察到的字段：

| 含义 | JSON 字段 |
|---|---|
| 当前页 |page |
| 每页条数 | page_size|
| 设备总数 |total |
| 设备列表 | items|

---

## 2.2 关键代码

```python
# 从第一页响应中读取分页信息
total = data["total"]
page_size = data["page_size "]

total_pages = (total + page_size - 1) // page_size

devices = []

for page in range(1, total_pages + 1):
    r = requests.get(
        api_url,
        params={
            "page": page,
            "page_size": page_size
        },
        timeout=10
    )

    page_data = r.json()
    page_items = page_data["items"]
    devices.extend(page_list)

print("最终采集设备数量：", len(devices))
```

---

## 2.3 采集结果

- 每页条数：page_size=5000
- 设备总数：total=5000
- 程序计算得到的总页数：total_pages=100
- 最终采集设备数：5000
- 唯一 `device_id` 数：5000

回答：

> 为什么总页数应由 API 返回的 `total` 和 `page_size` 计算，而不是直接在代码里写死？

答：后端设备总数量可能发生变化，写死页数会出现采集不全或者请求无效空页面；利用接口返回的total与page_size动态计算总页数，代码不用修改，自动适配数据量变化。

---

# 任务 3：Company HTML 解析

## 3.1 企业记录 Selector

- 企业记录 selector：article.company-card
- 第 1 页企业记录数：20

说明为什么选择这个 selector：
- 主要使用了 `id / class / data-* / 父子层级` 中的哪一种？
- 是否参考过浏览器 Copy selector 或 AI？
- 最终为什么采用现在这个写法？

答：主要使用了 class 选择器，结合标签名限定，article.company-card 匹配class 为 company-card 的 article 标签。
没有直接使用浏览器 Copy selector 生成的长路径；参考网页原始 HTML 源码，同时借助 AI 辅助确认节点结构。
浏览器直接复制的 selector 一般带有nth-child这类位置选择器，页面顺序变动就会失效。article.company-card 直接定位企业卡片本身，不依赖元素位置，简洁稳定，只要卡片 class 不变，就能正常提取全部企业记录，鲁棒性更强。



---

## 3.2 关键代码

只填写“记录定位 + 一条记录的关键字段”。

```python
cards = soup.select("article.company-card")

card = cards[0]

company_id = card.get("data-company-id")
name = card.select_one("h2 a").get_text(strip=True)
status = card.select_one(".badge").get_text(strip=True)

行业、区域、设备数量使用 `dt/dd` 结构后，请写出你最终如何得到：

```python
industry = details["所属行业"]
zone = details["所在区域"]
device_count = int(details["设备数量"].replace("台","").strip()
```

---

## 3.3 字段来源说明

任选 **2 个字段**说明：

1. 字段：company_id
   - 来自：` HTML 属性`
   - 使用的方法：card.get("data-company-id")

2. 字段：name
   - 来自：`文本 `
   - 使用的方法：select_one("h2 a") + get_text(strip=True)

---

# 任务 4：Company “下一页”分页

## 4.1 下一页元素

- “下一页” selector：遍历.pagination 容器内部 a 标签，匹配文本 “下一页”
- 第 1 页 `href` 示例：/companies?page=2

### 【截图 3】Company “下一页”元素

文件名：![Company 下一页元素](images/03_company_next.png)




---

## 4.2 关键代码

```python
next_link = soup.select_one(".pagination")

next_href = next_link.get("href")

next_url = urljoin(
    current_url,
    next_href
)
```


> 你的程序如何完成“当前页 → 找下一页 href → 请求下一页 → 再解析”的循环？

答：循环先请求 current_url 获取页面 HTML，解析本页全部企业存入列表；之后找到分页容器.pagination，遍历内部 a 标签，匹配文字 “下一页” 拿到相对 href；通过 urljoin 拼接完整 URL 赋值给 current_url；下一轮循环访问新地址，循环执行 3 次完成采集。

---

## 4.3 三页采集结果

| 页码 | 企业数 |
|---|---:|
| 1 | 20|
| 2 | 20|
| 3 |20 |
| **合计** | 60|

- 唯一 `company_id` 数：60

---

# 任务 5：结果验证

## 5.1 最终结果

| 检查项 | 结果 |
|---|---:|
| Company 总数 |60 |
| 唯一 `company_id` |60 |
| Device 总数 | 5000|
| 唯一 `device_id` |5000 |
| Company–Device 不一致企业数 |0 |

### 【截图 4】最终验证结果

文件名：![最终验证结果](images/04_validation.png)


---

## 5.2 一致性检查关键代码

```python
from collections import Counter

device_counts = Counter(
    x["company_id"] for x in devices
)

mismatch_count = 0

for company in companies:
    cid = company["company_id"]
    expected = company["device_count"]
    actual = device_counts.get(cid,0)

    if  expected != actual:
        mismatch_count += 1
```

回答：

> 为什么 `status_code == 200` 仍然不能说明“数据已经采集正确”？

答：status_code=200 仅代表 HTTP 网络请求成功，服务器正常返回响应报文；但是返回内容可能为空、返回部分数据、返回错误提示。还必须校验业务层面：数据条数、ID 唯一性、字段完整性、跨数据源对比，才能确认采集结果正确。

---

# 任务 6：保存采集结果
- `companies.json`：`已生成 `
- `devices.json`：`已生成`
---

# 实验结论
1. 静态加载与动态加载页面的处理方式有什么不同；
2. HTML “下一页”分页和 API `page` 分页有什么不同；
3. 你使用了哪些方法验证采集结果。

答：静态页面业务数据嵌入原始 HTML 源码，直接请求网页文本，使用 BeautifulSoup 解析；动态页面初始 HTML 不含业务数据，需要抓 Fetch/XHR 接口获取 JSON。API 分页直接传递 page 参数；HTML 分页需要解析下一页 a 标签的 href 拼接完整链接循环访问。本实验通过校验总记录条数、ID 集合去重、企业‑设备数量跨数据源比对完成结果校验，最后将采集数据持久化为 JSON 文件。
