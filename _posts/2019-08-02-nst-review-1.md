---
layout: post
title:  'A Review of "Neural Style Transfer: A Review" - 1'
subtitle: "Introduction to NST and a brief history of style transfer"
date:   2019-08-02 04:31:13 +0900
comments: true
---

## Neural Style Transfer(NST)
![](/img/posts/nst/nst-intro.PNG){:width="90%" .center-image}
*출처: Image Style Transfer Using Convolutional Neural Networks, Gatys et al.*{:.caption}

학문이 예술과 결합하는 것은 즐거운 일이다. 쇠라의 점묘화에서 빛의 회절 원리를 찾아내거나, 음계를 파동의 공명 현상과 연결지어 이해하는 행위들은 과학에 생기를 불어넣어준다. 그 뿐만 아니라 이런 식의 접근은 새로운 학문을 접할 때 흥미를 갖게 해주기도 한다. 딥러닝에서 이런 역할을 하는 것을 꼽으라면 단연 NST를 예로 들 수 있다. 작년 겨울 cs231n 강의를 들으면서 가장 기억에 깊게 남았던 것이 NST로 만든 고흐 스타일의 그림일 정도로 매력있는 분야였다.

Style Transfer는  이미지의 Content는 유지하면서 Style을 바꾸는 방법을 말한다. 위 그림을 예로 들면, 원본 이미지(A)의 Content(주택, 강, 하늘)은 그대로 유지하면서 그 스타일을 다르게 하여 이미지를 생성하는 것이다(B: 윌리엄터너, C: 고흐, D: 뭉크). Style Transfer중 Neural Network를 이용하는 것을 Neural Style Transfer(NST)라고 부르며, 특히 CNN(Convolutional Neural Network)이 발견은 Style Transfer에 큰 발전을 가져왔다. 이 글을 포함한 앞으로 이어질 글들에서 2017년에 나온 Review Paper *Neural Style Transfer: A Review* 를 읽고 요약, 구현을 하며 Style Transfer에 대한 Greedy Study를 하고자 한다.

**[Review Paper Link](https://arxiv.org/abs/1705.04058)**

## Style Transfer Without Neural Networks
NST가 있기 이전에도 이와 관련된 연구는 20년 넘게 진행되어 왔다. 그 당시에는 NPR(non-photorealistic rendering)이라는 이름으로 불렸다. AR(Artistic Rendering), 그 중에서도 2D 이미지에 대한 AR을 뜻하는 IB-AR 중 NST가 등장하기 이전, 즉 Neural Network(Especially CNN)를 이용하지 않는 알고리즘들에 대해 알아본다.

"IB-AR techniques without CNN" 크게 다음 4가지 분야로 나눌 수 있다.

- Stroke-Base Rendering
- Region-Based Techniques
- Example-Based Rendering
- Image Processing and Filtering

### Stroke Based Rendering
Stroke-based rendering(SBR)은 화가가 그림을 그리듯이, 캔버스를 선으로 채워서 원본 이미지를 기반으로 한 Artistic한 이미지를 생성하는 방식이다. 선을 무엇으로 정하냐(brush strokes, tiles, and stipples, etc.)에 따라 다양한 스타일의 이미지를 생성할 수 있다. 사람이 그림을 그리는 방식과 가장 유사하기 때문에 이미 존재해왔던 스타일(유화, 수채화, 스케치 등)을 효과적으로 표현할 수 있다는 장점이 있으나, 표현할 수 있는 스타일이 그것들에만 국한된다는 단점이 있다.

![](/img/posts/nst/example-sbr.jpg){:width="90%" .center-image}
*출처: http://www.fordartworks.co.uk/*{:.caption}

### Region-Based Techniques
Region-Based Techniques는 원본 이미지을 특정 영역으로 나눈 다음(Segmentation), 각 영역마다 특정한 Stroke를 채워 넣어서 이미지를 생성하는 방식이다. 영역을 세분화 하여 각 영역마다 다른 Stroke Style을 적용하여 디테일을 잘 표현할 수 있다는 장점이 있지만, 아래 그림과 같이 각 영역이 대응되어야 잘 표현되기 때문에 다양한 스타일을 유연하게 표현할 수 없다는 단점이 있다.

![](/img/posts/nst/example-rbt.PNG){:width="90%" .center-image}
*출처: Region-based painting style transfer, Shugo et al.*{:.caption}

### Example-Based Rendering
Example-Based Rendering은 일종의 Supervised-Learning으로써, 이를 처음 제시한 Hertzmann의 논문 *Image Analogies(2001)* 에 삽입된 아래 그림으로 단번에 이해할 수 있다. Image-Artistic Image의 Pair로 이루어진 Training Data들을 통해 그 관계를 파악하여 다른 이미지에도 이 규칙을 적용한다. Image Analogies는 다양한 Artistic한 스타일에 대해서 효과적이지만 Training Data를 구축하는 것이 어렵고, High-level Feature에 대해 학습이 어렵다는 단점이 있다. 눈치가 빠르다면 알겠지만 Neural Style Transfer는 Example-Based Rendering에 속하며, 따라서 Example-Based Rendering은 Image Analogies와 NST 두 갈래로 나뉜다고 볼 수 있다.

![](/img/posts/nst/example-ebr.PNG){:width="90%" .center-image}
*출처: Image Analogies, Hertzmann et al.*{:.caption}

### Image Processing and Filtering
원본 이미지에 단순히 필터를 씌우는 등의 처리를 하여 이미지를 생성하는 방식이다. Blurring filter 등이 이에 속하며, 필터링 알고리즘들은 일반적으로 straightforward하기 때문에 구현이 쉽고 간단하다는 장점이 있지만, Style Diversitiy 측면에서 매우 제한되어있다는 단점이 있다.


## Summary
종합해보면 위에 소개된 Style Transfer Without Neural Network들은 각각의 장점들을 가지고 있으나 flexibility, style diversity, 그리고 effective image structure extraction 등에 대해서 한계점을 가지고 있다. 기존의 방식들에 대한 이러한 한계점들이 Neural Style Transfer(NST)가 탄생한 배경이 되었다.
