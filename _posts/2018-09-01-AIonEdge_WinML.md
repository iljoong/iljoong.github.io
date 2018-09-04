---
layout: post
title:  "AI on Edge with WinML"
date:   2018-09-01 00:00:00 +0900
categories: ai
---

많은 선진 IT 기업들은 이제 _"Cloud First, Mobile First"_ 보다는 _"Intelligent Cloud, Intelligent Edge"_ 로 이야기하고 있습니다. 

Cloud Computing의 다음 트랜드로 Edge Computing 또는 [Fog Computing](https://en.wikipedia.org/wiki/Fog_computing)이 떠오르고 있습니다. 데이터의 외부 반출 요구 및 Public Cloud 서비스의 네트워크 지연(latency)으로 Edge Computing은 어떻게 보면 필연적인 기술이며, 특히 AI 서비스 관련해서는 데이터 반출뿐만 아니라 네트워크 지연이 중요합니다. 아무리 고성능 GPU/FPGA 서비스가 Public Cloud에 제공된다고 해도 네트워크 지연에 따른 실시간 응답속도를 실현하기 매우 어렵기 때문입니다.

이번에는 AI on Azure를 좀더 확대하여 __AI on Edge__ 를 위한 Windows ML에 대해서 소개하도록 하겠습니다.

## Windows ML Overview

[Windows ML](https://docs.microsoft.com/en-us/windows/ai/)은 Apple의 [CoreML](https://developer.apple.com/documentation/coreml)와 Google의 [Tensorflow Lite](https://www.tensorflow.org/mobile/tflite/)와 동일한 모바일(또는 엣지) 디바이스를 위한 ML 라이브러리/프레임워크 입니다. CoreML과 Tensorflow Lite와 차이는 각 회사의 고유 ML/DL 모델 포맷이 아닌 [ONNX](https://onnx.ai)라는 오픈 포맷을 사용합니다.

![winml](https://docs.microsoft.com/en-us/windows/ai/images/winml-layers.png)

Windows ML은 Build 2018 행사에 앞서 올 봄에 처음 소개되었지만 아직까지는 정식 지원하지 않으며 2018 Fall Update (RS5)에서 정식 지원될 예정입니다. 현재는 Windows 10 Insider Program을 통해 최신 Preview 버전으로 업데이트를 해서 사용할 수 있습니다. 최근에 Windows ML의 프레임워크의 네임스페이스에서 Preview 딱지를 때어 버렸고, 구현 방식이 일부 수정된 정식 프레임워크가 출시되었습니다.

스마트폰 시장에서 Windows는 성공하지는 못했지만 일반 산업환경에서는 개발의 편의성으로 Windows가 아직까지 많이 사용되기 때문에 WinML은 기존 Windows 기반의 서비스에 AI 추가하고자 하는 기업에 큰 장점이 될 것으로 봅니다.

> 현재 WinRT 플랫폼만 지원하며, Win32와 같은 legacy 플랫폼는 지원하지 않음

## Getting Started

WinML을 사용하기 위해서는 아래의 스텝을 수행해야 합니다.

- 최신 Windows 10으로 업데이트 (17728 이상)- [Windows Insider](https://insider.windows.com)에 가입한 후 Windows 설정에서 Windows Insider를 활성화
- 최신 Visual Studio 2017 설치 (15.8.1) - 설치 파일을 다운 후 최신 버전으로 설치
- 최신 Windows 10 SDK Preview 설치 (17723 이상) - Windows Insider에서 최신 [SDK](https://www.microsoft.com/en-us/software-download/windowsinsiderpreviewSDK)다운 받아 설치

Preview로 업데이트하기 때문에 좀 불안하다고 싶은 분들은 Hyper-V 또는 Azure(MSDN Subscription 필요)에서 Windows 10을 설치하여 업데이트할 수 있습니다.

> 기존 Preview 샘플들은 현재 정상적으로 동작하지 않고 onnx 모델 업그레이드(v1.2) 및 WinML API 수정이 필요

최신 WinML Sample은 [WinML Github 사이트](https://github.com/Microsoft/Windows-Machine-Learning)에서 다운받아 테스트할 수 있습니다.


## Building WinML App

App을 만들기 위해서는 가장 먼저 학습된 AI 모델이 필요한데 쉽고 빠르게 학습 모델을 만들고자 한다면 customvision.ai를 사용할 수 있습니다. 참고로, customvision.ai 에서 mobile용 모델로 프로젝트를 생성하면 onnx 및 coreml 등의 모바일용으로 모델을 export할 수 있는 기능을 제공합니다.

또는, 기존 사용하는 프레임워크에서 onnx로 저장 또는 변환할 수 있습니다. 예를 들어 Tensoflow로 생성된 모델 파일(.pb)을 onnx로 변환하여 사용할 수 있습니다.

두 가지 유형의 AI 모델을 사용하여 WinML기반 App을 만들어 보았습니다.

- __CelebWinML__ : 이전 블로그에서 소개한 [Facetag](https://github.com/iljoong/facetag)를 _Customvision.ai_ 로 학습시킨 모델을 사용하여 유명인 얼굴 인식
- __HangulWinML__ : Github에 공개된 _Tensorflow_ 한글 문자 인식 예제, [tensorflow-hangul-recognition](https://github.com/IBM/tensorflow-hangul-recognition/),를 사용하여 한글 필기체 인식. App의 기반은 [MNIST WinML 샘플](https://github.com/Microsoft/Windows-Machine-Learning/tree/master/Samples/MNIST/UWP/cs)을 사용

![winml app](/images/aoe-winmlapp.png)

참고로, _HangulWinML_ 샘플에 포함된 onnx 모델은 4개의 폰트와 10,000 steps 학습한 모델로 약 60%의 정확도(검증 데이터 기준)를 보였기 때문에 인식률이 높지는 않습니다. 정확도를 높이고자 하면 좀더 많은 폰트와 학습을 수행하시기 바랍니다. 

## Development Issues

두 가지 방법으로 얻은 모델을 사용하여 WinML기반 App을 만들면서 경험했던 몇 가지 이슈들을 공유합니다.

### 1. ONNX bug by customvision.ai

현재 [Customvision.ai](https://customvision.ai/)에서 export된 onnx 파일에 버그가 있어 실행 시 `error: Required attribute 'consumed_inputs' is missing.` 오류가 발생합니다. 해결 방법은 coreml 파일로 export를 먼저 하고 onnx로 변환해서 사용할 수 있습니다. 해당 이슈는 이미 customvision.ai 팀도 알고 있는 사항으로 업데이트될 예정이라고 합니다. 

Coreml을 onnx로 변환하기 위해서는 [winmltools](https://docs.microsoft.com/en-us/windows/ai/convert-model-winmltools)을 이용하거나 [Visual Studio AI Tools](https://marketplace.visualstudio.com/items?itemName=ms-toolsai.vstoolsai-vs2017) 익스텐션를 이용하여 손쉽게 convert 할 수 있습니다. 대신, 각 프레임워크에 필요한 python library를 추가 설치해야 합니다. 자세한 스텝은 [coreml을 onnx로 covert하는 링크](https://elbruno.com/2018/07/10/winml-how-to-convert-tiny-yolov3-model-in-coreml-format-to-onnx-and-use-it-in-a-windows10-app/)를 참조하기 바랍니다.

> Python `coreml`은 Windows에서 바로 지원하지 않아 `coreml_windows`로 설치하거나 또는 앞서 소개한 참조링크 확인 

### 2. Right Tensorflow version

Tensorflow 모델을 onnx로 전환하기 위해서는 몇 가지 주의가 필요합니다. Tensorflow의 학습이 완료되면 2개의 model file(.pb)이 생성됩니다. `frozen_~.pb`와  `optimized_~.pb` 파일이 생성되는데 __frozen__ 버전을 사용해야 합니다. Tensorflow model을 변환하기 위해서는 앞서 소개한 coreml과 유사하게 tensorflow와 [tf2onnx](https://github.com/onnx/tensorflow-onnx/tree/master/tf2onnx) 라이브러가 설치되어야 합니다.

그리고, onnx 문서(현재 기준)에서도 Tensorflow는 1.8까지만 테스트되었다고 하는데 tensorflow는 __1.8__ 로 생성된 모델만 제대로 인식됩니다. 최신 10.x로 생성된 모델은 정상동작 하지 않았습니다.

참고로, Frozen 모델이 아닌 경우 input shape을 정상적으로 맵핑하지 못하며, Tensorflow 10.x를 사용한 모델을 convert 한 경우 WinML에서 input을 찾지 못해 `input:0 name is not found` 에러를 발생합니다. 

### 3. MLGEN bug

VS는 onnx 파일을 자동으로 감지하여 SDK에 포함된 [mlgen](https://docs.microsoft.com/en-us/windows/ai/mlgen)을 이용하여 C# Wrapper/인테페이스 클래스 파일을 생성해 줍니다. 하지만, Tensorflow에서 변환시킨 onnx 파일의 output node의 이름을 정상적으로 맵핑해주지 못해 `output00`으로 잘못 매핑된 노드 이름을 아래와 같이 `output:0`으로 수정해 주어야 합니다. VS를 새로 열 때 마다 자동 변환하려고 하는데 수정 이후에는 취소를 선택하시기 바랍니다.

```
output.output00 = result.Outputs["output:0"] as TensorFloat;
```

참고로, [netron](https://github.com/lutzroeder/netron)이라는 도구를 사용하면 모델 파일의 Graph를 시각화 할 수 있고, input/output의 이름 및 shape/type을 알 수 있습니다. 

### 4. WinML on Azure VM

Azure VM에서는 WinML App이 실행 시 간혹 아래와 같은 오류가 발생합니다.

```
System.NotImplementedException: 'The method or operation is not implemented.'
```

PC에서는 이 현상이 발생하지 않았습니다. PC 버전과 Azure VM의 Windows Insider 버전이 다르기는 하지만 Azure에서는 일부 라이브러리가 정상적으로 동작하지 않는 것 같습니다.

[https://github.com/Microsoft/Windows-Machine-Learning/issues/10](https://github.com/Microsoft/Windows-Machine-Learning/issues/10)

## Implementation

### Recognition Code for CelebWin

아래는 Input 이미지를 학습된 Model에 적용하여 Output 값을 얻는 C# 코드입니다. 자동 생성된 인터페이스 클래스를 이용하면 5줄의 코드로 간단히 구현됩니다. 

```
VideoFrame inputimage = VideoFrame.CreateWithSoftwareBitmap(bitmap);
celebInput.data = ImageFeatureValue.CreateFromVideoFrame(inputimage);
celebOutput = await modelGen.EvaluateAsync(celebInput);

var resultVector = celebOutput.classLabel.GetAsVectorView();
txtcelebName.Text = resultVector[0];
```

### Recognition Code for Hangul

한글 필기체 인식은 앞서 소개한 이미지 인식과 달리 Input/Output에 대한 추가 처리가 필요합니다.

### Input

모델이 [1, 4096] 모양의 `float` 타입 데이터로 받기 때문에 필기체 이미지를 동일한 크기(64x64) 및 gray 스케일 `float` 타입으로 전환해야 합니다. 추가적으로, WinML은 입력 변수로 `float` 타입을 직접 받지 못하기 때문에 `Tensorfloat` 타입으로 변환이 필요합니다. 

```
SoftwareBitmap bitgray8 = SoftwareBitmap.Convert(inputimage.SoftwareBitmap, BitmapPixelFormat.Gray8);
var buff = new byte[64 * 64];
bitgray8.CopyToBuffer(buff.AsBuffer());
var fbuff = new float[4096];
for (int i = 0; i < 4096; i++)
    fbuff[i] = (float)buff[i] / 255;
long[] shape = { 1, 4096 };
charInput.input00 = TensorFloat.CreateFromArray(shape, fbuff);
```

### Output

출력은 앞선 이미지 인식과 달리 그대로 쓸 수 없고 2350 개의 클래스( 글자) 별 확률을 확인하여 상위 첫 클래스로 인식합니다. 인식율이 높지 않기 때문에 상위 2~6위의 클래스를 추가로 표시합니다. List의 위치가 해당 클래스(글자)이기 때문에 Dictionary 타입으로 변환 후, 확율이 높은 순으로 정렬하여 다시 Array로 변환하여 사용해야 하는데 C#에서는 이런 복잡한 변환을 LINQ 쿼리를 통해 비교적 간단히 변환할 수 있습니다.

```
//Evaluate the model
charOuput = await charModel.EvaluateAsync(charInput);

//Convert output to datatype
IReadOnlyList<float> VectorImage = charOuput.output00.GetAsVectorView();
IList<float> ImageList = VectorImage.ToList();

//Display top results
var topPred = ImageList.Select((value, index) => new { index, value })
    .ToDictionary(pair => pair.index, pair => pair.value)
    .OrderByDescending(key => key.Value)
    .ToArray();

string topLabeltxt = "";
for (int i = 1; i < 6; i++)
{
    var item = topPred[i];
    topLabeltxt += $"{charLabel[item.Key]} ";
}

numberLabel.Text = charLabel[topPred[0].Key];
topLabel.Text = topLabeltxt;
```

두 샘플의 상세 구현 소스는 [Github 링크](https://github.com/iljoong/winml)를 참조하시길 바랍니다.

## Closing

AI on Edge를 구현하기 위한 여러 방법 중의 하나로 Windows ML을 소개했습니다. 10월에 출시될 Windows 10 RS5에 정식으로 탑재가 되면 앞으로 좀더 다양한 AI App들이 활성화될 것으로 예상합니다. AI App을 손쉽게 구현할 수 있지만 아직까지 Windows/WinRT 개발 환경이 Python 개발 환경과 비교하여 이미지 및 데이터를 선후 처리하는 부분이 용이하지 않았습니다. Windows ML이 활성화되기 위해서는 이부분의 개선이 필요해 보입니다.

//iljoong
