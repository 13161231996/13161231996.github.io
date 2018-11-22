---

layout:     post
title:      "Logistic算法"
subtitle:   " \"Logistic\""
date:       2018-5-01 12:00:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 机器学习

---
from sklearn.linear_model import LogisticRegression

import numpy as np

import random

#sigmoid 函数

def sigmoid(inX):

    return 1.0 / (1+np.exp(-inX))

#改进的随机梯度上升算法

def stocGradAscent1(dataMatrix,classLabels,numIter=150):

    m,n = np.shape(dataMatrix)

    weights = np.ones(n)

    for j in range(numIter):

        dataIndex = list(range(m))

        for i in range(m):

            alpha = 4/(1.0+j+i)+0.01

            randIndex = int(random.uniform(0,len(dataIndex)))

            h = sigmoid(sum(dataMatrix[randIndex]*weights))

            error = classLabels[randIndex]-h

            weights = weights + alpha * error * dataMatrix[randIndex]

            del(dataIndex[randIndex])

    return weights

#梯度上升算法

def gradAscent(dataMatIn, classLabels):

	dataMatrix = np.mat(dataMatIn)										#转换成numpy的mat

	labelMat = np.mat(classLabels).transpose()							#转换成numpy的mat,并进行转置

	m, n = np.shape(dataMatrix)											#返回dataMatrix的大小。m为行数,n为列数。

	alpha = 0.01														#移动步长,也就是学习速率,控制更新的幅度。

	maxCycles = 500														#最大迭代次数

	weights = np.ones((n,1))

	for k in range(maxCycles):

		h = sigmoid(dataMatrix * weights)								#梯度上升矢量化公式

		error = labelMat - h

		weights = weights + alpha * dataMatrix.transpose() * error

	return weights.getA()

#使用python写的logistic分类器做预测

def colicTest():

    frTrain = open('horseColiTraining.txt')

    frTest = open('horseColicTest.txt')

    trainingSet = [];trainingLabels = []

    for line in frTrain.readlines():

        currLine = line.strip().split('\t')

        lineArr = []

        for i in range(len(currLine)-1):

            lineArr.append(float(currLine[i]))

        trainingSet.append(float(currLine[i]))

        trainingLabels.append(float(currLine[-1]))

    trainWeights = stocGradAscent1(np.array(trainingSet),trainingLabels,500)

    errorCount = 0;numTestVec = 0.0

    for line in frTest.readlines():

        numTestVec += 1.0

        currLine = line.strip().split('\t')

        lineArr = []

        for i in range(len(currLine)-1):

            lineArr.append(float(currLine[i]))

        if int(classifyVector(np.array(lineArr),trainWeights))!=int(currLine[-1]):

            errorCount += 1

    errorRate = (float(errorCount)/numTestVec)*100

    print('测试集错误率为：%.2f%%' % errorRate)

#分类函数

def classifyVector(inX, weights):

    prob = sigmoid(sum(inX*weights))

    if prob > 0.5: return 1.0

    else: return 0.0

#使用Sklearn 构建logistic回归分类器

def colicSklearn():

    frTrain = open('horseColicTraining.txt')

    frTest = open('horseColicTest.txt')

    trainingSet = [];trainingLabels = []

    testSet = []; testLabels = []

    for line in frTrain.readlines():

        currLine = line.strip().split('\t')

        print(currLine)

        lineArr = []

        for i in range(len(currLine)-1):

            lineArr.append(float(currLine[i]))

        print('lineArr',lineArr)

        trainingSet.append(lineArr)

        trainingLabels.append(float(currLine[-1]))

        print('trainingLabels',trainingLabels)

    for line in frTest.readlines():

        currLine = line.strip().split('\t')

        lineArr = []

        for i in range(len(currLine)-1):

            lineArr.append(float(currLine[i]))

        testSet.append(lineArr)

        testLabels.append(float(currLine[-1]))

    classifier = LogisticRegression(solver='sag',max_iter=5000).fit(trainingSet,trainingLabels)

    test_accurcy = classifier.score(testSet,testLabels)*100

    print('正确率：%f%%' % test_accurcy)

if __name__ == '__main__':

    colicSklearn()

---


