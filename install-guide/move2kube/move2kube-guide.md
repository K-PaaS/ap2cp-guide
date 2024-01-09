### [Index](https://github.com/K-PaaS/guide) > [ap2cp-guide](https://github.com/K-PaaS/ap2cp-guide) > Move2Kube

## Table of Contents

1. [개요](#1)  
 1.1. [목적](#1.1)  
 1.2. [참고](#1.2)
2. [준비사항](#2)  
 2.1. [테스트 머신 스펙](#2.1)  
 2.2. [Prerequisite](#2.2)  
3. [move2kube](#3)  
 3.1. [cf 인스턴스에서 어플리케이션 배포](#3.1)  
 3.2. [배포중인 어플리케이션의 정보수집](#3.2)  
 3.3. [어플리케이션 변환 준비](#3.3)  
 3.4. [어플리케이션 변환](#3.4)    
  3.4.1. [transform 활용](#3.4.1)   
  3.4.2. [서비스 설정](#3.4.2)   
  3.4.3. [레지스트리 정보 입력](#3.4.3)   
  3.4.4. [클러스터 타입 선택](#3.4.4)   
  3.4.5. [외부 연결방식 선택](#3.4.5)   
  3.4.6. [최소 레플리카 갯수 지정](#3.4.6)   
  3.4.7. [컨테이너 레지스트리 계정 정보 입력](#3.4.7)  
  3.4.8. [CD파이프라인의 대상 네임스페이스 입력](#3.4.8)  
  3.4.9. [ingress 설정 값 지정](#3.4.9)  
  3.4.10. [복제할 git repository 설정값 입력](#3.4.10)  
  3.4.11. [ 이미지를 레지스트리로 푸시하기 위한 Docker config.json이 있는 기존 K8s secret 입력](#3.4.11)    
 3.5. [이미지 빌드](#3.5)  
 3.6. [이미지 푸시](#3.6)  
4. [Kubernetes 설정 및 배포](#4)  
 4.1. [공통 가이드](#4.1)
  


 # <div id='1'/>1. 개요
 ## <div id='1.1'/>1.1. 목적
 본 문서는 cf에 배포된 Application을 move2kube를 이용하여 Kubernetes에 배포한다.

 ## <div id='1.2'/>1.2. 참고 자료
 - Cloud Foundry Installation Document: [https://docs.cloudfoundry.org/cf-cli/install-go-cli.html](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html)
 - Docker Installation Document: [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)
 - Kubernetes Installation Document : [https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
 - move2kube Installation Document : [https://move2kube.konveyor.io/installation](https://move2kube.konveyor.io/installation)
 - move2kube guide Document [https://move2kube.konveyor.io/tutorials/cloud-foundry/](https://move2kube.konveyor.io/installation)

 # <div id='2'/>2. 준비사항
 ## <div id='2.1'/>2.1. 테스트 머신 스펙
 테스트를 진행하기 위해서는 8coreCPU와 16GB RAM 이상의 스펙이 필요하다.

 ## <div id='2.2'/>2.2. Prerequisite
 - Docker 설치 : ([1.2 참고자료](#1.2))
 - kubectl 설치 : ([1.2 참고자료](#1.2))
 - CF CLI 설치 : ([1.2 참고자료](#1.2))
 - move2kube 설치 : ([1.2 참고자료](#1.2))

# <div id='3'/>3. move2kube
## <div id='3.1'/>3.1. cf 인스턴스에서 어플리케이션 배포
- Application을 cf를 통해 배포한다.
```
$ cf login -a <YOUR CF API endpoint>
# spring-music Application을 build 하여 준비
$ cf push -p ./source/spring-music-1.0.jar spring-music
```
## <div id='3.2'>3.2. 배포중인 어플리케이션의 정보수집
- cf 인스턴스에서 실행 중인 애플리케이션에 대한 정보를 수집한다.
```
$ move2kube collect -a cf

INFO[0000] Begin collection                             
INFO[0000] [*collector.CfAppsCollector] Begin collection 
ERRO[0000] failed to normalize for service/metadata name because it is an empty string 
INFO[0001] [*collector.CfAppsCollector] Done            
INFO[0001] [*collector.CfServicesCollector] Begin collection 
ERRO[0001] failed to normalize for service/metadata name because it is an empty string 
INFO[0001] [*collector.CfServicesCollector] Done        
INFO[0001] Collection done                              
INFO[0001] Collect Output in [/home/ubuntu/m2k_collect]. Copy this directory into the source directory to be used for planning.

$ mv m2k_collect/cf source
```
## <div id='3.3'>3.3. 어플리케이션 변환 준비
- plane 명령어를 통해 Kubernetes에서 실행되도록 앱을 변환하는 방법에 대한 계획을 수립한다.
```
$ move2kube plan -s source

INFO[0000] Configuration loading done                   
INFO[0000] Start planning                               
INFO[0000] Planning started on the base directory: '/home/ubuntu/source' 
INFO[0000] [DockerfileDetector] Planning                
INFO[0006] [DockerfileDetector] Done                    
INFO[0006] [CloudFoundry] Planning                      
INFO[0006] [CloudFoundry] Done                          
INFO[0006] [ComposeAnalyser] Planning                   
INFO[0006] [ComposeAnalyser] Done                       
INFO[0006] [Base Directory] Identified 0 named services and 0 to-be-named services 
INFO[0006] Planning finished on the base directory: '/home/ubuntu/source' 
INFO[0006] Planning started on its sub directories      
INFO[0006] Identified 1 named services and 0 to-be-named services in . 
INFO[0006] Planning finished on its sub directories     
INFO[0006] [Directory Walk] Identified 1 named services and 0 to-be-named services 
INFO[0006] [Named Services] Identified 1 named services 
INFO[0006] Planning done. Number of services identified: 1 
INFO[0006] Plan can be found at [/home/ubuntu/m2k.plan].
``` 
## <div id='3.4'>3.4. 어플리케이션 변환
### <div id='3.4.1'>3.4.1. transform 활용
- transform 명령어를 통해 Kubernetes에서 실행될 수 있도록 어플리케이션을 yaml 파일로 변환한다. 
```
$ move2kube transform -n spring-music

INFO[0000] Detected a plan file at path /home/ubuntu/m2k.plan. Will transform using this plan. 
INFO[0000] Starting transformation                      
? Specify a Kubernetes style selector to select only the transformers that you want to run.
ID: move2kube.transformerselector
Hints:
- Leave empty to select everything. This is the default.

? Select all transformer types that you are interested in:
ID: move2kube.transformers.types
Hints:
- Services that don't support any of the transformer types you are interested in will be ignored.

 ArgoCD, Buildconfig, CloudFoundry, ClusterSelector, ComposeAnalyser, ComposeGenerator, ContainerImagesPushScriptGenerator, DockerfileDetector, DockerfileImageBuildScript, DockerfileParser, DotNetCore-Dockerfile, EarAnalyser, EarRouter, Golang-Dockerfile, Gradle, Jar, Jboss, Knative, Kubernetes, KubernetesVersionChanger, Liberty, Maven, Nodejs-Dockerfile, OperatorTransformer, PHP-Dockerfile, Parameterizer, Python-Dockerfile, ReadMeGenerator, Ruby-Dockerfile, Rust-Dockerfile, Tekton, Tomcat, WarAnalyser, WarRouter, WinWebApp-Dockerfile, ZuulAnalyser
```
### <div id='3.4.2'>3.4.2. 서비스 설정
```
? Select all services that are needed:
ID: move2kube.services.[].enable
Hints:
- The services unselected here will be ignored.

 spring-music-1-0
INFO[0017] Using the transformation option 'Jar' for the service 'spring-music-1-0'. 
INFO[0017] Iteration 1                                  
INFO[0017] Iteration 2 - 1 artifacts to process         
INFO[0017] Transformer 'Jar' processing 1 artifacts     
INFO[0017] Transformer Jar Done                         
INFO[0017] Created 2 pathMappings and 2 artifacts. Total Path Mappings : 2. Total Artifacts : 1. 
INFO[0017] Iteration 3 - 2 artifacts to process         
INFO[0017] Transformer 'DockerfileImageBuildScript' processing 1 artifacts 
```
### <div id='3.4.3'>3.4.3. 레지스트리 정보 입력
```
? Enter the URL of the image registry where the new images should be pushed : 
ID: move2kube.target.imageregistry.url
Hints:
- You can always change it later by changing the yamls.

 index.docker.io
? Enter the namespace where the new images should be pushed : 
ID: move2kube.target.imageregistry.namespace
Hints:
- Ex : spring-music

 ski111
INFO[0022] Transformer DockerfileImageBuildScript Done  
INFO[0022] Transformer 'DockerfileParser' processing 1 artifacts 
INFO[0022] Transformer 'ZuulAnalyser' processing 1 artifacts 
INFO[0022] Transformer ZuulAnalyser Done                
INFO[0022] Transformer DockerfileParser Done            
INFO[0022] Created 1 pathMappings and 3 artifacts. Total Path Mappings : 3. Total Artifacts : 3. 
INFO[0022] Iteration 4 - 3 artifacts to process         
INFO[0022] Transformer 'ClusterSelector' processing 1 artifacts 
```
### <div id='3.4.4'>3.4.4. 클러스터 타입 선택
```
? Choose the cluster type:
ID: move2kube.target."default".clustertype
Hints:
- Choose the cluster type you would like to target

 Kubernetes
 INFO[0023] Transformer ClusterSelector Done             
INFO[0023] Transformer 'ArgoCD' processing 1 artifacts  
? For the service 'spring-music-1-0', do you require a StatefulSet instead of a Deployment?
ID: move2kube.services.spring-music-1-0.statefulset
 No
```
### <div id='3.4.5'>3.4.5. 외부 연결 방식 선택
```
? What kind of service/ingress should be created for the service spring-music-1-0's 8080 port?
ID: move2kube.services."spring-music-1-0"."8080".servicetype
Hints:
- Choose Ingress if you want a ingress/route resource to be created

 NodePort ## 인그레스나 로드밸런서 방식 선택 가능
```
### <div id='3.4.6'>3.4.6. 최소 레플리카 갯수 지정
```
? Provide the minimum number of replicas each service should have
ID: move2kube.minreplicas
Hints:
- If the value is 0 pods won't be started by default

 2
```
### <div id='3.4.7'>3.4.7. 컨테이너 레지스트리 계정 정보 입력
```
? [index.docker.io] What type of container registry login do you want to use?
ID: move2kube.target.imageregistry."index.docker.io".logintype
 username and password
? [index.docker.io] Enter the username to login into the registry : 
ID: move2kube.target.imageregistry."index.docker.io".username
 ski111
? [index.docker.io] Enter the password to login into the registry : 
ID: move2kube.target.imageregistry."index.docker.io".password
 *********
```
### <div id='3.4.8'>3.4.8. CD파이프라인의 대상 네임스페이스 입력
```
? Enter the destination namespace for the Argo CD pipeline
ID: move2kube.transformers.kubernetes.argocd.namespace
Hints:
- If Argo CD pipeline is not relevant to you, then leave empty to use the default value for it.

 
INFO[0038] Transformer ArgoCD Done                      
INFO[0038] Transformer 'ClusterSelector' processing 1 artifacts 
INFO[0038] Transformer ClusterSelector Done             
INFO[0038] Transformer 'Buildconfig' processing 1 artifacts 
INFO[0038] Transformer Buildconfig Done                 
INFO[0038] Transformer 'ComposeGenerator' processing 1 artifacts 
INFO[0038] Transformer ComposeGenerator Done            
INFO[0038] Transformer 'ContainerImagesPushScriptGenerator' processing 1 artifacts 
INFO[0038] Transformer ContainerImagesPushScriptGenerator Done 
INFO[0038] Transformer 'ClusterSelector' processing 1 artifacts 
INFO[0038] Transformer ClusterSelector Done             
INFO[0038] Transformer 'Knative' processing 1 artifacts 
INFO[0038] Transformer Knative Done                     
INFO[0038] Transformer 'ClusterSelector' processing 1 artifacts 
INFO[0038] Transformer ClusterSelector Done             
INFO[0038] Transformer 'Kubernetes' processing 1 artifacts 
INFO[0038] Transformer Kubernetes Done                  
INFO[0038] Transformer 'ClusterSelector' processing 1 artifacts 
INFO[0038] Transformer ClusterSelector Done             
INFO[0038] Transformer 'Tekton' processing 1 artifacts  
...
```
### <div id='3.4.9'>3.4.9. ingress 설정 값 지정
```
? Provide the Ingress class name for ingress
ID: move2kube.target."default".ingress.ingressclassname
Hints:
- Leave empty to use the cluster default
? Provide the ingress host domain
ID: move2kube.target."default".ingress.host
Hints:
- Ingress host domain is part of service URL

 spring-music.com

? Provide the TLS secret for ingress
ID: move2kube.target."default".ingress.tls
Hints:
- Leave empty to use http
```
### <div id='3.4.10'>3.4.10. 복제할 git repository 설정값 입력
```
? Enter the name of an existing K8s secret that has ssh credentials for cloning the git repo
ID: move2kube.target.cicd.tekton.gitreposshsecret
Hints:
- If this is not relevant to you then give an empty string to remove it from the YAML.

 
? Enter the name of an existing K8s secret that has username and password for cloning the git repo
ID: move2kube.target.cicd.tekton.gitrepobasicauthsecret
Hints:
- If this is not relevant to you then give an empty string to remove it from the YAML.
```
### <div id='3.4.11'>3.4.11. 이미지를 레지스트리로 푸시하기 위한 Docker config.json이 있는 기존 K8s secret 입력 
``` 
? Enter the name of an existing K8s secret that has Docker config.json for pushing images to the registry
ID: move2kube.target.cicd.tekton.registrypushsecret
Hints:
- If this is not relevant to you then give an empty string to remove it from the YAML.

 
INFO[0045] Transformer Tekton Done                      
INFO[0045] Created 16 pathMappings and 5 artifacts. Total Path Mappings : 19. Total Artifacts : 6. 
INFO[0045] Iteration 5 - 5 artifacts to process         
INFO[0045] Transformer 'Parameterizer' processing 4 artifacts 
INFO[0045] Transformer Parameterizer Done               
INFO[0045] Transformer 'ReadMeGenerator' processing 5 artifacts 
INFO[0045] Transformer ReadMeGenerator Done             
INFO[0045] Transformation done                          
INFO[0045] Transformed target artifacts can be found at [/home/ubuntu/spring-music].
```

## <div id=3.5/>3.5 이미지 빌드
- 생성된 대상 아티펙트를 사용하여 이미지를 빌드한다.
```
$ cd spring-music/scripts

$ ./buildimages.sh

building image spring-music-1-0
[+] Building 1.3s (8/8) FINISHED                                                                                                                      docker:default
 => [internal] load build definition from Dockerfile                                                                                                            0.1s
 => => transferring dockerfile: 888B                                                                                                                            0.0s
 => [internal] load .dockerignore                                                                                                                               0.0s
 => => transferring context: 2B                                                                                                                                 0.0s
 => [internal] load metadata for registry.access.redhat.com/ubi8/ubi-minimal:latest                                                                             1.1s
 => [1/3] FROM registry.access.redhat.com/ubi8/ubi-minimal:latest@sha256:87bcbfedfd70e67aab3875fff103bade460aeff510033ebb36b7efa009ab6639                       0.0s
 => [internal] load build context                                                                                                                               0.0s
 => => transferring context: 44B                                                                                                                                0.0s
 => CACHED [2/3] RUN microdnf update && microdnf install --nodocs java-17-openjdk-devel && microdnf clean all                                                   0.0s
 => CACHED [3/3] COPY spring-music-1.0.jar .                                                                                                                    0.0s
 => exporting to image                                                                                                                                          0.0s
 => => exporting layers                                                                                                                                         0.0s
 => => writing image sha256:3fea1215a4a1765e0a21494fa6b98838977bb76e77c1f54bcc677b2a9e429a7f                                                                    0.0s
 => => naming to docker.io/library/spring-music-1-0                                                                                                             0.0s
/home/ubuntu/spring-music
Done

```
## <div id=3.6/>3.6 이미지 푸시
- 생성된 대상 아티펙트를 사용하여 이미지를 푸시한다.
```

$ ./pushimages.sh

pushing image spring-music-1-0
Using default tag: latest
The push refers to repository [docker.io/ski111/spring-music-1-0]
6782c4ed7032: Layer already exists 
5cfa6a7e63c4: Layer already exists 
8f42ad26ccda: Layer already exists 
latest: digest: sha256:ee664268686d834c47fbd0b4a2f3414cfa567c0730568c805b8273f3fa2a04a1 size: 954
done
```
# <div id='4'/>4. Kubernetes 설정 및 배포
## <div id='4.1'/>4.1. 공통 가이드
- #### [공통 가이드](../common/common-guide.md)

### [Index](https://github.com/K-PaaS/guide) > [ap2cp-guide](https://github.com/K-PaaS/ap2cp-guide) > Move2Kube