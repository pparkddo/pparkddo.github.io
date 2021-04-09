---
layout: post
title: Nexus Repository 에 다량의 Artifact 들을 업로드하기
---

[이전 글]({{ "/2021/04/08/nexus-repository-in-closed-network/" | relative_url }}) 에서 인터넷이 되지 않는 환경에 `Nexus Repository` 를 설치하고 artifact 를 업로드 해보았습니다.  
그런데 `Nexus` 에서 기본적으로 제공하는 웹 UI 에서는 한 번에 하나의 artifact 만 업로드할 수 있었습니다. `Spring boot` 같은 큰 프로젝트를 사용할때는 약 800 개가 넘는 의존성을 설치해야합니다. 모든 의존성을 하나씩 업로드 할 수는 없는 노릇이라 다른 방법을 찾아보아야 했습니다.  

> Nexus Repository 의 다량의 Artifact 들을 업로드할 수 있는 방법을 찾아보자!

### 모든 의존성 다운받기
일단 프로젝트 빌드에 필요한 모든 의존성들을 다운받기 위해서는 아래 `goal` 을 사용해서 다운받아야합니다. 아래 명령어를 사용함으로써 plugins 을 포함한 모든 의존성들을 로컬 저장소(`{user.home}/.m2/repository`)에 다운받습니다.
```bash
mvn dependency:go-offline
```
이후 다운받은 모든 의존성들을 옮겨주기만 하면됩니다.

