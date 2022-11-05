# Chapter 1. Apach Airflow 살펴보기

> Airflow란?
> 
> 
> : Airflow is a platform to programmatically author, schedule and monitor workflows **프로그래밍 방식**으로 **워크플로우(작업 흐름)를 작성, 스케줄링 및 모니터링을 하는 플랫폼**이다.
> 

## Airflow 주요 개념

### 1. DAG

- Directed Acyclic Graph(방향성 비순환 그래프)
- Airflow는 태스크의 연결 관계를 DAG로 관리하고 웹 인터페이스를 통해 DAG 구조를 시각적으로 확인할 수 있다

### 2. Operator(오퍼레이터)

- 태스크의 Wrapper 역할. 원하는 작업을 실행하기 위함이다
- Action Operator : 기능, 명령 실행. ex) bash Operator, Python Operator
- Transfer Operator : 소스에터 Destination으로 데이터 전송
- Sensor Operator : 특정 조건을 감지하면 실행

### 3. Task(태스크)

- 데이터 파이프라인에 존재하는 Operator
- 데이터 파이프라인이 트리거되어 실행될 때 생성된 task를 Task Instance라고 한다

### 4. Workflow

- DAG를 통해 태스크 간 의존성을 정의하고, 각 태스크를 오퍼레이터로 실행하는 일련의 과정으로 정의

## Airflow 동작 원리

<figure><img src="images/Chapter 1 Apach Airflow 살펴보기/Untitled.png" width="100%"></figure>

- **Scheduler** : 워크플로우를 스케줄링한다. 모든 DAG와 태스크를 모니터링하고 관리하며, 주기적으로 실행해야 할 태스크를 찾고 해당 태스크를 실행 가능한 상태로 변경한다.
- **DAG Script** : 개발자가 작성한 파이썬 워크플로우 스크립트
- **Web Server** : 에어플로우 웹 인터페이스
- **Meta Store** : Airflow 메타데이터 저장소. 어떤 DAG가 존재하고 어떤 태스크로 구성되는지, 어떤 태스크가 실행중이고, 또 실행 가능한 상태인지 등의 정보 저장
- **Workers** : 어떤 환경에서 태스크가 실행될 지 정의. 태스크를 실행하는 주체

## Airflow 장점

### 1. Dynamic Data Pipeline

Airflow는 파이썬을 이용하여 데이터 파이프라인을 정의한다. 따라서 파이썬으로 가능한 대부분의 작업들을 Airflow 파이프라인에서 처리 가능하다

### 2. 확장성

Airflow는 확장성이 뛰어나다. 다양한 task를 병렬적으로 실행할 수 있으며, 쿠버네티스 클러스터, 분산 클러스터 환경에서 파이프라이닝이 가능하다.

### 3. 편리한 유저 인터페이스

웹 서버에서 제공하는 웹 인터페이스를 통해 데이터 파이프라인을 모니터링하고 관리하기 편하다.

### 4. High Extensiblility

플러그인 설치가 쉽다. 새로운 작업 툴이 나와 적용할 필요가 있을 땐 플러그인을 개발하여 적용할 수 있다.