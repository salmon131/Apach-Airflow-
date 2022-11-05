# Chapter 2. Airflow DAG의 구조

## 2.1 다양한 소스에서 데이터 수집

**사례) 로켓 애호가**

> 로켓 애호가 John은 로켓 발사에 대한 정보를 자동으로 수집하여 최신 로켓 발사에 대해 간파할 수 있도록 하는 프로그램을 작성하고자 한다.

<figure><img src="images/Chapter 2 Airflow DAG의 구조/Untitled.png" alt=""><figcaption><p>John의 로켓 이미지 다운로드 심성 모형</p></figcaption></figure>

1. Launch Libaray 2에서 다음 5개의 로켓 발사에 대한 데이터 가져오기
2. John의 컴퓨터에 데이터 저장
3. 인터넷에서 로켓 이미지 가져오기
4. John의 컴퓨터에 이미지 저장
5. 시스템 알림

## 2.2 Airflow DAG으로 작성

<figure><img src="images/Chapter 2 Airflow DAG의 구조/Untitled 1.png" alt=""><figcaption></figcaption></figure>

### 왜 Airflow인가?

* 다중 태스크를 병렬로 실행할 수 있다.
* 서로 다른 기술을 사용할 수 있다.
  * ex) 처음 태스크는 bash 스크립트로, 다음 태스크는 파이썬 스크립트로 실행한다.
* 태스크를 나누는 방법에는 정해진 답은 없다.

### 태스크와 오퍼레이터의 차이점?

**오퍼레이터**

* 단일 작업 수행 역할
  * BashOperator, PythonOperator: 일반적인 작업
  * EmailOperator, HTTPOperator: 특수 목적을 위한 작업

**태스크**

* 오퍼레이터와 같은 의미로 사용
* 단, Airflow에서 **태스크는 오퍼레이터의 ‘래퍼wrapper’ 또는 ‘매니저manager’** 로 생각한다.
* 즉 태스크는 오퍼레이터의 상태를 관리하고 사용자에게 상태 변경(시작/완료)를 표시하는 Airflow의 내장 컴포넌트이다.

<figure><img src="images/Chapter 2 Airflow DAG의 구조/Untitled 2.png" alt=""><figcaption></figcaption></figure>

> 전체 코드

```python
import json
import pathlib

import airflow.utils.dates
import requests
import requests.exceptions as requests_exceptions
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

dag = DAG( # 객체의 인스턴스 생성
    dag_id="download_rocket_launches", # Airflow UI에 표시되는 DAG 이름
    description="Download rocket pictures of recently launched rockets.",
    start_date=airflow.utils.dates.days_ago(14), # 워크플로가 처음 실행되는 날짜/시간
    schedule_interval="@daily", # DAG 실행 간격
)

download_launches = BashOperator( # BashOperator를 이용해 curl로 URL 결과값 다운로드
    task_id="download_launches", # 태스크 이름
    bash_command="curl -o /tmp/launches.json -L 'https://ll.thespacedevs.com/2.0.0/launch/upcoming'",  # noqa: E501
    dag=dag,
)

# 파이썬 함수는 결괏값을 파싱하고 모든 로켓 사진을 다운로드
def _get_pictures():
    # 경로가 존재하는지 확인
    pathlib.Path("/tmp/images").mkdir(parents=True, exist_ok=True)

    # launches.json 파일에 있는 모든 그림 파일을 다운로드
    with open("/tmp/launches.json") as f:
        launches = json.load(f)
        image_urls = [launch["image"] for launch in launches["results"]]
        for image_url in image_urls:
            try:
                response = requests.get(image_url)
                image_filename = image_url.split("/")[-1]
                target_file = f"/tmp/images/{image_filename}"
                with open(target_file, "wb") as f:
                    f.write(response.content)
                print(f"Downloaded {image_url} to {target_file}") # Airflow 로그에 저장하기 위해 stdout으로 출력
            
						#잠재적 에러 포착 및 처리
						except requests_exceptions.MissingSchema:
                print(f"{image_url} appears to be an invalid URL.")
            except requests_exceptions.ConnectionError:
                print(f"Could not connect to {image_url}.")

# DAG에서 PythonOperator를 사용하여 파이썬 함수 호출
# 편의를 위해 파이썬 함수와 task_id는 동일하게 한다.
get_pictures = PythonOperator(
    task_id="get_pictures", python_callable=_get_pictures, dag=dag
)

notify = BashOperator(
    task_id="notify",
    bash_command='echo "There are now $(ls /tmp/images/ | wc -l) images."',
    dag=dag,
)

# 태스크 실행 순서 지정
download_launches >> get_pictures >> notify
```

## 2.3 Airflow에서 DAG 실행하기

### **Airflow의 세 가지 컴포넌트**

1. 스케줄러
2. 웹 서버
3. 데이터베이스

### **파이썬 환경에서 실행**

```bash
pip install apache-airflow
```

### **도커 컨테이너에서 Airflow 실행**

