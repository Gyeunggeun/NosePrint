# 강아지 비문 Detection 모델 개발

**이 경 근 | askwhy96@ajou.ac.kr**

담당 | Preprocessing, Backbone Modeling

### **Contents**

# Our Task

유실견을 찾아서 이 강아지가 해당 유실견이 맞는 지 파악하기 위해서 강아지 비문 비교를 이용하기로 결정.

**강아지 비문**이란, 강아지 코에 있는 고유한 패턴을 의미. 즉, 강아지마다 모두 다른 비문을 가짐.

따라서 우리가 만들어야 하는 모델은 다중 분류를 통해

1. 강아지 비문을 **분류**  
2. 비문 마다 각각 패턴의 고유한 **특징(feature) 추출** 
3. 추출한 feature들로 해당 강아지가 원하는 강아지와 일치하는지 **Detection**을 하는 모델

## 데이터셋의 개요

강아지 얼굴을 정면(약 10~30cm 거리)에서 촬영한 강아지 근접 사진

개체별 강아지 사진은 최소 4장 ~ 최대 10장

![a019_4.jpg](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c23aa949-f9d5-4de5-ae33-7bee5b378c3f/a019_4.jpg)

수집 방법은 직접 촬영(강아지 유치원 봉사활동) + 데이터 크롤링 + 지인들을 통해 받은 강아지 사진들

**강아지 숫자 총 50마리, 총 사진 376장**을 데이터셋으로 활용

# 모델 설계

## 이미지 데이터 전처리

먼저 각 이미지에 대한 라벨링을 진행

![화면 캡처 2023-06-11 033931.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/53d8c482-1387-4fcb-aa4d-cfc26bd5ab60/%ED%99%94%EB%A9%B4_%EC%BA%A1%EC%B2%98_2023-06-11_033931.png)

위와 같이 코 부분 근처 박스 형태의 라벨링 진행

라벨링한 사진에 대해 Resize 진행

```python
import cv2
import json

# JSON 파일을 불러옵니다
with open('a026_001.json', 'r') as f:
    data = json.load(f)

# 이미지 파일과 bounding box 좌표 출력
img = cv2.imread('a026_001.png')
bbox = data['bbox']  # 해당 내용은 json 파일마다 개별적으로 저장되어 있음

# 이미지와 bounding box의 크기를 바꿉니다
resized_img = cv2.resize(img, (224, 224))
scale_x = 224 / img.shape[1]
scale_y = 224 / img.shape[0]
resized_bbox = [int(scale_x * coord) if i % 2 == 0 else int(scale_y * coord) for i, coord in enumerate(bbox)]

# resize된 이미지를 저장하고 bounding box를 출력
cv2.imwrite('resized_a026_001.png', resized_img)
print("Resized bounding box: ", resized_bbox)

# JSON 파일에 resize된 바운딩 박스 정보를 업데이트
data['bbox'] = resized_bbox
with open('resized_a026_001.json', 'w') as f:
    json.dump(data, f)
```

- resize는 ResNet-152 모델에 호환되는 x=244, y=244 크기로 resize 진행

이후 사진 Masking을 통해 외부 배경의 변수 없이 비문만 모델링에 사용되도록 함

![화면 캡처 2023-06-11 045401.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9477094a-a70c-460b-b626-d533a9281469/%ED%99%94%EB%A9%B4_%EC%BA%A1%EC%B2%98_2023-06-11_045401.png)

최종적으로는 다음과 같이 강아지 코 이외 배경 부분은 검정색과 잘 대비되는 흰색으로 Masking

## 데이터 증강

ImageDataGenerator 패키지를 사용하여 train data에 한해 클래스 별 이미지 증강

```python
# ImageDataGenerator 적용 과정(이미지 증강 과정)

train_datagen = ImageDataGenerator(rescale = 1./255,
                                   rotation_range=40, # 회전제한 각도 40도
                                   zoom_range=0.2, # 확대 축소 20%
                                   width_shift_range=0.2, # 좌우이동 20%
                                   height_shift_range=0.2, # 상하이동 20%
                                   shear_range=0.2, # 반시계방햐의 각도
                                   horizontal_flip=True, # 좌우 반전 True
                                   fill_mode="nearest")

valid_datagen = ImageDataGenerator(rescale = 1./255)

train_generator = train_datagen.flow_from_dataframe(train_df,
                                                    x_col='Filepath',
                                                    y_col='Label',
                                                    target_size=(128, 128),
                                                    batch_size=75,
                                                    class_mode='categorical'
                                                    )

validation_generator = valid_datagen.flow_from_dataframe(strat_valid_df,
                                                     x_col='Filepath',
                                                     y_col='Label',
                                                     target_size=(128, 128),
                                                     batch_size=32,
                                                     class_mode='categorical',
                                                     shuffle=False
                                                     )
```

## 사용 모델 및 평가 지표

