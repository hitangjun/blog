---
title: 机器学习纲要
date: 2016-10-20 17:21:20
description: 
categories:
- Machine Learing
tags:
- Machine Learing
toc: true
author: John Tang
comments:
original:
permalink: 
---
Welcome to [Machine Learing](http://www.manning.com/MachineLearninginAction)! (作者网站，相关源码资料可从这下载)

## 机器学习纲要：
### 监督学习
##### kNN算法 
##### 决策树 
...

<!-- more -->

### 机器学习流程
1、收集数据
2、准备数据
3、分析数据
4、训练算法
5、测试算法
6、使用算法

## Quick Start

### kNN算法 

``` python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

__author__ = 'John Tang'

from os import listdir
from numpy import *
import operator
import matplotlib
import matplotlib.pyplot as plt

def createDataSet():
    group = array([[1.0,1.1],[1.0,1.0],[0,0],[0,0.1]])
    labels = ['A','A','B','B']
    return group,labels

def file2matrix(filename):
    """读取数据文件"""
    fr = open(filename)
    arrayOLines = fr.readlines()
    numberOfLines = len(arrayOLines)
    returnMat = zeros((numberOfLines,3))
    # print returnMat
    classLabelVector = []
    index = 0
    for line in arrayOLines:
        line = line.strip()# 截取调所有的回车字符
        listFromLine = line.split('\t')
        returnMat[index,:] = listFromLine[0:3]
        classLabelVector.append(int(listFromLine[-1])) #-1表示列表中的最后一列元素
        index += 1
    return returnMat,classLabelVector

def classify0(inX,dataSet,labels,k):
    """
    kNN算法：
        1）计算已知类别数据集中的点与当前点直接的距离；
        2）按照距离递增顺序排序
        3）选取与当前点距离最小的K个点
        4）确定前K个点所在类别的出现频率
        5）返回前K个点出现频率最高的类别作为当前点的预测分类
    """
    # 距离计算
    dataSetSize = dataSet.shape[0] #shape矩阵的行数和列数，维度
    diffMat =  tile(inX,(dataSetSize,1)) - dataSet #tile 将数组inX复制为dataSetSize行1列的矩阵，然后矩阵减dataSet
    sqDiffMat = diffMat**2
    sqDistances = sqDiffMat.sum(axis=1) #axis=1 就是将一个矩阵的每一行向量相加
    distances = sqDistances**0.5
    sortedDistIndicies = distances.argsort() #返回的是数组值从小到大的索引值

    # 选择最小的K个点
    classCount={}
    for i in range(k):
        voteIlabel = labels[sortedDistIndicies[i]]
        classCount[voteIlabel] = classCount.get(voteIlabel,0) + 1
    #排序
    # operator.itemgetter(1)定义函数b，获取对象的第1个域的值；reverse=True列表元素将被倒序排列
    sortedClassCount = sorted(classCount.iteritems(),key=operator.itemgetter(1),reverse=True)
    return sortedClassCount[0][0]

def autoNorm(dataSet):
    """
    归一化特征值 newValue=(oldValue - min)/(max-min)
    降低不同数据项间由于取值范围不同对结果的影响，通过归一化特征值能够降低该影响
    """
    minVals = dataSet.min(0)
    maxVals = dataSet.max(0)
    ranges = maxVals - minVals
    normDataSet = zeros(shape(dataSet))
    m = dataSet.shape[0]
    normDataSet = dataSet - tile(minVals,(m,1)) # 将数据集减去每列的最小值，（tile函数将一个向量复制为M行1列的矩阵）
    normDataSet = normDataSet/tile(ranges,(m,1))# 将数据集除以每列的区间差值
    return normDataSet,ranges,minVals

def datingClassTest():
    hoRatio = 0.10
    datingDataMat,datingLabels = file2matrix('datingTestSet.txt')
    normMat, ranges, minVals = autoNorm(datingDataMat)
    m = normMat.shape[0]
    numTestVecs = int(m*hoRatio)
    errorCount = 0.0
    for i in range(numTestVecs):
        classifierResult = classify0(normMat[i,:],normMat[numTestVecs:m,:],datingLabels[numTestVecs:m],3)
        print "the classifier came back with:%d, the real answer is : %d" % (classifierResult, datingLabels[i])
        if(classifierResult != datingLabels[i]) :
            errorCount +=1.0
    print  "the total error rate is: %f" % (errorCount/float(numTestVecs))

def classifyPerson():
    resultList = ['not at all','in small doses','in large doses']
    percentTats = float(raw_input("percentage of time spent playing video games?"))
    ffMiles = float(raw_input("frequent flier miles earned per year?"))
    iceCream = float(raw_input("liters of ice cream consumed per year?"))

    # 读取数据并归一化特征值
    datingDataMat,datingLabels = file2matrix('datingTestSet2.txt')
    normMat, ranges, minVals = autoNorm(datingDataMat)

    #录入要预测的数据
    inArr = array([ffMiles, percentTats, iceCream])

    #对数据进行分类
    classifierResult = classify0((inArr-minVals)/ranges,normMat,datingLabels,3)
    print "You will probably like this persion:", resultList[classifierResult -1]

def img2vector(filename):
    returnVect = zeros((1,1024))
    fr = open(filename)
    for i in range(32):
        lineStr = fr.readline()
        for j in range(32):
            returnVect[0,21*i+j] = int(lineStr[j])
    return returnVect

def handwritingClassTest():
    hwLabels = []
    trainingFileList = listdir('trainingDigits')
    m = len(trainingFileList)
    trainingMat = zeros((m,1024))
    for i in range(m):
        fileNameStr = trainingFileList[i]
        fileStr = fileNameStr.split('.')[0]
        classNumStr = int(fileStr.split('_')[0])
        hwLabels.append(classNumStr)
        trainingMat[i,:] = img2vector('trainingDigits/%s' % fileNameStr)

    testFileList = listdir('testDigits')
    errorCount = 0.0
    mTest=len(testFileList)
    for i in range(mTest):
        fileNameStr = testFileList[i]
        fileStr = fileNameStr.split('.')[0]
        classNumStr = int(fileStr.split('_')[0])
        vectorUnderTest = img2vector('testDigits/%s' % fileNameStr)

        classifierResult = classify0(vectorUnderTest,trainingMat,hwLabels,3)
        print "the classifier came back with: %d, the real answer is: %d" % (classifierResult,classNumStr)
        if(classifierResult!=classNumStr):errorCount +=1.0
    print "\n the total number of errors is:%d" % errorCount
    print "\nthe total error rate is: %f" % (errorCount/float(mTest))


if __name__=='__main__':
    # group,labels = createDataSet()
    # print classify0([0,0],group,labels,3)

    # datingDataMat,datingLabels = file2matrix('datingTestSet.txt')
    # print datingDataMat
    # print datingLabels[0:20]
    #
    # fig = plt.figure()
    # ax = fig.add_subplot(111)
    # ax.scatter(datingDataMat[:,1],datingDataMat[:,2],15.0*array(datingLabels),15.0*array(datingLabels))
    # plt.show()

    # datingClassTest()

    # classifyPerson()

    handwritingClassTest()
```

