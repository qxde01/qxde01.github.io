---
title: R之文本分类
date: 2013-10-08 23:32:15
tags:tags: [R,NLP,文本挖掘,文本分类]
categories: NLP
---
 [RTextTools](http://www.rtexttools.com/) 是一个关于文本分类的工具包，汇集了9种算法：
*   BAGGING( ipred: bagging)：bagging集成分类
*   BOOSTING (caTools: LogitBoost )：Logit Boosting 集成分类
*   GLMNET(glmnet:glmnet)：基于最大似然的广义线性回归
*   MAXENT(maxent:maxent)：最大熵模型
*   NNET(nnet:nnet)：神经网络
*   RF( randomForest: randomForest )：随机森林  
*   SLDA(ipred:slda)：scaled 线性判别分析
*   SVM(e1071:svm)：支持向量机
*   TREE (tree:tree)：递归分类树
<!--more-->

 [RTextTools](http://cran.r-project.org/web/packages/RTextTools/index.html) 提供非常简单的接口和清晰的流程使用这些算法创建一个文本分类模型。下面使用电视剧《 [龙门镖局](http://movie.douban.com/subject/11627047/) 》在豆瓣上的324篇长篇影评数据为例，介绍RTextTools的使用方法。使用 [Rdouban](https://github.com/qxde01/Rdouban) 可以非常方便的获取到豆瓣影评数据，使用 [Rwordseg](http://jliblog.com/app/rwordseg) 分词，分类标签为用户评分。324篇影评中，一星11篇，二星27篇，三星97篇，四星123篇，五星66篇，豆瓣综合评分7.4（截止2013-09-09）。本文主要参考  RTextTools作者的   《  [ RTextTools: A Supervised Learning Package for Text Classification ](http://74.125.128.160/url?sa=t&rct=j&q=RTextTools%3A%20A%20Supervised%20Learning%20Package%20for%20Text%20Classification&source=web&cd=1&cad=rja&ved=0CCwQFjAA&url=%68%74%74%70%3a%2f%2f%6a%6f%75%72%6e%61%6c%2e%72%2d%70%72%6f%6a%65%63%74%2e%6f%72%67%2f%61%72%63%68%69%76%65%2f%32%30%31%33%2d%31%2f%63%6f%6c%6c%69%6e%67%77%6f%6f%64%2d%6a%75%72%6b%61%2d%62%6f%79%64%73%74%75%6e%2d%65%74%61%6c%2e%70%64%66&ei=JyE3UoDAPMW1iAeml4D4BA&usg=AFQjCNFifONBfB0UkaItfz29s9xprBo_nQ&bvm=bv.52164340,d.dGI) 》一文。 

**1\. 创建文档-词项矩阵**

影评数据名称为`longmen`，移除所有的数字和字母，其中一篇影评为英文，被移除，仅处理323篇中文评论。其中用于生成`TermDocumentMatrix`的列为`longmen$word`，是影评分词后的词汇集合，可以用`create_matrix`函数生成`TermDocumentMatrix`，但是此函数对中文处理可能产生乱码，所有可以换一种处理方式：

```r
longmen <- longmen[ nchar(longmen $ word) > 0 ,]
myReader <- readTabular(mapping = list(Content =  "word" ))  
corpus <- Corpus(DataframeSource(longmen ),  
readerControl = list(reader = myReader ,language =  "zh_cn" ))  
mat <- TermDocumentMatrix(corpus , control = list(wordLengths = c( 2 , Inf)))  
mat <- weightTfIdf(mat , normalize = TRUE )  
mat <- t(removeSparseTerms(mat , sparse = 0.99 ))  
# mat2<- create_matrix(longmen$word,removeNumbers=TRUE, stemWords=FALSE,   
#                    weighting=weightTfIdf,removeSparseTerms=.99)
```
为了降低后面的计算量和内存开销， removeSparseTerms设置为0.99（不一定好） ，保留词汇2660个，实际上有词汇13428个，得到的数据是归一化后的TF-IDF矩阵。
  

**2\. 创建容器（** **Container** **）**  

使用函数`create_container`创建一个`container`，分类标签是评分`longmen$rating`，前250（约77.4%）行为训练集，剩余的为测试集。
```r
container <- create_container(mat ,longmen$rating,trainSize = 1 : 250 ,testSize = 251 : 323 , virgin = FALSE )
```
**3\. 训练模型**

使用`train_model`函数对`container`进行训练，选择其中7种算法，另外两种太吃内存了，破电脑受不了。`train_models`函数可以一次性选择所有的算法。
```r
SVM <- train_model(container, "SVM" )  
GLMNET <- train_model(container,"GLMNET" )  
MAXENT <- train_model(container , "MAXENT" )  
# SLDA <- train_model(container,"SLDA")  
# BAGGING <- train_model(container,"BAGGING")  
BOOSTING <- train_model(container,"BOOSTING" )  
RF <- train_model(container , "RF")  
NNET <- train_model(container , "NNET" )  
TREE <- train_model(container ,'TREE')
```

**4\. 分类**
对测试集进行分类预测。
```r
SVM_CLASSIFY <- classify_model(container ,  SVM)  
GLMNET_CLASSIFY <- classify_model(container ,  GLMNET)  
MAXENT_CLASSIFY <- classify_model(container ,  MAXENT)  
# SLDA_CLASSIFY <- classify_model(container,SLDA)  
# BAGGING_CLASSIFY <- classify_model(container,  BAGGING)  
BOOSTING_CLASSIFY <- classify_model(container ,  BOOSTING)  
RF_CLASSIFY <- classify_model(container ,  RF)  
NNET_CLASSIFY <- classify_model(container , NNET)  
TREE_CLASSIFY <- classify_model(container ,  TREE)
```

**5\. 分析**

结果解析是机器学习过程非常重要的一步。函数`create_analytics`帮助我们理解测试集的分类结果，使用`summary`函数返回四个结果：类标签（`label_summary`）、算法摘要（`algorithm_summary`）、文档摘要（`document_summary`）、集成摘要(`ensemble_summary`)。算法摘要给出了四个评估指标：[精确度(precision),召回率(recall)](http://en.wikipedia.org/wiki/Precision_and_recall),[准确率(accuracy](http://en.wikipedia.org/wiki/Accuracy_and_precision) ),[F-scores](http://en.wikipedia.org/wiki/F-measure)。

*   accuracy = 正确识别的个体总数/识别出的个体总数：反映了分类器统对整个样本的判定能力——能将正的判定为正，负的判定为负 ；
*   precision= 正确识别的个体总数/测试集中存在的个体总数：反映了被分类器判定的正例中真正的正例样本的比重；
*   recall = 正确识别的个体总数 / 测试集中存在的个体总数：反映了被正确判定的正例占总的正例的比重；
*   F值  =  precision\*recall\*2 / (precision+ recall)：F1-Measure，综合了precision和recall，平衡二者之间的矛盾，当F1值较高时结果比较理想。

注意；输出的结果出现NaN时，说明数据集太小，无法给出确定的估计，本文使用的数据集太小，结果很差。

 类标签摘要汇总了所有类别的一些统计量，包括：
*   **NUM\_MANUALLY\_CODED** ：每类包含文档的数量
*   **NUM\_CONSENSUS\_CODED** ： the number that were coded using the ensemble method 
*   **NUM\_PROBABILITY\_CODED** ： the number that were coded using the probability method
*   **PCT\_CONSENSUS\_CODED** and **PCT\_PROBABILITY\_CODED** ： the rate of over- or under-coding with each method 
*   **PCT\_CORRECTLY\_CODED_CONSENSUS** and **PCT\_CORRECTLY\_CODED_PROBABILITY** ：  the percentage that were correctly coded using either the ensemble method or the probability  method 

文档摘要给出了每一个算法对每一个文档的分类标签和对应的概率值（`doc_summary`）。
```r
analytics <- create_analytics(container ,cbind(SVM_CLASSIFY , BOOSTING_CLASSIFY ,
           RF_CLASSIFY , GLMNET_CLASSIFY,NNET_CLASSIFY , TREE_CLASSIFY , MAXENT_CLASSIFY))  
summary(analytics)  
topic_summary <- analytics@label_summary  
alg_summary <- analytics@algorithm_summary  
ens_summary <- analytics@ensemble_summary  
doc_summary <- analytics@document_summary  
create_ensembleSummary(analytics@document_summary)
```
** 6\. 集成分类一致性(Ensemble Agreement)**

使用集成分类的方法可以提高分类的准确性，通常采用投票的原则，即每一种算法做一次分类，如果多个算法都得到相同的分类结果，就认为这种结果更为可信，也就是少数服从多数的原则。文本分类采用至少四种分类算法的集成，能够达到不错的效果。函数 create_ensembleSummary产生的结果和上述的集成摘要结果相同，返回集成分类的召回率和覆盖度（coverage ）。覆盖度可以理解为满足召回率阈值的文档百分比。覆盖度随着召回率的增大而减小，通常90%的召回率是比较由于价值的结果，如果在不少于4种算法和不太低的覆盖度（个人认为>60%）情况下，这种结果是相当令人满意的。本文的数据集的结果如下，召回率都不令人满意。

n|n-ENSEMBLE COVERAGE|n-ENSEMBLE RECALL
---|---|---
n>= 1|1.00|0.29
n>= 2|1.00|0.29
n>= 3|1.00|0.29
n>= 4|0.84|0.33
n>= 5|0.59|0.35
n>= 6|0.25|0.33
n>= 7|0.10|0.29

**7\. 交叉验证(** **Cross Validation)**

可以使用 N-折交叉验证评估算法的准确性（Accuracy）和稳定性，以决定哪些算法可以参与到集成算法中。N-折交叉验证，初始采样分割成N个子样本，一个单独的子样本被保留作为验证模型的数据，其他N-1个样本用来训练。交叉验证重复 N 次，每个子样本验证一次，平均 N 次的结果或者使用其它结合方式，最终得到一个单一估测。这个方法的优势在于，同时重复运用随机产生的子样本进行训练和验证，每次的结果验证一次，10折交叉验证是最常用的。函数`cross_validate`可以完成此项操作。其中比较靠近1的数字表示在预测方面具有较高的可信度，比较靠近0的数字则表示预测的可信度较低。本文采用的数据集样本数实在太少，选择N=3。
```r
N = 3  
cross_SVM <- cross_validate(container , N , "SVM" )  
cross_GLMNET <- cross_validate(container , N , "GLMNET" )  
cross_MAXENT <- cross_validate(container , N , "MAXENT" )  
# cross_SLDA <- cross_validate(container,N,"SLDA")  
# cross_BAGGING <- cross_validate(container,N,"BAGGING")  
cross_BOOSTING <- cross_validate(container , N , "BOOSTING")
cross_RF <- cross_validate(container , N , "RF" )  
cross_NNET <- cross_validate(container , N , "NNET" )
cross_TREE <- cross_validate(container , N , "TREE" )
```
其中最好的两个模型是 MAXENT（0.85）和 GLMNET（0.65）.
文本分类（实际上文本挖掘都是如此）是非常消耗计算资源的应用，对于小数据根本没有意义，尤其对BAGGING 、BOOSTING等这类超级复杂度算法，小内存根本无法计算。代码和数据**[在这里](https://github.com/qxde01/myRproj/tree/master/longmen)**。
