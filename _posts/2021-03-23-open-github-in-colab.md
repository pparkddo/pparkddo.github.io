---
layout: post
title: Google Colab 에서 Github 파일 열기
---

![google-colab-intro]({{ site.image.path | relative_url }}/2021-03-23-google-colab-intro.png)

`jupyter notebook` 환경을 이용한 `numpy` 와 `pandas` 강의를 사내에서 하게되었다.
코드를 보고 직접 실행을 하면서 배우는 것이 더 효과적이라 생각해서 `jupyter` 에서 미리 준비한 강의 소스코드 (`.ipynb`) 를 사용하여 실습을 진행하려 했다. 그런데 프로그래밍 언어를 처음 접하는 수강생들이 많아서 환경설정을 하는 데 어려움이 있을거라 생각이 문득 들었다.  


> 프로그래밍을 처음 접하는 분들도 손쉽게 `jupyter notebook` 환경설정과 소스코드 다운로드를 하는 법?


#### 1. `anaconda` 를 이용한 설치  

`anaconda` 를 통해 `jupyter` 를 설치하고 강의에 필요한 소스코드는 `.zip` 파일로 배포할 생각을 했다. 그런데 설치과정을 생각해보면 처음 하는 사람이 따라하면서 실패할 포인트가 너무 많았다.  

- 환경 변수 잘못 설정 또는 미설정
- 이미 설치되어있는 `anaconda` 를 재설치 (회사 공용 노트북을 사용하는 경우)  
- 설치시간이 너무 오래걸려 강의시간이 부족해짐
- 프로젝트 경로를 정확히 설정하여 소스코드를 설치해야 함

그래서 이 방법은 pass...


#### 2. `colab` 에서 `github` 파일을 열기

로컬 컴퓨터에 프로그램을 설치하지 않고 코드를 실행시킬 수 있는 방법을 찾다가 `colab` 을 찾게 되었다.  
`colab` 에서는 아래와 같이 다양한 방법으로 소스코드를 들고올 수 있다.

- 새 파일
- `google drive` 에서 불러오기
- `github` repository 에서 불러오기
- 로컬에서 업로드

이 중 `github` repository 에서 불러오기를 사용하면 아래 이미지와 같은 화면을 볼 수 있다.  

![colab-github-open]({{ site.image.path | relative_url }}/2021-03-23-colab-github-open.png)
*github repository 에서 불러오기*

입력칸에 `github` repo 의 url (여기서는 *https://github.com/pparkddo/nano-degree*) 을 넣게 되면 불러올 수 있는 `.ipynb` 파일들이 아래에 뜨게 된다. 리스트에서 불러올 파일을 클릭하면 해당 파일을 `colab` 에서 실행시킬 수 있게된다.

![colab-select-ipynb]({{ site.image.path | relative_url }}/2021-03-23-colab-select-ipynb.png)
*불러온 .ipynb 파일*


### 3. `colab` 에서 `github` 파일을 바로 열기

`colab` 을 이용하여 [1번](#1-anaconda-를-이용한-설치) 방식보다는 훨씬 간편하게 실행환경을 구성할 수 있게 되었다. 더 편하게 접근할 방식을 찾기 위해 `colab` 공식 안내문서를 보았는데 찾아보니 url 로 직접 접근할 수 있는 방법이 있었다.

> Loading Public Notebooks Directly from GitHub  
> Colab can load public github notebooks directly, with no required authorization step. 
>
> *\- [colab-github-demo.ipynb](https://colab.research.google.com/github/googlecolab/colabtools/blob/master/notebooks/colab-github-demo.ipynb)*

위 설명에 따르면 `github` 에 Public 으로 노트북 파일을 공개하면 인증절차도 필요없이 바로 소스코드를 불러올 수 있다고 한다. 예를 들어 `github` 에 아래 주소를 가진 파일이 있으면  

[https://github.com/pparkddo/nano-degree/blob/main/numpy.ipynb](https://github.com/pparkddo/nano-degree/blob/main/numpy.ipynb)

앞의 https://github.com/ 부분을 https://colab.research.google.com/github/ 으로 바꾸면 된다.  

[https://colab.research.google.com/github/pparkddo/nano-degree/blob/main/numpy.ipynb](https://colab.research.google.com/github/pparkddo/nano-degree/blob/main/numpy.ipynb)

위와 같이 `github` 의 코드를 불러서 `colab` 으로 열 수 있는 링크를 생성할 수 있다. 이 링크를 [github README](https://github.com/pparkddo/nano-degree#nano-degree) 페이지에 삽입해서 한 번의 클릭만으로 실행환경을 설정을 할 수 있게 만들었다.
