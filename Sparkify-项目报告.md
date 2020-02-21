# Sparkify 项目 
**项目概述**
* 这是一个虚构的在线音乐流媒体网站Sparkify，数百万用户通过该网站的服务，使用免费套餐，或使用高级订阅模式，获得听取音乐更多权限。用户可以随时升级，降级或取消其服务。用户喜欢这项服务对于网站十分重要（涉及到用户的粘连，留存等）。每次用户与服务进行交互（即，降级服务，播放歌曲，注销，喜欢歌曲，添加歌单等）时，都会生成数据。所有这些数据都潜藏着有价值的信息，通过对于这些信息的挖掘分析，我们可以使用户的体验得到缇舍管并使网站得到更好的发展。我们感兴趣的目标是是否可以预测哪些用户有流失的风险。目标是在这些用户离开之前对其进行预测，以便用户可以在流失前获得优惠和奖励以减少用户流失率。

**项目目的**
* 对一个数字音乐服务公司的产品，建立一个模型来预测客户的流失情况

**数据集**

* 该数据集包含sparkify用户行为日志。日志包含有关用户的一些基本信息以及有关单个操作的信息。
* mini_sparkify_event_data.json(128MB)
* 本地操作系统压缩为bz2格式，mini_sparkify_event_data.json.bz2(6,45MB)

**项目技术**
spark：
* pyspark.sql用于对数据进行处理清洗，
* pysapk.ml用于特征工程处理以及搭建机器学习模型

python：
* pandas：spark转化为pandas dataframe格式进行数据探索
* numpy
* matplotlib进行可视化探索
 
**技术特点**
* 因为spark独特的lazy evaluation模式，需要将所有操作打包好再进行处理，对于转换，Spark将他们添加到计算的DAG中，只有在驱动程序请求一些数据时，DAG才会真正执行，这样做的优势在于，Spark有机会全面了解DAG之后，可以做出许多优化决策[1]。相当于我们告诉Spark最终感兴趣的答案是什么，它直接找到到达那里的最佳方法，节省时间和不必要的计算。[2]

**项目流程**：
* 加载清洗数据
* 探索性数据分析
* 特征工程
* 建模
* 模型选择并预测

**模型评估标准**
* 在机器学习中，我们做分类时，会用到一些指标来评判算法的优劣，最常见的就是准确率accuracy
```
Accuracy=Npre/Ntotal
```
* Npre为预测对的样本数，Ntotal为测试集的总样本数，但是准确率有时过于简单，不能全面反映算法的性能，这时就需要引入精准率*precision*，召回率*recall*，*F-1 Score*

---
混淆矩阵|Positive|Negative
--|:--:|--:
Positive|True Postive|False Positve
Negative|False Negative|True Negative
* 在混淆矩阵中：

    $precision=\frac{TP}{TP+FP}$

    $recall=\frac{TP}{TP+FN}$

    $F1=\frac{2precision*recall}{precision+recall}$
    
    $F1=\frac{TP+TN}{TP+FN+TN+FP}$
 
 * Precision:被我们算法选为positive的数据中，有多少是真的positive  
 * Recall：实际应为Positive的数据中，多少被我们选为了Positive
 * 准确率：Accuracy，我们正确分类了多少
 * F-1 Score：是precision和recall整合在一起的判定标准

    > recall 体现了分类模型H对正样本的识别能力，recall 越高，说明模型对正样本的识别能力越强，precision 体现了模型对负样本的区分能力，precision越高，说明模型对负样本的区分能力越强。F1-score 是两者的综合。F1-score 越高，说明分类模型越稳健 [3].

* 综上，如果我们希望模型既能准确识别正样本，又能有效区分负样本，所以需要权衡precision和recall，所以选择F-1 Score作为模型评分标准
 
## 2 加载和清洗数据 
* 创建一个SpakSession，读取文件(bz2)，查看文件数据情况

* 数据字典