```bash
#!/bin/bash

# Note: 아래와 같이 하나의 도커 컨테이너 안에 여러 프로세스를 실행하는것은 예제를 위한 간단한 데모 환경을 위해서임
# 웹 서버, 스케줄러, 메타스토어를 별도 컨테이너에서 실행하는 프로덕션 상의 설정 방법은 10장에 나와있음

set -x

SCRIPT_DIR=$(cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd)

# -t: 터미널 -i: 상호작용
# -p: 호스트 포트:컨테이너 포트
# 컨테이너 8080 포트 개방한 뒤 호스트의 8080포트와 연결
# -v: 호스트 경로:컨테이너 내의 경로로 마운트
# 컨테이너에서 메타스토어 초기화 후 웹서버와 스케줄러 시작

docker run \
-ti \
-p 8080:8080 \
-v ${SCRIPT_DIR}/../dags/download_rocket_launches.py:/opt/airflow/dags/download_rocket_launches.py \
--name airflow \
--entrypoint=/bin/bash \
apache/airflow:2.0.0-python3.8 \
-c '( \
airflow db init && \
airflow users create --username admin --password admin --firstname Anonymous --lastname Admin --role Admin --email admin@example.org \
); \
airflow webserver & \
airflow scheduler \
'
```

### Airflow UI 둘러보기

<figure><img src="images/Chapter 2 Airflow DAG의 구조/Untitled 3.png" alt=""><figcaption></figcaption></figure>

* DAG을 실행하려면 버튼을 토글해서 ‘On’ 상태로 만들어야 한다.
* 그 다음 DAG 트리거(>버튼)을 눌러서 실행한다.
* 다운로드된 이미지 확인시
  * 도커 컨테이너로 접속 후 /tmp/images 로 이동한다.

```bash
docker exec -it airflow /bin/bash
or
docker exec -it [scheduler CONTAINER ID] /bin/bash
```

```bash
(emd) 😊 [ml@gpu01] /data2 $ docker exec -it airflow /bin/bash
airflow@9f7e3fb76f26:/opt/airflow$ cd /tmp/images
airflow@9f7e3fb76f26:/tmp/images$ ls
epsilon_image_20221009075145.jpg          falcon_heavy_image_20220129192819.jpeg      launcherone_image_20200101110016.jpeg  rs1_image_20211102160004.jpg            soyuz25202.1b_image_20190520165608.png
falcon_9_block__image_20210506060831.jpg  gslv2520mk2520iii_image_20190604000938.jpg  proton-m_image_20191211081456.jpeg     soyuz25202.1b_image_20190520165337.jpg  soyuz_2.1a_image_20201013143850.jpg
```

## 2.4 스케줄 간격으로 실행하기

```python
dag = DAG(
   dag_id="download_rocket_launches",
   start_date=airflow.utils.dates.days_ago(14),
   schedule_interval="@daily",
)
```

* schedule\_interval="@daily" : DAG을 하루에 한 번 실행
* start\_date가 14일전으로 주어짐
* 14일 전부터 현재까지 시간을 1일 간격으로 나누면 14가 된다.
  * 아래에서 컬럼 수가 14개, 행 3개는 각각의 태스크

<figure><img src="images/Chapter 2 Airflow DAG의 구조/Untitled 4.png" alt=""><figcaption></figcaption></figure>

## 2.5 실패한 태스크에 대한 처리

<figure><img src="images/Chapter 2 Airflow DAG의 구조/Untitled 5.png" alt=""><figcaption></figcaption></figure>

* 이미지를 가져오는 태스크가 실패할 경우
* 순차실행되어야 하므로 그 다음 알람 태스크는 실행되지 않음

<figure><img src="images/Chapter 2 Airflow DAG의 구조/Untitled 6.png" alt=""><figcaption></figcaption></figure>

<figure><img src="images/Chapter 2 Airflow DAG의 구조/Untitled 7.png" alt=""><figcaption></figcaption></figure>

<figure><img src="images/Chapter 2 Airflow DAG의 구조/Untitled 8.png" alt=""><figcaption><p>   </p></figcaption></figure>

<figure><img src="images/Chapter 2 Airflow DAG의 구조/Untitled 9.png" alt=""><figcaption></figcaption></figure>

* 실패한 태스크를 삭제하려면 클릭 > 팝업에서 ‘Clear’ > ‘OK’
* 문제가 해결되고 DAG가 ON 되어있다면 태스크가 다시 성공적으로 실행되고 녹색 뷰가 됨

## 2장 정리

* Airflow의 워크플로는 DAG로 표시함
* 오퍼레이터는 단일 태스크를 나타냄
* Airflow는 일반적인 작업과 특정 작업 모두에 대한 오퍼레이터의 집합을 포함
* Airflow UI는 DAG 구조를 확인하기 위한 그래프 뷰와 시간 경과에 따른 DAG 실행 상태를 보기 위한 트리 뷰를 제공
* DAG 안에 있는 태스크는 어디에 위치하든지 재시작할 수 있음
