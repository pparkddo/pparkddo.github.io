---
layout: post
title: 인터넷이 되지 않는 환경에서 Nexus Repository 운영하기
---

현장에서 운영되고 있는 설비에 관한 정보를 엑셀로 관리하고 있었습니다. 현장 설비이기 때문에 현장상황에 따라 수시로 이름과 설비관리번호가 바뀌었습니다. 설비운영을 위해 수량과 통계를 자주 냈고 이후 사업계획에 기반자료로 사용됐습니다. 이렇게 매우 중요한 자료였으나 모든 자료를 엑셀로 관리하다보니 과거 이력을 정리하기도 쉽지않고 여러 사람이 다른 파일을 가지고 수정하니 버전문제가 생기는 등 불편한 점이 많았습니다.  

![complicate-excel]({{ site.image.path | relative_url }}/2021-04-08-complicate-excel.png)
*복잡하게 관리되는 엑셀 (예시)*  

이런 문제들을 개선하고 업무를 효율성을 높이기 위해 사내에 업무지원 어플리케이션(백오피스?)를 개발하기로 했습니다. 여러 사람들이 자료를 관리하기에는 웹 어플리케이션이 좋다고 생각해서 사내에 `Java`, `Spring` 그리고 `Maven` 개발환경을 구성하려고 했습니다. 그런데 보안정책상 인트라넷(=사내망)과 인터넷망(=사외망)이 분리된 환경이라 사내망에서 의존성을 설치하기 위해 외부 저장소에 직접 접속할 수 없었습니다.  
`Maven` 을 사용해서 의존성을 설치하려하면 `Could not calculate build plan: Plugin (...) dependencies could not be resolved` 라는 에러메시지가 뜨기 일수였습니다. 이유는 위에서 말했던 대로 `Maven` 중앙 레포지토리에 접속을 할 수 없어 의존성을 다운로드 받지 못했기 때문입니다.

![maven-error]({{ site.image.path | relative_url }}/2021-04-08-maven-error.png)
*Maven 에서 의존성을 다운로드 받지못한 모습*

대안으로 인터넷이 되는 환경에서 직접 jar 파일들을 다운받고 그 파일들을 `Java Build Path` 에 로컬에서 일일히 넣어주는 방법도 생각했지만 의존성 관리도 힘들고 다른 사람들과 협업하기에 너무 복잡한 환경이 될 거 같았습니다.  
그래서 외부환경과 같이 `pom` 파일도 사용할 수 있고 사내망에서 개발하려는 다른 사람들에게 의존성에 대한 걱정을 덜어줄 수 있도록 제가 직접 `Nexus Repository` 를 운영하기로 했습니다.  

> 인터넷과 연결되지 않은 환경에서 Nexus Repository 를 운영해보자!

#### Nexus Repository 설치
`nexus repository` 는 `maven`, `npm` 등 여러가지 build tool 을 지원하는 무료 repository manager 입니다. 검색해보니 여러 곳에서 사내 저장소를 구축할 때 많이 사용하는 것 같아서 이 제품을 선택했습니다. 게다가 무료라이선스라 부담없이 사용할 수 있습니다. 다운로드는 아래 링크에 가서 하실 수 있고 안내에 따라 설치를 진행하면 됩니다.  

- [https://www.sonatype.com/products/repository-oss-download](https://www.sonatype.com/products/repository-oss-download)  

파일을 설치하고 압축을 푼 뒤 `nexus-<version>/` 형태를 가진 디렉토리에 들어갑니다. 디렉터리안에는 `bin/` 폴더가 있고 그 안에 `nexus.exe` 실행파일이 있습니다. 실행은 꼭 `nexus-<version>/` 폴더에서 아래 커맨드로 실행시켜줘야 합니다.

```cmd
bin/nexus.exe /run
```

처음에 `bin/` 폴더안에서 실행시키는데 정상적으로 실행이 안돼서 고생했습니다. 알고보니 관련 내용이 [공식 tutorial](https://guides.sonatype.com/repo3/quick-start-guides/proxying-maven-and-npm/#part-1-installing-and-starting-nexus-repository-manager-3) 에 있었습니다.  

#### Default Admin Password ?
Artifact 를 업로드하려면 업로드 권한을 가진 계정으로 로그인 되어있어야 합니다. 관리자 권한으로 접속하시려면 `../sonatype-work/nexus3/` 디렉토리 안 `admin.password` 파일에 있는 비밀번호를 치고 들어가면 됩니다! `nexus` 를 한 번 실행시키고 난 후에 `admin.password` 파일이 생성되니 주의하시기 바랍니다.  

![default-admin-password]({{ site.image.path | relative_url }}/2021-04-08-default-admin-password.jpg)
*Nexus 첫 실행시 admin.password 파일을 만듭니다*

### Upload Artifact
만약 `maven` 중앙 저장소를 `proxying` 할 수 있다면 중앙 저장소에 올라와있는 artifact 들은 사내망에 구축한 `nexus repository` 에 올릴 필요가 없습니다. 하지만 현재 인터넷망으로의 연결이 되지 않는 상황이라 직접 모든 artifact 들을 올려줘야합니다. 인터넷망에서 의존성을 설치하고 망 파일전송 시스템을 이용해 사내망으로 들고 온 다음 해당 의존성을 웹을 통해 업로드할 수 있습니다.  
업로드하는 방법은 관리자로 접속한 후 왼쪽 메뉴바에서 Upload 메뉴를 클릭하고 `file`, `pom.xml` 을 업로드 하면 나머지 `group id`, `artifact id` 등은 자동으로 입력됩니다. 그리고 하단의 Upload 버튼을 누르면 정상적으로 업로드 됩니다.  

![nexus-upload]({{ site.image.path | relative_url }}/2021-04-08-nexus-upload.png)
*Nexus repository 에 artifact 업로드하기*

### 마무리
이렇게 사내망에 `Nexus Repository` 를 설치하고 의존성을 업로드해서 운영해보았습니다. 그런데 간단히 `Spring-boot` 만 실행해도 약 800개가 넘는 의존성이 필요한데 이렇게 하나씩 업로드 할 수는 없습니다. 다음 글에서는 로컬에 있는 artifact 들을 어떻게 여러개를 업로드할 수 있을 지 알아보겠습니다!