```
     |-- artist: string (nullable = true)
     |-- auth: string (nullable = true)
     |-- firstName: string (nullable = true)
     |-- gender: string (nullable = true)
     |-- itemInSession: long (nullable = true)
     |-- lastName: string (nullable = true)
     |-- length: double (nullable = true)
     |-- level: string (nullable = true)
     |-- location: string (nullable = true)
     |-- method: string (nullable = true)
     |-- page: string (nullable = true)
     |-- registration: long (nullable = true)
     |-- sessionId: long (nullable = true)
     |-- song: string (nullable = true)
     |-- status: long (nullable = true)
     |-- ts: long (nullable = true)
     |-- userAgent: string (nullable = true)
     |-- userId: string (nullable = true)

```
* 在这些变量中，值得注意的是**userId**，代表了每一个用户；**gender**，**level**，**location**等类型变量，需要在后续选取是进行格式转换，因为机器学习模型只接收数值变量；**page**变量包含了用户的各种行为，也是一个重要的变量，可以分析用户行为与用户流失之间的种种关系；**registration**和**ts**变量作为时间相关变量，是不规范的一种格式，不过可以通过两个变量之间的差值获知用户使用应用的生命周期

### 缺失值
* 去除可能存在空值的列的，由于**userId**对应了每一个用户本身，需要确保**userId**没有空值，所以对**userId**，**sessionId**进行dropna处理
* 去除空值后查看发现**userId**列有许多空白，存在空字符串的情况，于是对其再进行处理，去除含空字符串的**userId**

```
    +------+---------+
    |userId|sessionId|
    +------+---------+
    |      |     1359|
    |      |      565|
    |      |      814|
    |      |     1053|
    |      |      268|
    |      |     1305|
    |      |     1446|
    |      |     1592|
    |      |      164|
    |      |     1183|
    +------+---------+

```
* 因为用户的行为存在许多差异，所以其余的变量存在空值是可以被允许的
* 查看数据集前后数量变化,数据集明显减少了
```

    去除空字符用户前数据集的数量： 286500
    去除空字符串用户后数据集的数量： 278154

```

## 3 探索性数据分析
### page

* 因为，用户基本上行为体现在**page**列，首先查看数据**page**列
```

    +-------------------------+
    |                     page|
    +-------------------------+
    |                   Cancel|
    |         Submit Downgrade|
    |              Thumbs Down|
    |                     Home|
    |                Downgrade|
    |              Roll Advert|
    |                   Logout|
    |            Save Settings|
    |Cancellation Confirmation|
    |                    About|
    |                 Settings|
    |          Add to Playlist|
    |               Add Friend|
    |                 NextSong|
    |                Thumbs Up|
    |                     Help|
    |                  Upgrade|
    |                    Error|
    |           Submit Upgrade|
    +-------------------------+
```
* 观察**page**列表可以看到一些有意思的行为，比如**Downgrade**，**Thumbs Up/Down**行为等，我们需要一种行为来判断用户是否流失。而如**Downgrade**，**Submit Downgrade**，**Cancel**等行为发生时用户仍是未流失的。
* 而**Cancellation Confirmation**来定定义用户流失是恰当的，首先这个时间次数很少，其次作为取消的确认，表明了用户的流失
### churn
* 这里我们使用**udf**自定义函数来将
**Cancellation Confirmation**转换为0和1，并使用窗口函数按照**userId**进行分组，给每一个用户标记上最终的event_churn状态（按分组取最大值表示最终状态）

```
    +------+-----+
    |userId|churn|
    +------+-----+
    |100010|    0|
    |200002|    0|
    |   125|    1|
    |   124|    0|
    |    51|    1|
    |     7|    0|
    |    15|    0|
    |    54|    1|
    |   155|    0|
    |100014|    1|
    +------+-----+
```
### time
* 从数据中我们可以看出属性‘ts’以及‘registration’为时间属性，我们需要将他们转换成timestamp的形式方便查看