후보 Backbone 모델로 ResNet, VGG-19, Vision Transformer, YOLOv7 선정

이 중 feature 추출에 가장 우수한 성능을 보인 **ResNet-152** 모델 선정

최종 채택된 Backbone 모델의 학습 과정은 다음과 같음

- Pre-train
    
    ResNet-152v2 사용
    
- Fine-tuning
    - 사용된 하이퍼파라미터
        - batch_size = 75
        - warmup_epochs = 5
        - lr = 분수함수 형태로 변화하도록 learning rate scheduler 지정
            
            ```python
            def scheduler(epoch):
                if epoch < warmup_epochs:
                    return initial_learning_rate * epoch / warmup_epochs
                else:
                    return max(initial_learning_rate / (1 + 0.1 * (epoch - warmup_epochs)), lr_floor)  # 하한선 적용
            ```
            
        - Early_stopping 정의
            - patience = 3
        - Optimizer = ‘Adam’ 사용, 손실함수는 ‘Categorical Crossentropy’ 사용하여 각 클래스의 확률 분포 차이 최소화
- Validation
    - 사용된 하이퍼파라미터
        - batch_size = 32

## Loss 계산

샴 네트워크를 도입하여 서로 다른 두 이미지의 시각적 유사성을 측정

같은 네트워크 구조를 가진 두 개의 병렬 네트워크를 통해 이미지를 처리하며, 네트워크의 마지막에는 두 가지 유사성 점수 Contrastive loss와 Arcface loss 계산

각 Loss는 엔드-투-엔드 방식으로 최적화되며, 최종 목표 함수는 두 손실의 결합

**Contrastive loss:** 같은 클래스 이미지는 점수를 최대화, 다른 클래스는 최소화

- 같은 class의 사진(positive pair)은 euclidian 거리를 최대한 가깝도록 구성
- 다른 class의 사진(negative pair)는 euclidian 거리가 커지도록 설정

```latex
Lcon (i, x1, x2) = (1 − i) {max (0, m − d)}^2 + i * d^2,
```

- **`i`**는 두 이미지가 같은 클래스에 속하는지를 나타내는 수치
- **`d`**는 임베딩된 이미지 쌍 사이의 유클리드 거리
- **`m`**은 이미지 pair 사이의 유사성 점수를 얼마나 작게 만들 것인지를 결정하는 하이퍼파라미터

**Arcface loss**: 클래스 간 분리와 클래스 내 유사성을 동시에 최대화하는 방향으로 모델 학습

```python
ArcFace Loss = -log( e^s(cos(θyi + m)) / (Σ e^s(cos θj) )
```

- **`N`**과 **`n`**은 각각 배치 크기와 클래스 번호
- **`θyi`**는 ground truth 각도
- **`m`** 과 **`s`**는 각도와 특징을 의미하는 하이퍼파라미터

구한 두 loss는 다음과 같은 수식으로 최종 loss가 결정

```python
Ltotal = Lcon + 2 * Larc(anchor) + Larc(pair).
```

## 성능 평가

모델의 성능을 측정하기 위해, Accuracy와 F1-score를 사용

 Accuracy는 분류 모델에서 가장 자주 사용되는 지표

 F1-score는 오분류(FP, FN)에 대한 영향력을 낮춰주기 때문에 분류 모델 평가에 유용

# 모델 학습 결과

BackBone 모델에 따른 성능 비교

- 하이퍼파라미터는 위의 Fine-tuning에서 사용한 지표 동일하게 사용

| No. | Backbone Model | F1 | Accuracy |
| --- | --- | --- | --- |
| 1. | ResNet-152v2 | 0.943 | 0.951 |
| 2. | ResNet-50v2 | 0.920 | 0.932 |
| 3. | Vision Transformer | 0.911 | 0.901 |
| 4. | VGG-19 | 0.769 | 0.832 |

![색깔;;.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b0bc42d3-daf0-4aae-9d4c-97efd3179dc3/%EC%83%89%EA%B9%94.png)

Feature Extraction 모델의 하이퍼파라미터 변화에 따른 모델 성능 변화

| No. | Data Augmentation | Learning Rate Scheduler | F1 | Accuracy |
| --- | --- | --- | --- | --- |
| 1 | X | X | 0.515 | 0.528 |
| 2 | O | Step 형식의 epoch 적용, warmup:10 | 0.733 | 0.769 |
| 3 | O | Step 형식의 epoch 적용, warmup:5 | 0.810 | 0.802 |
| 4 | O | CosineDecay 적용, warmup:5 | 0.823 | 0.815 |
| 5 | O | CosineDecay 적용, initial : 1e^-2, decay step:30, warmup:3 | 0.863 | 0.856 |
| 6 | O | 분수함수 형태의 Decay
initial : 1e-3, floor : 1e-5, warmup:3 | 0.943 | 0.951 |

최종 모델의 Epoch에 따른 Accuracy와 Loss 변화
 
 