### File system 에 직접 artifact 들을 넣어주기
가장 원시적인 방법이지만 제대로 작동한다면 가장 편하게 해결할 수 있는 방법이기도 합니다. `nexus` 에서 내부적으로 artifact 들을 모아놓은 폴더에 필요한 의존성을 모두 붙여넣기하는 방법입니다. [공식 문서](https://help.sonatype.com/repomanager3/installation/directories#Directories-DataDirectory)에 따르면 `blobs/` 에 데이터가 있다고 합니다. 실제로 디렉터리 안을 보면 바이너리 형태로 데이터가 들어가있어 단순하게 파일들을 붙여넣는 것으로 데이터를 옮기는 것을 불가능해 보입니다. 그리고 만약 그게 가능하다고 해도 `nexus`가 데이터를 관리하는데 쓰는 `OrientDB` 의 `metadata` 등을 수정해야 하기 때문에 좋은 방법은 아닌 것 같습니다.  

[stackoverflow 의 아~주 예전 글](https://stackoverflow.com/questions/1410580/nexus-supports-mass-upload-of-artifacts#answer-1410676)을 보면 예전에는 `blob` 형태가 아니라서 직접 넣어주는 것도 가능했던 거 같습니다. 현재 `nexus3` 환경을 쓰는 우리는 댓글에 나와있다시피 불가능한 방법입니다.

### Nexus Repository API 사용하기
`nexus` 저장소의 documentation 을 찾아보니 `REST and Integration API` 라고 되어있는 걸 발견했습니다. 찾아보니 `Components API` 라고 저장소의 components 과 상호작용할 수 있는 API 를 제공하는 것을 알게되었습니다. 기본적인 `CRUD` 기능인 `List`, `Get`, `Upload`, `Delete` 등이 있었습니다. [문서](https://help.sonatype.com/repomanager3/rest-and-integration-api/components-api#ComponentsAPI-UploadComponent)를 확인해보면 아래와 같이 `linux` 의 `curl` 명령을 이용한 업로드 예제가 있는데 사내에 서버를 구축할 환경은 `Windows` 라서 `curl` 명령을 사용하려면 복잡했습니다. 
```bash
curl -v \
     -u admin:admin123 \
     -X POST 'http://localhost:8081/service/rest/v1/components?repository=maven-releases' \
     -F maven2.groupId=com.google.guava \
     -F maven2.artifactId=guava \
     -F maven2.version=24.0-jre \
     -F maven2.asset1=@guava-24.0-jre.jar \
     -F maven2.asset1.extension=jar
```
또 하나의 파일이 아니라 여러 개의 파일을 업로드하려면 스크립트를 작성해야하는 데 기능 확장성과 생산성이 좋은 `Python` 을 사용하기로 했습니다. `curl` 과 동일한 기능을 할 수 있는 `requests` 모듈을 이용해서 아래와 같은 스크립트를 작성했고 정상적으로 업로드되는 것을 확인했습니다.
```python
def upload_component(
    url: str,
    user_id: str,
    password: str,
    pom_name: str,
    jar_name: str = None) -> None:
    """
    의존성 중에는 pom 파일만 있는 경우가 있기때문에
    jar_name 은 선택인자로 받습니다.
    """
    files = {"maven2.asset1": open(pom_name, "rb")}
    data = {"maven2.asset1.extension": "pom"}
    if jar_name:
        files["maven2.asset2"] = open(jar_name, "rb")
        data["maven2.asset2.extension"] = "jar"
    response = requests.post(
        url,
        auth=(user_id, password),
        files=files,
        data=data
    )
    return response.text
```

### 업로드 성능 향상 (Feat. ThreadPoolExecutor)
[concurrent.futures 모듈](https://docs.python.org/3/library/concurrent.futures.html) 은 파이썬에서 고수준의 동시성 비동기 프로그래밍 인터페이스를 제공하는 모듈입니다. 여러 쓰레드(`ThreadPoolExecutor`)를 사용하거나 프로세스(`ProcessPoolExecutor`)를 사용해서 손쉽게 비동기 실행을 할 수 있게 합니다.  

업로드하는 데 걸리는 시간은 `CPU` 작업 시간이 아니라 `Network I/O` 시간이 주를 이루므로 `GIL` 의 영향을 받아도 상관없어서 `ThreadPoolExecutor` 로 처리하도록 했습니다. 업로드해야할 모든 파일들을 각각의 쓰레드에 배분하고 실행했습니다. 여러 쓰레드를 관리하는 `Main Thread` 에서는 배부된 모든 작업이 마무리되는 것을 기다려야 하기에 아래 처럼 `wait()` 함수에 `return_when=ALL_COMPLETED` 옵션을 줘서 모든 작업이 끝날때 까지 프로그램을 종료하지 않도록 하였습니다.

```python
with ThreadPoolExecutor() as executor:
    futures = [
        executor.submit(
            upload_component,
            url,
            user_id,
            password,
            pom_file,
            jar_file
        )
        for pom_file, jar_file in zip(pom_files, jar_files)
    ]
    wait(futures, return_when=ALL_COMPLETED)
```

얼마나 성능이 향상되었는지 확인하기 위해 `Spring boot starter` 구동에 필요한 `pom` 파일 853개, `jar` 파일 464개를 업로드 하는 데 걸리는 시간을 측정해보았습니다. 기존 방법대로 `for loop` 에서 하나씩 처리했을 때는 `28분 50초` 가 걸렸지만 8코어 환경에서 `ThreadPoolExecutor` 를 사용했을 경우 `46초` 밖에 소요되지 않았습니다. 약 `37.6배`나 빨라졌습니다!

### 마무리 
저장소를 운영하면서 앞으로 의존성을 업로드 하는 일이 잦아질거 같아서 따로 [실행 패키지를 만들어 배포](https://github.com/pparkddo/maven_uploader)했습니다. `argparse` 패키지를 이용해서 간단히 `cli` 를 만들기도 했습니다. 지금은 이미 업로드되어있는 artifact 들도 중복해서 업로드하는데 (물론 같은 artifact 라면 하나만 생기고 이후 업로드 요청은 무시됩니다) 업로드되어 있는 artifact 들은 제외하고 업로드하는 기능을 나중에 추가해볼 수 있을 거 같습니다.  
사실 `nexus` 에 다량의 artifact 들을 올리는 스크립트이기 때문에 `maven-uploader` 라는 패키지 이름보다 `nexus-bulk-uploader` 라는 이름이 더 어울리지 않나 싶습니다. 이후에 더 사용하면서 불편한 점이 생기면 그 때 이름을 변경하든지 해야겠습니다.  

하여튼 800 개가 넘는 의존성들을 일일히 웹 UI 를 통해 올리는 것보다 `REST API` 를 이용해서 훨씬 편하고 빠르게 올리게 되었습니다~
