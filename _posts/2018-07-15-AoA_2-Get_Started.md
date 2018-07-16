---
layout: post
title:  "AI on Azure: Get Started!"
date:   2018-07-15 01:00:00 +0900
categories: ai
---

이번 블로그에서는 AI의 가장 핫한 분야인 _Deep Learning_ 을 Azure에서 빠르게 시작할 수 있는 내용을 소개합니다. 참고로, 일반적인 Azure 기반의 AI 서비스를 소비하는 내용이 보다는 AI를 구축하는 핵심 기술을 중심으로 소개하고자 합니다. 기초 이론이 중요하기는 하지만 먼저 준비해야 할 것들이 무엇인지 알아보도록 하겠습니다.

- Deep Learning Framework
- Tools
- Examples

## Deep Learning Framework

Deep Learning을 막 시작을 하려고 할 때 가장 먼저 고민이 되는 부분이 바로 Deep Learning 프레임워크(이하 프레임워크) 입니다. Deep Learning이 주목받기 시작하던 초기에는 [Caffe](http://caffe.berkeleyvision.org/)가 주로 쓰였지만 현재는 [Tensorflow](https://www.tensorflow.org/)가 가장 많이 사용되고 있습니다. Tensorflow는 빠른 오픈소스화 및 Google의 영향력으로 현재 커뮤니티 가장 활발하고 공개된 구현 소스가 가장 많지만, [Facebook/PyTorch](https://pytorch.org/) 및 [Microsoft/CNTK](https://www.microsoft.com/en-us/cognitive-toolkit/)도 프레임워크를 오픈소스화하여 빠르게 추격 중에 있어 지금은 마치 DL 프레임워크 춘추전국시대와 같습니다. 

Tensorflow가 현재 가장 많이 사용되지만 이후에 나온 프레임워크에 비해서 사용하기 좀 어렵다는 것이 일반적인 의견입니다. [Keras](https://keras.io/)가 바로 사용성 부분을 해결하기 위해서 나온 상위 API로 Tensoflow, Theano 및 CNTK 등을 백앤드 프레임워크로 사용합니다. 현재 논문 수집 사이트인 [arXiv](arxiv.org)의 인용/사용된 프레임워크로 Keras가 2위에 위치하고 있고 빠르게 성장하고 있습니다. 추가로, Keras의 장점은 아래의 링크를 참조하시길 바랍니다.

![Arxiv mentions](/images/aoa-arxiv-keras.png)

source: [https://cran.r-project.org/web/packages/keras/vignettes/why_use_keras.html](https://cran.r-project.org/web/packages/keras/vignettes/why_use_keras.html)

Keras는 개인적으로도 프레임워크가 훨씬 간결하고 사용이 편리하다고 봅니다. 최근 Tensorflow도 [Keras API를 native하게 지원](https://www.tensorflow.org/api_docs/python/tf/keras)하여 장점이 일부 희석되었지만, 그래도 성능이 더 좋은 백앤드를 선택할 수 있다는 것이 또 하나의 큰 장점입니다. 

- [HKBU에서 테스트한 각 프레임워크별 성능 벤치마킹 자료](https://www.comp.hkbu.edu.hk/~chxw/papers/ccbd_2016.pdf)
- [Keras의 백앤드 성능을 비교](http://minimaxir.com/2017/06/keras-cntk/)

참고로, 다양한 프레임워크의 모델 포맷을 표준화 해주는 [ONNX(Open Neural Network Exchange)](https://onnx.ai/)가 있어 앞으로 프레임워크 간 이동이 좀더 수월해질 수 있을 것 같습니다. Keras는 아직은 Native하게 지원하지 않고 별도의 도구([https://github.com/onnx/onnxmltools](https://github.com/onnx/onnxmltools))를 제공하고 있습니다.

## Tools

### Python

Deep Learning(이하 DL)의 기본 개발언어는 파이선(Python)입니다. 언어와 문법 자체는 상대적으로 어렵지 않지만, Machine Learning(이하 ML) 또는 DL의 생태계를 Python으로 학습하는 것이 힘든 부분이고 통계와 ML 배경지식이 없다면 더욱더 어렵다고 느낄 것 입니다.

- [Python version 2.7 vs 3.5/3.6](https://www.quora.com/Which-version-of-Python-2-or-3-is-better-for-a-data-science-beginner)
- [Anaconda Python](https://www.anaconda.com/)
- [Numpy](http://www.numpy.org/) & [Pandas](https://pandas.pydata.org/)
- [Scikit learn](http://scikit-learn.org/stable/)
- [Plot](https://matplotlib.org/)

참고로, Python 버전에 따른 환경이 달라서 초기에 "초보가 격는" 그런 고생했습니다. Data Scientist는 2.x 버전을 선호한다고 하지만 최근에는 대부분 3.5(3.6)이 대세인 것 같습니다. 아직까지는 3.6은 활성화되지 않은 것 같으며, Ubuntu 17.x 부터 공식 지원합니다. 일부 2.7로 작성된 오픈소스가 종종 있는데, 아래와 같이 가상 환경(Azure DSVM 예)을 설정하면 됩니다.

python 2.7 
```
$ source activate root
```

python 3.5
```
$ source activate py35
```  

Python이 처음인 분들은 아래의 링크를 참조하시기 바랍니다.

- [Python Basics at w3school](https://www.w3schools.com/python/)
- [Quick Python & Numpy tutorial](http://cs231n.github.io/python-numpy-tutorial/)
- [Python Data Science Handbook](https://jakevdp.github.io/PythonDataScienceHandbook/)
- [Useful Python references by Chris Albon](https://chrisalbon.com/)

### Jupyter Notebook

IDE Tool로는 DL의 Visual Studio라고 이야기 할 수 있는, [_Jupyter Notebook_](http://jupyter.org/)을 가장 많이 사용합니다. Azure에 생성한 원격 VM에서 UI 때문에 Windows VM을 고려할 수도 있지만, SSH 터널링을 사용하여 Ubuntu VM(DSVM)에 Jupyter Notebook을 손쉽게 연결하여 사용할 수 있습니다.

원격 VM(DSVM)에서 Jupyter Notebook 실행
```
jupyter notebook --no-browser

>> to login with a token:
>>        http://localhost:8888/?token=da979ee8253d1fb578c8c6b674ce7e420495018695ba4ab4
```

로컬 Windows 10 PC에서 SSH 터널링 연결 실행
```
ssh -nNT -L 8888:localhost:8888 <username>@<dsvm ip/hostname>
```

로컬 PC의 브라우저에서 token이 포함된 login url을 오픈하여 사용

![Jupyter Notebook](https://cdn-images-1.medium.com/max/1600/1*Bpvn80Jq6JL4b5ZYMq8pCw.png)

참고로, 원격 VM으로 접속하여 사용하는 경우에는 PC의 카메라 및 마우스 이벤트와 같은 기능을 사용할 수 없는 제약이 있습니다.

### Installing Frameworks & Tools

DL 프레임워크를 실행하기 위해서는 관련 여러 Tool들을 설치해야 합니다. CPU 또는 GPU VM에 따라 설치해야 할 드라이버, SW등이 많아 어려울 수 있고 또한 추가적인 프레임워크를 설치하는 것도 여간 귀찮은 작업입니다. Azure를 사용한다면, [Azure DSVM (Ubuntu 16.04)](https://docs.microsoft.com/en-us/azure/machine-learning/data-science-virtual-machine/overview) 사용을 적극 권장합니다. DSVM에는 왠만한 프레임워크 및 관련 Tool들과 다양한 샘플들이 모두 설치되어 있어 바로 사용 가능합니다.

클라우드가 아닌 개인 PC를 사용할 경우에는 [Azure ML Workbench](https://docs.microsoft.com/en-us/azure/machine-learning/service/quickstart-installation)를 권장합니다. PC에서 사용하지만, Azure 구독이 있어야만 사용 가능합니다. 

참고로, DSVM은 일반 CPU VM에도 사용할 수 있지만 GPU VM을 사용하는 것 역시 권장합니다. GPU VM이 비용적으로 부담이 되지만 CPU VM 대비 약 6~10배 성능(NC6의 CPU vs. GPU)이 높아 실사용에 무리가 없습니다. 간단한 샘플은 CPU VM에 가능하지만, 좀 복잡한 모델 또는 대량의 데이터를 처리할 경우 많은 시간이 정말로 많이 소요됩니다. (일부 샘플의 경우 GPU VM(K80)에서도 1시간 이상의 학습 시간이 소요)

설치없이 사용하고 싶다면 [Azure Notebook](https://notebooks.azure.com) 또는 [Kaggle Kernel](https://www.kaggle.com/)을 고려할 수 있습니다. 사용의 제약이 좀 있으나 무료라는 가장 큰 장점이 있습니다. Kaggle의 경우 최근 [GPU Kernel](https://www.kaggle.com/dansbecker/running-kaggle-kernels-with-a-gpu)도 제공하고 있습니다. 

## Starting Examples

AI/Deep Learning을 시작하는데 있어 먼저 대표적인 예제들을 테스트해보는 것을 추천하며, MNIST와 CIFAR10을 Keras로 먼저 실행해 보는 것을 권장합니다.

- __MNIST__: 필기 숫자를 인식하는 ML의 대표적인 예제로 50,000 개의 학습 데이터와 10,000의 테스트 데이터가 제공. Deep Learning이 급부상하기 이전의 초기 CNN으로 사용된 예제
    - dataset: [http://yann.lecun.com/exdb/mnist/](http://yann.lecun.com/exdb/mnist/)
    - keras example: [https://github.com/keras-team/keras/blob/master/examples/mnist_cnn.py](https://github.com/keras-team/keras/blob/master/examples/mnist_cnn.py)

- __CIFAR10__: 10개 클래스의 32x32 이미지를 이용한 Image Classification의 대표적인 예제
    - dataset: [https://www.cs.toronto.edu/~kriz/cifar.html](https://www.cs.toronto.edu/~kriz/cifar.html)
    - keras example: [https://github.com/keras-team/keras/blob/master/examples/cifar10_cnn.py](https://github.com/keras-team/keras/blob/master/examples/cifar10_cnn.py)

- __Dogs & Cats__: Kaggle에서 진행했던 개/고양이 이미지 식별 대회로 25,000개의 테스트 데이터 제공
    - dataset: [https://www.kaggle.com/c/dogs-vs-cats](https://www.kaggle.com/c/dogs-vs-cats)
    - keras example: [https://blog.keras.io/building-powerful-image-classification-models-using-very-little-data.html](https://blog.keras.io/building-powerful-image-classification-models-using-very-little-data.html)

제가 정리한 [AI on Azure Sample](https://github.com/iljoong/aoa-sample)를 참고하시기 바랍니다.

## Beyond Examples

예제로 충분히 학습되었다고 생각하면, Kaggle 경진대회를 시작해보는 것도 학습의 또다른 방법이고 실력을 키우기 위한 좋은 것 같습니다. __Kaggle__ 에서 상금이 걸린 경진대회도 있으나, 초보자들도 참가할 수 있는 경진대회들도 있으니 시도해 보시기 바랍니다.

저도 현재 진행중인 MNIST 예제와 동일한 ['Digit Recognizer'](https://www.kaggle.com/c/digit-recognizer) 참가했는데, 일부 참조와  튜닝으로 _0.99657_ prediction으로 상위 10%내(2017-07-15 현재)에 포함되었습니다.

// iljoong