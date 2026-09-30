# 🗓️ TIL: OpenCV 색공간(BGR·HSV)과 필터링·Morphology

### 핵심 키워드

#OpenCV #BGR #HSV #filter #Morphology

---

## 오늘 배운 것

- **강의자료**: 16강. OpenCV 기초 (입출력·HSV·필터링·모폴로지)

### 이미지 = NumPy 배열

- **기본 규약**
	- 컬러 이미지는 `(H, W, 3)` 형태의 `uint8` 배열, 값 범위 0~255
	- 원점은 **왼쪽 위**, y는 아래로 증가
	- 인덱싱은 `img[y, x]` (행 = 세로 먼저) ⭐
	- 단, `cv2.circle`, `cv2.rectangle` 같은 그리기 함수의 좌표 인자는 `(x, y)` 순서 ⭐
- **uint8 주의**
	- NumPy 덧셈은 overflow 시 wraparound: `np.uint8(250) + np.uint8(10) = 4` (260 mod 256)
	- 밝기 연산은 `cv2.add`(saturation, 255에서 멈춤) 또는 float 변환 후 처리

```python
img = cv2.imread("scene.jpg")
print(img.shape, img.dtype)       # (480, 640, 3) uint8
roi = img[100:300, 200:400]       # img[y1:y2, x1:x2]
```

### RGB vs BGR

- **채널 순서만 다르고 담긴 정보는 같음**
	- RGB: PNG·JPG 등 이미지 포맷, Matplotlib에서 사용
	- BGR: OpenCV 내부 처리 순서 (기술적 이유보다 초기 관례에서 유래)
- **순서를 혼동하면** R과 B가 뒤바뀜 → 파란 물체를 빨강으로 검출. G는 가운데라 그대로 ⭐
	- Matplotlib으로 표시할 때 `img[:, :, ::-1]`로 뒤집어야 색이 맞음

| 구분 | 이름 | 혼합 방식 | 용도 |
| --- | --- | --- | --- |
| RGB | 빛의 삼원색 | 가산 혼합 | 디스플레이, 카메라 센서 출력 |
| CMY(K) | 색의 삼원색 | 감산 혼합 | 인쇄 |

- ⭐ 색 검출에 쓰는 변환은 CMYK가 아니라 **HSV**. OpenCV `cvtColor`에는 BGR↔CMYK 변환 코드가 없음

### 색 검출에 HSV를 쓰는 이유

- **BGR의 한계**
	- 밝기 정보가 세 채널에 섞여 있음 → 조명이 바뀌면 세 값이 함께 변해서 범위를 정하기 어려움
- **HSV**
	- "무슨 색인가(H)"와 "얼마나 밝은가(V)"를 분리 → 조명이 바뀌면 주로 V만 변하고 H는 거의 유지

| 채널             | 의미                  | 일반 범위  | OpenCV 범위       | 조명 변화 영향 |
| -------------- | ------------------- | ------ | --------------- | -------- |
| H (Hue)        | 색상                  | 0~360° | `0~179` (°/2) ⭐ | 작음       |
| S (Saturation) | 채도 (0이면 무채색: 흰·회·검) | 0~100% | `0~255`         | 중간       |
| V (Value)      | 명도 (0이면 검정)         | 0~100% | `0~255`         | 큼        |

- **범위 설정 기준**: H는 좁게, S·V는 넉넉하게
- **빨강은 두 구간** ⭐
	- H가 0과 179에서 이어지는 원형 척도 → `H ∈ [0,10]`과 `H ∈ [170,179]`를 or 연산으로 합쳐야 함

```python
hsv  = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
mask = cv2.inRange(hsv, (0, 120, 70), (10, 255, 255))   # 범위 안 255, 밖 0
```


### 필터링(블러)

- **원리**: 작은 커널로 이미지를 훑으며 계산하는 컨볼루션(저역통과) → 노이즈 억제 후 `inRange`
- 커널 크기는 홀수 `(5, 5)`, 클수록 효과가 강하지만 경계가 뭉개짐

| 필터   | 함수                 | 원리                       | 효과적인 노이즈                                       | 단점                    |
| ---- | ------------------ | ------------------------ | ---------------------------------------------- | --------------------- |
| 가우시안 | `cv2.GaussianBlur` | 중심에 가까울수록 가중치가 큰 가중 평균   | Gaussian Noise (센서 발열·센서 자체 노이즈로 생기는 자글자글한 잡음) | 경계가 흐려져 물체·배경 구분이 약해짐 |
| 미디언  | `cv2.medianBlur`   | 픽셀(3×3이면 9개)을 정렬해 중앙값 선택 | Salt & Pepper Noise (전송 오류로 픽셀이 0 또는 255로 튐)   | 가우시안 노이즈에는 상대적으로 약함   |

- 미디언은 평균을 내지 않고 실제 이웃 값 중 하나를 고름 → <u>튀는 값(0/255)이 결과에 섞이지 않고 경계 보존이 더 잘 됨</u>

```python
smooth = cv2.GaussianBlur(img, (5, 5), 0)
median = cv2.medianBlur(img, 5)
```

### Morphology — 이진 마스크 정리

- `inRange`로 만든 마스크의 점 노이즈와 구멍을 정리하는 연산

| 연산 | 동작 | 효과 |
| --- | --- | --- |
| 침식(erosion) `erode` | 흰 영역을 한 겹 깎음 | 점 노이즈 제거, 물체도 작아짐 |
| 팽창(dilation) `dilate` | 흰 영역을 한 겹 불림 | 구멍 메움, 물체도 커짐 |
| 열림(opening) `MORPH_OPEN` | 침식 → 팽창 | 바깥 점 노이즈 제거, 크기 유지 |
| 닫힘(closing) `MORPH_CLOSE` | 팽창 → 침식 | 안쪽 구멍 메움, 크기 유지 |

- 표준 순서: **열림 → 닫힘** ⭐

```python
kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (5, 5))
clean  = cv2.morphologyEx(mask,  cv2.MORPH_OPEN,  kernel)
clean  = cv2.morphologyEx(clean, cv2.MORPH_CLOSE, kernel)
```

### 전체 파이프라인

```
BGR 프레임 → GaussianBlur → BGR2HSV → inRange(빨강은 두 구간 OR) → open → close → 마스크 → 컨투어/바운딩 박스
```

---

## 알게 된 것

- OpenCV 배열 인덱싱은 `img[y, x]`, 그리기 함수 좌표는 `(x, y)` → 같은 코드 안에서 순서가 다름 ⭐
- Morphology 열림(opening)은 점 제거, 닫힘(closing)은 구멍 메움 - 둘 다 물체 크기는 유지


---

## 느낀점
- 오늘 우분투를 포맷하고 새 버전을 깔았다. 프로그램들을 다시 까느라 정신이 없었다..
- OpenCV는 복습하는 느낌으로다가 들었다. 그래도 어느정도 다 대학교 때 들어본 내용이라 good!