<a name="stock-knowledge-graph"></a>
# stock-knowledge-graph
利用网络上公开的数据构建一个小型的证券知识图谱（知识库）。
<a name="MU9S1"></a>
# 数据获取
本此实践项目需要用到两种数据源：一种是公司董事信息，另一种是股票的行业以及概念信息。
<a name="NfPL4"></a>
##  公司董事信息
这部分数据是由[同花顺个股](http://stockpage.10jqka.com.cn/)的⽹页爬取而来的。

- ![image.png](https://cdn.nlark.com/yuque/0/2024/jpeg/38415538/1711097014198-d36588e7-9d4d-4bd9-85ab-d220e084afcb.jpeg#averageHue=%23f3f2f1&clientId=u0273a9a4-01fd-4&from=paste&id=ue5b76cc7&originHeight=1889&originWidth=1860&originalType=url&ratio=1.25&rotation=0&showTitle=false&size=546029&status=done&style=none&taskId=u645f278c-c9a9-4599-9107-f8604fc93aa&title=)
<a name="ra703"></a>
### 定位抽取信息
在我们给定的html文件中，需要对每一个股票/公司抽取董事会成员的信息，这部分信息包括董事会成员“姓名”、“职务”、“性别”、“年龄”共四个字段。首先，姓名和职务的字段来自于：在这里总共有12位董事成员的信息，都需要抽取出来。另外，性别和年龄字段也可以从下附图里抽取出来。
<a name="bAo71"></a>
### 使用Xpath对数据进行提取
![image.png](https://cdn.nlark.com/yuque/0/2024/png/38415538/1711097729119-69bf961e-fba3-4ed7-90ad-43ba9ce56e77.png#averageHue=%23f7f6f5&clientId=u0273a9a4-01fd-4&from=paste&height=559&id=u205de8ad&originHeight=699&originWidth=1822&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=188788&status=done&style=none&taskId=u1e64f0cf-f2aa-4e09-b7e8-620b89fddcb&title=&width=1457.6)<br />这部分数据包含在`data`目录下的`stockpage`压缩文件中，⾥面的每一个文件是以`XXXXXX.html`命名，其中`XXXXXX`是股票代码。
```python
def extract(stockpage_dir, executive_csv):
    """从网页中对董事会信息进行提取
Args:
stockpage_dir:证券网页的压缩文件夹
executive_csv: 生成的csv存储路径
"""
#提取文件夹中的html网页
pages = map(lambda : os.path.join(stockpage_dir, ), os.listdir(stockpage_dir))
pages = filter(lambda : .endswith('html'), pages)

#绘制表头
headers = ['name', 'gender', 'age', 'code', 'jobs']
with open(directors_csv, 'w', encoding='utf-8') as file_directors:
    file_directors_csv = csv.DictWriter(file_directors, headers)
file_directors_csv.writeheader()

#针对每一个需要提取证券信息的公司网页
for page in pages:
    file_name = page.split(r'/')[-1]
    code = file_name.split('.')[0]
    executives = []
    with open(page, 'r', encoding='gbk') as file_page:
        content = file_page.read()
        
        #使用Etree对网页内容进行解析
        html = etree.HTML(content)
        divs = html.xpath('//div[@id="ml_001"]//div[contains(@class, "person_table")]')
        for div in divs:
            item = {}
            item['name'] = div.xpath('.//thead/tr/td/h3/a/text()')[0].replace(',', '-')
            item['jobs'] = div.xpath('.//thead/tr[1]/td[2]/text()')[0].replace(',', '/')
            gender_age_education = div.xpath('.//thead/tr[2]/td[1]/text()')[0].split()
            try:
                item['gender'] = gender_age_education[0]
                if item['gender'] not in ('男', '女'):
                    item['gender'] = 'null'  # null for unknown
            except IndexError:
                item['gender'] = 'null'
            try:
                item['age'] = gender_age_education[1].strip('岁')
                try:
                    item['age'] = int(item['age'])
                except ValueError:
                    item['age'] = -1  # -1 for unknown
            except IndexError:
                item['age'] = -1
            item['code'] = code
            executives.append(item)
    # write to csv file
    file_directors_csv.writerows(executives)
```
<a name="kK2Df"></a>
### 董事会信息存储
在第二个步骤中，针对每个公司，提取后的信息都会写入一个 executive_prep.csv文件，格式如下：

| name | gender | age | code | jobs |
| --- | --- | --- | --- | --- |
| 朴明志 | 男 | 51 | 600007 | 董事⻓/董事 |
| 高燕 | 女 | 60 | 600007 | 执⾏董事 |
| 刘永政 | 男 | 50 | 600008 | 董事⻓/董事 |
| ··· | ··· | ··· | ··· | ··· |

<a name="GZOBO"></a>
##  股票行业以及概念信息
对于这部分信息，我们可以利⽤工具`Tushare`来获取，官网为http://tushare.org/ ，使用pip命令进行安装即可。下载完之后，在python里即可调用股票行业和概念信息。参考链接：[http://tushare.org/classifying.html#id2](http://tushare.org/classifying.html#id2)<br />![image.png](https://cdn.nlark.com/yuque/0/2024/png/38415538/1711098682649-ded64201-1b42-44fd-85e6-19cc1857c968.png#averageHue=%2389a451&clientId=u0273a9a4-01fd-4&from=paste&height=542&id=u65223f88&originHeight=677&originWidth=1844&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=155046&status=done&style=none&taskId=u38e0b367-a834-4a92-93ab-738675d0bc6&title=&width=1475.2)<br />通过以下的代码即可获得股票行业信息，并把返回的信息直接存储在`stock_industry_prep.csv`文件里。
```python
import tushare as ts
df = ts.get_industry_classified()
# TODO 保存到"stock_industry_prep.csv"
```
类似的，可以通过以下代码即可获得股票概念信息，并把它们存储在`stock_concept_prep.csv`文件里。
```python
df = ts.get_concept_classified()
# TODO 保存到“stock_concept_prep.csv”
```
<a name="Oszu5"></a>
# 设计知识图谱
设计一个这样的图谱：

-  创建“人”实体，这个人拥有姓名、性别、年龄 
-  创建“公司”实体，除了股票代码，还有股票名称 
-  创建“概念”实体，每个概念都有概念名 
-  创建“行业”实体，每个行业都有⾏业名 
-  给“公司”实体添加“ST”的标记，这个由LABEL来实现 
-  创建“人”和“公司”的关系，这个关系有董事长、执行董事等等 
-  创建“公司”和“概念”的关系 
-  创建“公司”和“行业”的关系 

![image.png](https://cdn.nlark.com/yuque/0/2024/png/38415538/1711098831654-5da3fb14-5d76-44f5-abb4-8f3980c5f9de.png#averageHue=%23e36d55&clientId=u0273a9a4-01fd-4&from=paste&height=472&id=u55988d0e&originHeight=590&originWidth=1381&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=69986&status=done&style=none&taskId=u9ebe968c-3277-4445-97c9-5477ed65b01&title=&width=1104.8)

使用Neo4j构建知识图谱
<a name="jJ1Mo"></a>
# 数据导入
<a name="Uwhp9"></a>
## 从CSV文件中导入
<a name="d7Ntm"></a>
### 创建可以导⼊Neo4j的csv文件
在数据获取阶段，我们已经分别生成了 `executive_prep.csv`, `stock_industry_prep.csv`, `stock_concept_prep.csv`，但这些文件不能直接导入到Neo4j数据库。所以需要做⼀些处理，并生成能够直接导入Neo4j的csv格式。<br />	我们需要生成这⼏个文件：`executive.csv`,  `stock.csv`, `concept.csv`, `industry.csv`, `executive_stock.csv`,`stock_industry.csv`, `stock_concept.csv`。对于格式的要求，请参考：[https://neo4j.com/docs/operations-manual/current/tutorial/import-tool/](https://neo4j.com/docs/operations-manual/current/tutorial/import-tool/)
```python
#以build_executive为例
def build_executive(executive_prep, executive_import):
    """Create an 'executive' file in csv format that can be imported into Neo4j.
    format -> person_id:ID,name,gender,age:int,:LABEL
    label -> Person
    整体思路：将executive_prep中的表格的每一行人员提取并赋予其知识图谱内的相应字段
    """
    count=0
    print('Writing to {} file...'.format(executive_import.split('/')[-1]))
    with open(executive_prep, 'r', encoding='utf-8') as file_prep, \
    open(executive_import, 'w', encoding='utf-8') as file_import:
        file_prep_csv = csv.reader(file_prep, delimiter=',')
        file_import_csv = csv.writer(file_import, delimiter=',')
        headers = ['person_id:ID(Executive)', 'name', 'gender', 'age:int', ':LABEL']
        file_import_csv.writerow(headers)
        for i, row in enumerate(file_prep_csv):
            if i == 0 or len(row) < 3:
                continue
            info = [row[0], row[1], row[2]]
            #这里不加count会导致在导入数据时md5加密的ID产生重复（有重复数据或重名现象）
            # generate md5 according to 'name' 'gender' ,'age','count'(Avoid id duplicates)
            info_id = get_md5('{},{},{},{}'.format(row[0], row[1], row[2],count))
            info.insert(0, info_id)
            info.append('Person')
            file_import_csv.writerow(info)
            count+=1
        print('- done.')
```
<a name="WqgV3"></a>
### 使用neo4j-admin将数据文件导入Neo4j
注意：首先需要将这些数据文件存放至Neo4j实例安装的import目录下。<br />![image.png](https://cdn.nlark.com/yuque/0/2024/png/38415538/1711343711086-bd43a30f-accd-4fe1-b246-99bb35b23c4f.png#averageHue=%23fbfbfa&clientId=u5bca77cc-8d31-4&from=paste&height=408&id=uec243ea9&originHeight=510&originWidth=914&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=43471&status=done&style=none&taskId=u8a4f3f16-af7c-44ae-b0ce-7da0c0da7bc&title=&width=731.2)<br />根据官方提供的neo4j数据导入的方式，使用neo4j-admin命令进行导入。【详细可见：[https://neo4j.com/docs/getting-started/data-import/csv-import/#batch-importer](https://neo4j.com/docs/getting-started/data-import/csv-import/#batch-importer)】
```shell
neo4j-admin database import full 
>>    --nodes=Stock=import/stock.csv `
>>    --nodes=Industry=import/industry.csv `
>>    --nodes=Concept=import/concept.csv `
>>    --nodes=Executive=import/executive.csv `
>>    --relationships=ExeStock=import/executive_stock.csv `
>>    --relationships=StockIndustry=import/stock_industry.csv `
>>    --relationships=StockConCept=import/stock_concept.csv `
>>    --overwrite-destination `
>>    --ignore-extra-columns `
>>    --verbose
```
该命令行执行时会将import的数据导入到系统默认的neo4j数据库中，数据导入后,可在neo4j提供的浏览器界面中看到对应的知识图谱<br />![image.png](https://cdn.nlark.com/yuque/0/2024/png/38415538/1711354135890-c9361012-b6c1-4ebf-9660-7676ab8e3107.png#averageHue=%2387c58e&clientId=u5bca77cc-8d31-4&from=paste&height=709&id=u65ba8909&originHeight=886&originWidth=1907&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=448169&status=done&style=none&taskId=u13ab42a1-1884-4f83-bfba-21741f0748e&title=&width=1525.6)
<a name="EmpEy"></a>
# 知识图谱查询
通过已经构建的知识图谱，使用Cyper编写相关的语句对知识图谱进行查询。<br />(1) 有多少个公司目前是属于“ST”类型的?
```cypher
match (n:ST) return count(distinct(n))
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/38415538/1711354342062-3906ccc0-a441-43e1-8c37-44f8ff414c83.png#averageHue=%23b7e4da&clientId=u5bca77cc-8d31-4&from=paste&height=736&id=u28133cde&originHeight=920&originWidth=1816&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=176269&status=done&style=none&taskId=ue189ad69-6120-4ebf-8786-21aafc895f2&title=&width=1452.8)<br />（2）查询综合行业下的所有公司
```cypher
MATCH (n:Industry {name: '综合行业'})-[:industry_of]-(c:Company) 
RETURN n,c
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/38415538/1711355242755-c9f9ca17-af81-4233-ad04-688724cb16bf.png#averageHue=%23d0e4eb&clientId=u5bca77cc-8d31-4&from=paste&height=625&id=Dowf0&originHeight=781&originWidth=1855&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=216222&status=done&style=none&taskId=u176bcaab-a22a-4a63-bb20-c9dbb10f4ad&title=&width=1484)<br />（3）对于有锂电池概念的所有公司，独立董事中女性人员比例是多少？
```cypher
MATCH (m:Concept{name:'锂电池'})<-[:concept_of]-(n:Company)<-[:employ_of{jobs:'独立董事'}]-(p:Person{gender:'女'})
MATCH (m:Concept{name:'锂电池'})<-[:concept_of]-(n:Company)<-[:employ_of{jobs:'独立董事'}]-(p2:Person)
RETURN count(distinct(p))*1.0/count(distinct(p2))
```

![image.png](https://cdn.nlark.com/yuque/0/2024/png/38415538/1711355421382-dd1d1b06-f186-4976-bc71-5a7151f9b21c.png#averageHue=%23b2bfac&clientId=u5bca77cc-8d31-4&from=paste&height=340&id=uabb33adc&originHeight=425&originWidth=1457&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=69556&status=done&style=none&taskId=udb1db6aa-aebc-4d48-bf88-17fa471c937&title=&width=1165.6)