* 使用datetime.datetime.fromtimestamp().strftime("%Y-%m-%d %H:%M:%s")，因为数据时间戳为13位,最后三位为0,要转换成10位，所以需要时间在原有基础上再除以1000，转换的自定义函数如下：
```
ts_transform = udf(lambda x:datetime.datetime.fromtimestamp(x/1000).strftime("%Y-%m-%d %H:%M:%s"))

```

### 可视化探索
#####  Churn on Gender 可视化
* 首先探索用户的个人属性，也就是用户是谁（性别，付费等级）
---
![churn on gender](https://github.com/EchoTrees/Sparkify_project/blob/master/img/churn_on_%20gender.png)

* 从图中可看出，男用户流失比女用户略多一点
---
####  Churn on Level 可视化
* 最初我使用**churn**来做的筛选，是不对的，因为**churn**标记的用户里，可能存在同一个用户有两个**level**的情况，并不能反映用户在流失时当下的**level**情况，所以应该选择<u>page=="Cancellation Confirmation"</u>来作为筛选的条件
---

![churn on level](https://github.com/EchoTrees/Sparkify_project/blob/master/img/churn_on_level.png)

* 从图形中可以观察得出曾使用过付费的用户反而比免费使用的用户流失的多，说明付费所提供的服务本身可能很不好，公司需要在这方面做出改进。
---



####  Churn on Ratio of Thumbs Up/Down 可视化
* **page**列包含了用户的种种行为，其中点赞和差评行为是和用户流失存在着一定的关系，对此我们进行了可视化分析

* 筛选出好评和差评的数据
* 对数据格式进行一定的调整，并计算出点赞和差评的平均值
* 因为单看点赞和差评的数量并不足以体现其与用户流失之间的关系，所以我们创建了一个新列**ratio_thumbs**用来存放点赞和差评行为之间的比值，将两个变量融合为一个变量
---
![churn on ratio of thumbs up/down ](https://github.com/EchoTrees/Sparkify_project/blob/master/img/Churn_on_Ratio_of_Thumbs_Up:Down.png)
* 从图中观察发现，点赞与差评之间的比值与用户流失与否有着一定的关系，比值越高的用户相对来说不容易流失。

---

#### Churn on Time-interval 可视化
* 用户使用时间的长短是否也影响了用户的流失情况，对此我们将时间变量做了一定的处理

```python
# 添加新的一列，计算用户使用时间区间
df_time = df_clean.select('userId','registration','ts','churn')\
    .withColumn('time_interval',(df_clean.ts-df_clean.registration)).toPandas()
```


```python
df_time = df_time.groupby(['userId','churn'])['time_interval'].max().reset_index()
# 对时间区间一列做转换，将其换算成日为单位
df_time.time_interval = df_time.time_interval.apply(lambda x:round(x/1000/3600/24))
```
---
![churn on time](https://github.com/EchoTrees/Sparkify_project/blob/master/img/churn_on_time.png)
* 从图形中可以看出，流失的用户相对于没有流失的用户总体使用应用服务的时间要少一些
---
## 4 特征工程
### 创建特征
* 选取一些可能与用户流失相关的特征
---
* 首先选择创建**label**，作为机器学习的目标标签
```

    +------+-----+
    |userId|label|
    +------+-----+
    |100010|    0|
    |200002|    0|
    |   125|    1|
    |   124|    0|
    |    51|    1|
    |     7|    0|
    |    15|    0|
    |    54|    1|
    |   155|    0|
    |100014|    1|
    +------+-----+
```
---
* 创建用户总共听过的歌曲数量特征
```
    +-------+------------------+-----------------+
    |summary|            userId|      total_songs|
    +-------+------------------+-----------------+
    |  count|               225|              225|
    |   mean|65391.013333333336|          1236.24|
    | stddev|105396.47791907164|1329.531716432519|
    |    min|                10|                6|
    |    max|                99|             9632|
    +-------+------------------+-----------------+
    
```
---
*  创建用户性别特征，由于性别特征为字符串格式，需要做一个转换,使用方法StringIndexer
```
# index_gender = StringIndexer(inputCol='gender', outputCol='gender_index')
# df_clean = index_gender.fit(df_clean).transform(df)
```


```python
# 创建性别特征
# f_2 = df_clean.select('userId','gender')\
#     .dropDuplicates()\
#     .replace(['M','F'],['0','1'],'gender')\
#     .select('userId',col('gender').cast('int'))
```


```python
#df_clean.select('userId','gender_index').dropDuplicates().show()
#在这个操作中出现了一个问题，就是无法去重
```


```python
# 尝试先去重，在做转换
df_gender = df_clean.dropDuplicates(['userId','gender'])
index_gender = StringIndexer(inputCol='gender', outputCol='gender_index')
df_gender = index_gender.fit(df_gender).transform(df_gender)
```


```python
f_2 = df_gender.select('userId',col('gender_index').cast('int'))
#转换成功，发现StringIndexer方法应该是根据列表内变量顺序来进行编码的
f_2.show(10)
```

    +------+------------+
    |userId|gender_index|
    +------+------------+
    |    44|           1|
    |    46|           1|
    |    41|           1|
    |    72|           1|
    |300023|           1|
    |    39|           1|
    |100010|           1|
    |    40|           1|
    |    94|           1|
    |    35|           1|
    +------+------------+
    only showing top 10 rows
    



```python
f_2.describe().show()
```

    +-------+------------------+-------------------+
    |summary|            userId|       gender_index|
    +-------+------------------+-------------------+
    |  count|               225|                225|
    |   mean|65391.013333333336| 0.4622222222222222|
    | stddev|105396.47791907165|0.49968243883744773|
    |    min|                10|                  0|
    |    max|                99|                  1|
    +-------+------------------+-------------------+
    


---
* 时间区间特征,方法同前文一样

```

    +-------+------------------+-------------------+
    |summary|            userId|        time_period|
    +-------+------------------+-------------------+
    |  count|               225|                225|
    |   mean|65391.013333333336|  79.84568348765428|
    | stddev|105396.47791907164|  37.66147001861255|
    |    min|                10|0.31372685185185184|
    |    max|                99|  256.3776736111111|
    +-------+------------------+-------------------+
    
```
---
* 由于**page**记录了用户的行为，从**page**中选取一些行为作为特征变量

* 特征创建 Thumbs Up/Down
```

    +-------+------------------+-----------------+
    |summary|            userId|    num_thumbs_up|
    +-------+------------------+-----------------+
    |  count|               220|              220|
    |   mean| 66420.27727272727|            57.05|
    | stddev|106196.51156121881|65.67028650524044|
    |    min|                10|                1|
    |    max|                99|              437|
    +-------+------------------+-----------------+
    
    +-------+------------------+-----------------+
    |summary|            userId|    num_thumbs_up|
    +-------+------------------+-----------------+
    |  count|               220|              220|
    |   mean| 66420.27727272727|            57.05|
    | stddev|106196.51156121881|65.67028650524044|
    |    min|                10|                1|
    |    max|                99|              437|
    +-------+------------------+-----------------+
    
```    
---
* 播放列表特征
```

    +-------+------------------+-----------------+
    |summary|            userId|  num_to_playlist|
    +-------+------------------+-----------------+
    |  count|               215|              215|
    |   mean| 66103.63720930232|30.35348837209302|
    | stddev|106360.47999565038| 32.8520568555997|
    |    min|                10|                1|
    |    max|                99|              240|
    +-------+------------------+-----------------+
    
```
---
* 艺术家收听数量特征
```

    +-------+------------------+-----------------+
    |summary|            userId|       num_artist|
    +-------+------------------+-----------------+
    |  count|               225|              225|
    |   mean|65391.013333333336|697.3733333333333|
    | stddev|105396.47791907164| 603.956976648881|
    |    min|                10|                4|
    |    max|                99|             3545|
    +-------+------------------+-----------------+
    
```
---
* 用户账户等级特征，这里涉及到同一用户存在多个等级的情况，需要对此进行处理
* 首先将level转换成int类型
```
f_8 = f_8.replace(['paid','free'],['1','0'],'level')\
    .select('userId',col('level').cast('int'))
```
* 然后将level其保留最终状态(paid->1)
```
window_level = Window.partitionBy('userId')
f_8 = f_8.withColumn('level',max('level')\
.over(window_level)).dropDuplicates()
```
```

    +-------+------------------+-------------------+
    |summary|            userId|              level|
    +-------+------------------+-------------------+
    |  count|               225|                225|
    |   mean|65391.013333333336| 0.7333333333333333|
    | stddev|105396.47791907164|0.44320263021395906|
    |    min|                10|                  0|
    |    max|                99|                  1|
    +-------+------------------+-------------------+
```    
### 特征与标签聚合
* 各项特征数据量不同，按照outer方式聚合，并填充缺失值为0
```
data_final = f_1.join(f_2,'userId','outer')\
    .join(f_3,'userId','outer')\
    .join(f_4,'userId','outer')\
    .join(f_5,'userId','outer')\
    .join(f_6,'userId','outer')\
    .join(f_7,'userId','outer')\
    .join(f_8,'userId','outer')\
    .join(label,'userId','outer')\
    .drop('userId')\
    .fillna(0)
```
* 以上是特征以及标签聚合的数据集

###  数据拆分转换
* 首先需要向量化特征,使用VectorAssembler方法来转换，非常方便
* 选取特征列,除**label**列以外特征进行向量化
* 接下来对特征进行缩放，使用Normalizer进行归一化
---
```
    +--------------------+
    |            features|
    +--------------------+
    |[0.82625192606090...|
    |[0.80614057312604...|
    |(8,[0,2,6],[0.151...|
    |[0.90652788346460...|
    |[0.87083903311466...|
    |[0.78108089108527...|
    |[0.86718673945040...|
    |[0.89031723474802...|
    |[0.83990033207259...|
    |[0.77879815013530...|
    |[0.86997047097193...|
    |[0.81704689880398...|
    |[0.86480089369949...|
    |[0.83845124110136...|
    |[0.87891040991532...|
    |[0.90372770617133...|
    |[0.82527601111947...|
    |[0.89309744501302...|
    |[0.83836080089797...|
    |[0.80532586682396...|
    +--------------------+
    only showing top 20 rows
```    


* 至此特征与目标标签准备完成

## 5 建模

### 拆分训练集，验证集和测试集


```python
train, validation, test = data_final.randomSplit([0.6,0.2,0.2],seed = 42)
```

### 选择模型建模
* 因为是分类问题，目标0或1，所以选择分类算法
* 首先选择一个算法，比如LogisticRegression，RandomForestClassifier等
* 加载数据得到模型（训练集）
* 使用模型进行预测（验证集）
* 选择一个合适的评分方法，对结果进行评价；因为流失顾客数据集很小，选用 F1 score 作为优化指标。
```
（evaluator：MulticlassClassificationEvaluator),evaluator.evaluate(results,{evaluator.metricName:'f1'})

```

```
# 模型初始化
lr = LogisticRegression(maxIter=5)

# 设置评分
evaluator_f1 = MulticlassClassificationEvaluator(metricName='f1')

# 建立一个简单的参数网格
paramGrid = ParamGridBuilder().build()

crossval_lr = CrossValidator(estimator = lr,
                             estimatorParamMaps = paramGrid,
                             evaluator = evaluator_f1, 
                             numFolds = 3)
```


```
# 训练模型
from time import time
start = time()
cvModel_lr = crossval_lr.fit(train)
end = time()
print('The Training process took {} seconds'.format(end-start))
```

    The Training process took 825.1287245750427 seconds



```
# 使用验证集进行预测
results_lr = cvModel_lr.transform(validation)
```

```
# 评估器进行评估
evaluator = MulticlassClassificationEvaluator(predictionCol = 'prediction')

print('Logistic Regression Metrics:')
print('\nAccuracy: {}'.format(evaluator.evaluate(results_lr,{evaluator.metricName:"accuracy"})))
print('\nF-1 Score: {}'.format(evaluator.evaluate(results_lr,{evaluator.metricName:"f1"})))
```

* Logistic Regression Metrics:
    
    Accuracy: 0.7142857142857143

    F-1 Score: 0.6711672473867596


#### SVM

* 步骤同上

* The Training process took 1067.491333246231 seconds

* SVM Metrics:

    Accuracy: 0.7755102040816326

    F-1 Score: 0.6774571897724607


#### Random Forest


* The Training process took 1074.8405141830444 seconds

* Random Forest Metrics:

    Accuracy: 0.8163265306122449

    F-1 Score: 0.7624711423030751


#### GBT

* The Training process took 1074.8405141830444 seconds

* GBT Metrics:

    Accuracy: 0.8163265306122449

    F-1 Score: 0.7624711423030751

### 模型总结
不同模型建模后测试结果如下：

* **Logistic Regression**：
    **准确率:0.7551020408163265**
    **F-1 Score:0.6672994779307071**
    **模型训练时间:825.1287245750427秒**
    
---
* **SVM**：
    **准确率:0.7755102040816326**
    **F-1 Score:0.6774571897724607**
    **模型训练时间:1067.491333246231秒**
    
---
* **Random Forest**：
    **准确率:0.8163265306122449**
    **F-1 Score:0.7624711423030751**
    **模型训练时间:1074.8405141830444秒**
    
---
* **GBT**：
    **准确率:0.7142857142857143**
    **F-1 Score:0.7142857142857142**
    **模型训练时间:1157.4184792041779秒**
    
---
* 综合对比下来，**Random Forest**模型的准确率以及F-1 Score均为最佳，虽然时间略慢于Logistic Regression，但介于数据量相对来说比较小的情况，我们倾向选择模型表现好的**Random Forest**作为预测模型，并在后续本地搭建环境中对其进行调参。

### 最优模型
* 综上我们选择**Random Forest**对测试集进行预测，并用F-1对模型结果进行评分


```python
last_results = cvModel_rf.transform(test)
```


```python
# 评估器进行评估
evaluator = MulticlassClassificationEvaluator(predictionCol = 'prediction')
print('Random Forest Metrics:')
print('\nAccuracy: {}'.format(evaluator.evaluate(last_results,{evaluator.metricName:"accuracy"})))
print('\nF-1 Score: {}'.format(evaluator.evaluate(last_results,{evaluator.metricName:"f1"})))
```

    Random Forest Metrics:
    
    Accuracy: 0.7352941176470589
    
    F-1 Score: 0.6479031804109204
#### 调参优化
* 在原有模型基础上，对参数**numTrees**和**maxDepth**进行参数网格调参[4]
```
# 建立一个简单的参数网格
paramGrid = ParamGridBuilder()\
    .addGrid(rf.numTrees,[10,20])\
    .addGrid(rf.maxDepth,[5,10])\
    .build()
```
* 得到模型评分矩阵
```
cvModel_rf.avgMetrics

[0.7026751131822128,
 0.7026751131822128,
 0.7068527171227145,
 0.7099845733111454]
```
* 根据矩阵结果，我们选择评分最优参数进行模型训练并对测试集进行预测，得到最终结果：

---
    Best Model Metrics:

    Accuracy: 0.7352941176470589

    F-1 Score: 0.7036625971143174
---
* 从结果可以看出，模型表现得到提升，最终**F-1 Score**为**0.7036625971143174**


## 6 总结
**在本次项目中，我们尝试建立一个模型来预测一个应用的客户流失情况**
* 在数据加载清洗中，我们首先在本地操作系统中将数据压缩成了bz2格式，然后对数据集进行加载，去除了可能存在空值的数据，去除了含有空字符串的用户
* 在探索性数据化分析中，我们首先定义了客户流失的行为特征，并将其通过（1，0）值对每个用户进行标记，并将时间相关特征转换成方便人阅读的格式
* 可视化分析中，我们探索了不同变量于用户流失之间的关系，我们发现：

    1. **男用户流失比女用户略多一点**
    2. **曾付费的用户反而比免费使用的用户流失的多，说明应用所提供的服务可能很不好**
    3. **点赞与差评之间的比值与用户流失与否有着一定的关系，比值越高的用户相对来说不容易流失**
    4. **流失的用户相对于没有流失的用户总体使用应用服务的时间要少一些**
    
* 我们对数据进行特征工程，包括特征筛选创建，聚合特征与标签，其中我们将level，gender特征转换为了int格式，方便模型训练，最后我们得出了7个特征一个目标标签的数据集；接着我们对特征进行向量化，并使用Nomalizer对特征进行归一化，得到最终数据集
* 我们将最终数据集拆分成训练集，验证集和测试集，并选取4个机器学习模型，对训练集进行训练
* 训练出的模型，通过验证集的预测，综合比较我们选择了**Random Forest**作为最佳的模型，并让该模型对测试集进行预测，并评价结果
* 最后观察结果发现**F-1 Score**比验证集预测有着较大的下滑，模型可能存在过拟合的问题，可以通过后续在本地环境中调参，尝试寻找更优的模型
* 还有一种可能是特征缩放器Nomalizer的效果可能没有别的缩放器好
* 另外在特征创建中，特征选择也可能影响到最后模型的表现

**模型调优**
* 在参数网格中，对参数**numTrees**和**maxDepth**进行调参分别设为
```
rf.numTrees,[10,20]
rf.maxDepth,[5,10]
```
* 得到**numTrees=20,maxDepth=10**为最优参数组合后，训练模型并进行预测，最终模型得到了优化：

    **F-1 Score**提升到了 **0.7036625971143174**

* 最后对项目的特征重要性进行了探索，发现**level**特征对于模型表现来说影响最大，**total_songs**相对来说重要性一般

**难点**
* 在做特征处理我尝试用StringIndex方法将类别变量进行转换，但在用这个方法转换的过程中发现，得出的结果并不能够去重，因为这个方法将变量的顺序作为转换的参考标准，在这个地方我卡住了挺久，解决方法有可以对变量先进行去重，再做StringIndex转换，或者对于只有两个值的变量如**gender**，可以使用独热编码对其进行转换，效果会更好。
* 在特征的选择方面，会影响模型的表现，如何选择合适数量的特征，是我需要提高的

**项目改善**
* 对于此项目，还有值得改善的地方，比如我们还可以可以对用户的降级行为（Downgrade）或升级行为（Upgrade）进行预测，对于在线音乐公司来说，用户降级也是值得关注的行为，因为这将导致服务订阅费用的减少，对于公司的利润有着直接的影响

**关于Spark**
* 通过Spark技术使我对大数据技术有了更多的了解，它通过减少操作的执行次数来提高系统的效率。并了解了它在现实世界中应用的潜力（很多依赖订阅（付费和未付费）的在线业务）。
* 其lazy evaluation的特性使得设计操作步骤十分从容，一个好厨师准备好食材就等下锅的感觉，优化流程，节省时间，使项目可管理性大大提高。

## 参考资料
---
[1]<https://stackoverflow.com/questions/38027877/spark-transformation-why-its-lazy-and-what-is-the-advantage>
[2]<https://techvidvan.com/tutorials/spark-lazy-evaluation/>
[3]<https://blog.csdn.net/matrix_space/article/details/50384518>
[4]<https://www.cnblogs.com/Allen-rg/p/9255286.html>