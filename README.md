# DLPP文档
该项目仅包含评审结果预测的算法部分，主要为```src```文件夹下的内容。项目已上传至[GitHub](https://github.com/ayonel/DLPP),可以fork到本地进行修改。已经删除了很多无用代码。

### 整体目录结构
>*__DLPP__*
>>*__front__*  主要为原型系统的前端代码，使用```vue.js```  
>>*__github__*  主要为原型系统的后端代码，使用```Spring Boot```  
>>*__JAVA__*  利用```WEKA```实现的gousios方法（**已废弃**）  
>>*__spider__* 利用```scrapy```实现的爬虫   
>>*__src__* 核心代码部分  
>>>*__anova__* 计算单方差检验时生成的数据文件（**不重要**）  
>>>*__ayonel__* PRPredict的核心代码  
>>>>*__feature_calculate__* 计算特征  
>>>>>*__1-cal_attributes.py__* 计算第一批属性,使用insert入库  
>>>>>*__1-cal_attributes_addition.py__* 计算另一批属性,使用update入库  
>>>>>*__1-cal_attributes_addition.py__* 计算相似度属性,使用update入库

>>>>*__feature_selection_data__* 存放特征选择后数据文件  
>>>>*__tuning_deprecate__* 对xgboost和randomforest进行调参，将调参结果入库（**可以视为废弃**）  
>>>>*__clf.py__* 核心分类文件
>>>>*__clf-parameter__* 特征选择文件
>>>>*__LoadData__* 数据加载文件

>>>*__database__* 封装数据库相关代码  
>>>*__eval__* 封装评估工具方法  
>>>*__gousios__* GPredict相关代码，包括特征计算以及算法实现，算法部分与*__clf.py__*类似  
>>>*__preprocess__* 爬虫将数据入库后，做的一些预处理工作，包括添加月份字段，对文本分词等  
>>>*__stat__* 一些零碎的统计代码和数据  （**不重要**）  
>>>*__constants.py__* 一些常量定义  
>>>*__utils.py__* 封装一些工具

从上可看出最终要的PRPredict的实现最重要的两个核心文件分别为```clf.py```、```LoadData.py```


### 整体流程
整体流程为，先利用spider中的代码爬取相关数据，并存入本地MongoDB，接下来做一些清洗验证及预处理工作；之后进行属性计算，并把计算好的属性也存入MongoDB， 
PRPredict运行时，先从MongoDB中获取pr数据，并拿到计算好的特征数据，再进行分类计算。爬虫及预处理工作已经完成，可以不动。

### clf方法工作流程
如果要跑PRPredict，只需要运行```clf.py```文件的main方法，并根据需求传入不同的参数，代码中已做了详细注释。main方法调用run_monthly方法，该方法为核心部分。  
run_monthly一运行，首先会调用LoadData.py中的load_data_monthly方法加载整个数据集，并返回data_dict（这是一个字典，键是组织名，值为每个组织数据集的迭代器）以及pullinfo_list_dict(这也是一个字典，键为组织名，值为每个组织的所有pr列表)  
之后对每个项目分别进行计算，并在计算完成后在console中打印结果。计算方法为，首先拿到第一轮的数据作为训练集，然后在迭代器中不断的生成下一批数据作为预测集，  
并利用训练好的模型进行预测，预测完成后将测试集加入训练集，并重新训练模型。

ayonel_numerical_attr 是输入的数值特征，ayonel_boolean_attr是输入的所有bool特征，如果需要更改特征，只需要更改这两个列表，重新跑即可。  

### 数据库及表定义
DLPP的MondoDB开在localhost的27017端口（默认端口），而决策者推荐（IREC）则开在localhost的27018端口。如果要开启DLPP的数据库，只需要以管理员权限运行cmd,  
之后输入```net start MongoDB```即可。如果要开启IREC的数据库，在cmd中输入```mongod --dbpath=F:/mongodb2/data --port=27018```即可，为简单起见，我对
IREC的数据开启方法封装在了IREC的database中的dbutil的主方法中，只需要运行这个方法即可。MongoDB的管理工具使用robomongo

DLPP的每个项目为一个数据库，库名为组织名称，每个库中有12个表，描述如下： 

+ ayonel 为计算出的特征，每个pr的所有特征作为一条记录   
+ commit 为爬取的commit信息，每个commit作为一条记录
+ commitfile 为爬取的commit的文件信息，每个文件作为一条记录，因此多条记录可能属于同一个commit
+ gousios 为计算出的gousios的特征，每个pr的所有特征作为一条记录
+ model 调参时保存的参数信息，每个轮次为一条记录，其中model字段代表是xgboost还是randomforest(*__可以视为废弃__*)
+ pullcomment 为爬取的pr的评论信息，每条评论作为一条记录，因此多条记录可能属于同一个pr
+ pullcommit 为爬取的pr的commit信息，每条commit作为一条记录，因此多条记录可能属于同一个pr
+ pullfile 为爬取的pr的文件信息，每个文件作为一条记录，因此多条记录可能属于同一个pr
+ pullinfo 为爬取的pr的基本信息，一条pr为一条记录
+ result (*__可以视为废弃__*)
+ reviewer 项目的决策者（核心成员），一个决策者为一条记录
+ stat 项目的一些统计(*__可以视为废弃__*)

### Tips 
#### mongo装饰器  
对于需要连接数据库的方法，我封装了mongo装饰器，只需要在需要的方法前加上@mongo注解即可，在方法定义时传入一个client变量，调用时可不用传。

#### 入库小心
对于要入库的代码，一定要检查仔细，确保入库的结果是正确的，否则会花费很大的精力修改
#### 字段对应
为了论文的写作规范，论文中的一些特征命名与库中不一致，主要是：  
is_reviewer-----> is_reviewer_commit  
file_changes-----> files_changes
history_author_pr_num----->history_commit_num
history_author_pass_pr_num----->history_pass_pr_num
history_author_passrate----->history_commit_passrate
history_author_review_time----->history_commit_review_time
#### gousios迁移
ayonel表中的一些属性是由gousios表中迁移过来的，迁移的属性为gousios的输入
#### IREC的项目组织类似，核心代码在ensembleresults文件夹下






  










