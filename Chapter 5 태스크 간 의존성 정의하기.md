# Chapter 5 태스크 간 의존성 정의하기

## Chapter 5. 태스크 간 의존성 정의하기

### 5.1 기본 의존성 유형

#### 선형 의존성 유형 (linear chain)

* 연속적으로 실행되는 작업
* 이전 태스크가 성공해야 다음 태스크 수행
  * ex) 2장의 로켓 사진 가져오기 DAG
* 오른쪽 비트 시프트 연산자(>>)를 사용하여 태스크 간에 의존성을 만들 수 있음.

```jsx
# 작업 의존성을 각각 설정하기
download_launches >> get_pictures
get_pictures >> notify

#여러 개의 의존성을 설정하기
download_launches >> get_pictures >> notify
```

* 의존성이 충족된 경우에만 다음 태스크를 스케줄 하기 때문에 여러 태스크에서 순서가 명확하게 정의된다는 장점이 있음

#### 팬인/팬아웃 의존성 (Fan-in/Fan-out)

* 하나의 태스크가 여러 다운스트림 태스크에 연결되거나 그 반대의 동작을 수행하는 유형
*   태스크 간 복잡한 의존성 구조를 만들 수 있음

    * ex) 1장의 우산 수요 예측 사례
    * 서로 다른 소스에서 데이터를 가져와서 정제하는 작업은 서로 의존성이 없음
    * 두 데이터를 합치는 작업은 앞의 두 작업 모두에 의존성이 있음



    <figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled.png" alt=""><figcaption></figcaption></figure>

**팬아웃 의존성**

* 한 태스크가 여러 다운스트림 태스크와 연결되는 경우
* 두 태스크의 업스트림에 DAG의 시작을 나타내는 더미 태스크를 추가하여 암묵적으로 팬아웃 종속성을 정의할 수 있다.

```bash
from airflow.operators.dummy import DummyOperator
 
start = DummyOperator(task_id="start")
start >> [fetch_weather, fetch_sales]
```

**팬인 의존성**

* 한 태스크가 여러 업스트림 태스크와 연결되는 경우
* 아래와 같이 팬인 종속성을 정의할 수 있다.

```python
[clean_weather, clean_sales] >> join_datasets
```

* 전체 태스크 결합 후 생성되는 DAG과 실행순서

<figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 1.png" alt=""><figcaption></figcaption></figure>

* 시작 태스크(1)가 완료 되면 fetch\_weather(2a)와 fetch\_sales(2b) 태스크가 병렬로 실행됨
  * 팬아웃 의존성
* fetch\_weather(2a)가 완료되면 clean\_weather(3a)가 실행되고, fetch\_sales(2b)가 완료되면 clean\_sales(3b)가 실행됨
* clean\_weather(3a)와 clean\_sales(3b)가 모두 완료되면 join\_datasets(4) 태스크가 실행됨
  * 팬인 의존성

### 5.2 브랜치하기

* 시스템상의 변경이 발생할 경우 이전 시스템과 새로운 시스템이 정상 동작하기 위해 브랜치를 작성한다.
  * ex) 판매 데이터가 다른 소스에서 제공될 예정인 경우

#### 태스크 내에서 브랜치하기

* 데이터 수집 태스크 내에 시스템 변경 날짜를 변수로 주어 해당 날짜 전후인지 확인 후 다른 코드가 실행되도록 한다.

```
def _fetch_sales(**context):
    if context["execution_date"] < ERP_CHANGE_DATE:
        _fetch_sales_old(**context)
    else:
        _fetch_sales_new(**context)
```

* 장점) DAG 자체의 구조를 수정하지 않고도 DAG에서 약간의 유연성을 허용할 수 있다.
* 단점)
  *   완전히 다른 태스크 체인이 필요한 경우 적용할 수 없다.



      <figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 2.png" alt=""><figcaption></figcaption></figure>
  *   DAG 실행 중에 Airflow에서 어떤 코드 분기를 사용하고 있는지 확인하기 어렵다.

      * 실제 브랜치가 작업 내에 숨겨져 있기 때문에 DAG 뷰는 이전과 달라지는 점이 없다.



      <figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 3.png" alt=""><figcaption></figcaption></figure>

#### DAG 내부에서 브랜치하기

* 서로 다른 시스템에 각각 태스크 셋을 개발하고 DAG가 데이터 수집 작업을 어떤 시스템에서 실행할지 선택하도록 한다.

<figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 4.png" alt=""><figcaption></figcaption></figure>

**BranchPythonOperator**

* 어떤 다운스트림 태스크 세트를 실행할지 선택하는 기능을 제공한다.
* 해당 오퍼레이터는 작업 결과로 다운스트림 태스크의 ID를 반환한다.

```python
def _pick_erp_system(**context):
   if context["execution_date"] < ERP_SWITCH_DATE:
       return "fetch_sales_old"
   else:
       return "fetch_sales_new"

pick_erp_system = BranchPythonOperator(
   task_id="pick_erp_system",
   python_callable=_pick_erp_system,
)
fetch_sales_old = PythonOperator(
        task_id="fetch_sales_old", python_callable=_fetch_sales_old
    )

pick_erp_system >> [fetch_sales_old, fetch_sales_new]
```

* 최종적으로 DAG는 아래와 같이 만들어진다.
* 그러나 예상과 달리 **join\_dataset 이후 태스크는 수행되지 않는다.**

<figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 5.png" alt=""><figcaption></figcaption></figure>

* 왜냐하면 pick\_erp\_system으로 두 정제 태스크 중 하나만 실행완료되는 경우, join\_datasets 는 자신의 업스트림 태스크가 모두 성공적으로 완료되지 않았기 때문에 태스크를 실행하지 않게 된다.
* 이는 **잘못된 트리거 규칙**으로 브랜치가 결합되었기 때문에 발생하는 문제이다.

**트리거 규칙**

*   trigger\_rule : 개별 태스크에 대해 트리거 규칙을 정의할 수 있는 인수

    * `all_success` : 모든 상위 태스크가 성공해야 해당 태스크 실행
    * `none_failed` : 모든 상위 항목이 실행 완료 및 실패가 없을 시에 즉시 작업 실행

    ```python
    join_datasets = PythonOperator(task_id="join_datasets", trigger_rule="none_failed")
    ```

**DummyOperator**

* 더미 작업을 위한 Airflow 내장 오퍼레이터
* 서로 다른 브랜치를 결합하여 브랜치 조건을 명확하게 하기 위해 사용

<figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 6.png" alt=""><figcaption></figcaption></figure>

* 브랜치 구조를 더 명확하게 하기 위해 브랜치 후에 추가적인 조인 작업을 추가하여, 나머지 DAG를 진행하기 전 브랜치의 계보를 연결하는 join\_erp\_branch 태스크를 추가한다.
* 해당 태스크를 통해 필요한 트리거 규칙을 설정할 수 있으므로 DAG의 다른 태스크에 대한 트리거 규칙을 변경할 필요가 없다.
* 즉, 더 이상 **join\_datasets 태스크에 대한 트리거 규칙을 설정할 필요가 없고, 브랜치를 좀 더 독립적으로 유지할 수 있게 된다.**

### 5.3 조건부 태스크

* Airflow는 특정 조건에 따라 DAG에서 특정 태스크를 건너뛸 수 있는 다른 방법도 제공한다.

#### 태스크 내에서 조건

* 장점) 가장 최근 실행된 DAG에 대해서만 모델을 배포하도록 DAG를 변경하여 해결할 수 있다.
* 단점)
  * 의도한 대로 동작은 하지만 배포 로직 조건이 혼용되고, PythonOperator 이외의 다른 기본 제공 오퍼레이터에서는 활용할 수 없다.
  * 배포 조건이 deploy\_model 태스크 내에서 내부적으로 확인되기 때문에 Airflow UI에서 태스크 결과를 추적할 때 혼란스러울 수 있다.

```python
def _deploy_model(**context):
    if _is_latest_run(**context):
        print("Deploying model")

deploy = PythonOperator(
			task_id="deploy_model",
			python_callable=_deploy,
)
```

#### 조건부 태스크 만들기

* 미리 정의된 조건에 따라서 실행된다.
* 현재 실행이 가장 최근 실행한 DAG인지 확인하는 태스크를 추가하고, 이 태스크의 다운스트림에 배포 태스크를 추가하여 배포를 조건부화 할 수 있다.

<조건부 태스크 추가 전>

<figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 7.png" alt=""><figcaption></figcaption></figure>

<조건부 태스크 추가 후>

<figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 8.png" alt=""><figcaption></figcaption></figure>

```python
from airflow.exceptions import AirflowSkipException

def _latest_only(**context):
    # 실행 윈도우에서 경계를 확인
    left_window = context["dag"].following_schedule(context["execution_date"])
    right_window = context["dag"].following_schedule(left_window)

    # 현재 시간이 윈도우 안에 있는지 확인
    now = pendulum.now("UTC")
    if not left_window < now <= right_window:
        raise AirflowSkipException("Not the most recent run!")
```

<figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 9.png" alt=""><figcaption></figcaption></figure>

* 가장 최근 실행에 대해서만 배포 태스크가 수행됨

#### 내장 오퍼레이터 사용하기

* LastOnlyOperator 클래스
  * PythonOperator를 기반으로 동일한 작업을 가능하게 한다.
  * 조건부 배포를 구현하기 위해 복잡한 로직을 작성 할 필요가 없다.
  * 그러나 많이 복잡한 경우에는 PythonOperator 기반으로 구현하는 것이 더 효율적이다.

```python
from airflow.operators.latest_only import LatestOnlyOperator

latest_only=LatestOnlyOperator(
			task_id="latest_only",
			dag=dag
)

join_datasets >> train_model >> deploy_model
latest_only >> deploy_model
```

### 5.4 트리거 규칙에 대한 추가 정보

#### 트리거 규칙이란?

* 태스크가 실행 준비가 되어 있는지 여부를 결정하는 조건
* 기본 트리거 규칙은 `all_success`
  * 즉, 모든 의존적 태스크가 성공적으로 완료되어야함

<figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 10.png" alt=""><figcaption></figcaption></figure>

A. 위의 DAG에서 선행 태스크가 필요없는 유일한 ‘start’ 태스크를 실행하여 DAG 실행을 시작

B. 시작 태스크 성공적 완료 → 다음 태스크 실행 준비 → Airflow에 의해 선택

#### 실패의 영향

<figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 11.png" alt=""><figcaption></figcaption></figure>

* 태스크가 실행중 실패할 경우
* 그 다음 태스크는 `upstream_failed` 상태가 할당
* `all_success` 규칙에 의해 다운 스트림 태스크 실행되지 않음
* 해당 동작 유형을 **“전파”** 라고 한다.

#### 기타 트리거 규칙

| 트리거 규칙        | 트리거 조건                                                | 사용 사례                                               |
| ------------- | ----------------------------------------------------- | --------------------------------------------------- |
| all\_success  |                                                       |                                                     |
|  (default)    | 모든 상위 태스크가 성공적으로 완료                                   | 기본 트리거 규칙임                                          |
| all\_failed   | 모든 상위 태스크가 실패했거나 상위 태스크의 오류로 인해 실패했을 경우               | 태스크 그룹에서 하나 이상 실패가 예상되는 상황에서 오류 처리 코드를 트리거          |
| all\_done     | 결과에 관계없이 모든 상위 태스크가 실행을 완료                            | 모든 태스크가 완료되었을 때 실행할 청소 코드를 실행(예: 시스템 종료 또는 클러스터 중지) |
| one\_failed   | 하나 이상의 상위 태스크가 실패하자마자 트리거되며 다른 상위 태스크의 실행 완료를 기다리지 않음 | 알림 또는 롤백과 같은 일부 오류 처리 코드를 빠르게 트리거                   |
| one\_success  | 한 상위 태스크가 성공하자마자 트리거되며 다른 상위 태스크의 실행 완료를 기다리지 않음      | 하나의 결과를 사용할 수 있게 되는 즉시 다운스트림 연산/알림을 빠르게 트리거         |
| none\_failed  | 실패한 상위 태스크가 없지만, 태스크가 성공 또는 건너뛴 경우                    | 조건부브랜치의 결합 (join\_erp\_branch)                      |
| none\_skipped | 건너뛴 상위 태스크가 없지만 태스크가 성공 또는 실패한 경우                     | 모든 업스트림 태스크가 실행된 경우, 해당 결과를 무시하고 트리거                |
| dummy         | 업스트림 태스크의 상태와 관계없이 실행                                 | 테스트 시                                               |

### 5.5 태스크 간 데이터 공유

#### XCom을 사용하여 데이터 공유하기

* XCom은 기본적으로 태스크 간에 메시지를 교환하여 특정 상태를 공유할 수 있게 함
* **사용 예시:**
  * 우산 사용 예제에서 훈련된 모델 배포 시 모델의 버전 식별자를 deploy\_model 태스크에 전달해야 가능
  * 이때 XCom을 이용하여 train\_model ↔ deploy\_model 간에 모델 식별자 공유
  * Airflow 컨텍스트의 태스크 인스턴스의 xcom\_push 메서드를 사용하여 값을 게시

```python
def _train_model(**context):
   model_id = str(uuid.uuid4())
   context["task_instance"].xcom_push(key="model_id", value=model_id)

train_model = PythonOperator(
   task_id="train_model", 
   python_callable=_train_model,
)
```

&#x20;\- 등록된 값은 Admin > XComs에서 확인 가능 - xcom\_pull을 사용하여 다른 태스크에서도 XCom 값을 확인 가능

<figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 12.png" alt=""><figcaption></figcaption></figure>

```python
def _deploy_model(**context):
   model_id = context["task_instance"].xcom_pull(
       task_ids="train_model", key="model_id"
   )
   print(f"Deploying model {model_id}")

deploy_model = PythonOperator(
   task_id="deploy_model", 
   python_callable=_deploy_model,
)
```

* 이때 xcom\_pull을 통해 값을 가져올 때 dag\_id 및 실행날짜 설정 가능
  * (default) **:** 현재 DAG id와 실행 날짜로 설정
* **다른 DAG 또는 다른 실행 날짜로부터 값을 가져오기 위해 특정 값을 지정할 수 있지만, 매우 합당한 이유가 있지 않는 한 디폴트로 설정된 값을 사용하는 것을 권장함**

**템플릿에서 XCom 값 사용 가능**

```python
def _deploy_model(templates_dict, **context):
    model_id = templates_dict["model_id"]
    print(f"Deploying model {model_id}")
 
deploy_model = PythonOperator(
    task_id="deploy_model",
    python_callable=_deploy_model,
    templates_dict={
        "model_id": "{{task_instance.xcom_pull(
	         task_ids='train_model', key='model_id')}}"
    },
)
```

**XCom 값을 자동으로 게시하는 오퍼레이터**

* Ex 1) BashOperator 에 `xcom_push=true` 로 설정시 stdout에 기록된 마지막 행을 XCom 값으로 게시
*   Ex 2) PythonOperator 는 파이썬 호출 가능한 인수에서 반환된 값을 XCom 값으로 게시

    ```python
    def _train_model(**context):
       model_id = str(uuid.uuid4())
       return model_id
    ```



    * 기본 키는 return\_value로 등록됨

    <figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 13.png" alt=""><figcaption></figcaption></figure>

#### XCom 사용시 고려사항

* 단점)
  * 풀링 태스크는 필요한 값을 사용하기 위해 태스크 간에 묵시적인 의존성이 필요함
  * DAG에 표시되지 않으며 태스크 스케줄 시에 고려되지 않음
    * 따라서 서로 다른 DAG에서 실행 날짜 사이에 XCom 값을 공유하는걸 권장하지 않음
  * 오퍼레이터의 원자성을 무너뜨리는 패턴이 될 가능성이 있음
    * Ex) API 접근 토큰을 가져와 XCom을 이용해 전달하는 경우 토큰 사용 시간이 만료되어 두 번째 태스크를 재실행하지 못할 수 있음
  * XCom에 저장하는 모든 값은 직렬화를 지원해야 한다는 기술적 한계 존재
    * 즉, 람다 또는 여러 다중 멀티프로세스 관련 클래스 같은 유형은 저장 불가
  * 사용되는 백엔드에 의해 XCom 값의 저장 크기가 제한될 수 있음
    * _SQLite_—BLOB 유형으로 저장, 2GB 제한
    * _PostgreSQL_—BYTEA 유형으로 저장, 1GB 제한
    * _MySQL_—BLOB 유형으로 저장, 64KB 제한
* 따라서 태스크 간의 의존성을 명확하게 기록하고 사용법을 신중히 검토해야함

#### 커스텀 XCom 백엔드 사용하기

* Airflow2의 XCom 백엔드 지정 옵션을 사용하면 커스텀 클래스를 정의하여 XCom을 저장 및 검색 가능
* 해당 클래스 사용시 BaseXCom 기본 클래스를 상속해야함
* 값을 직렬화/역직렬화 하기 위해 두 가지 정적 메서드 각각 구현

```python
from typing import Any
from airflow.models.xcom import BaseXCom

class CustomXComBackend(BaseXCom):

   @staticmethod
   def serialize_value(value: Any): # XCom 값이 오퍼레이터에 게시될 때마다 호출
       ...

   @staticmethod
   def deserialize_value(result) -> Any: # XCom 값이 백엔드에서 가져올 때 호출
       ...
```

* 장점)
  * XCom 값 저장 선택을 다양화
  * ex) 클라우드 스토리지에도 저장 가능

### 5.6 Taskflow API로 파이썬 태스크 연결하기

* 많은 태스크 연결시 파이썬 태스크 및 의존성을 정의하기 위해 새로운 데코레이터 기반 API 추가 제공

#### Taskflow API로 파이썬 태스크 단순화하기

* **이전 접근 방식:**
  * 훈련/배포 함수로 정의한 후 Python Operator를 사용하여 태스크를 생성함
  * 모델 ID를 공유하기 위해 xcom\_push 및 xcom\_pull을 명시적으로 사용함
*   **Taskflow API:**

    * 파이썬 함수를 태스크로 쉽게 변환
    * DAG 정의에서 태스크 간에 데이터 공유를 명확하게 함

    ```python
    import uuid

    import airflow

    from airflow import DAG
    from airflow.decorators import task

    with DAG(
        dag_id="12_taskflow",
        start_date=airflow.utils.dates.days_ago(3),
        schedule_interval="@daily",
    ) as dag:

        @task
        def train_model():
            model_id = str(uuid.uuid4())
            return model_id

        @task
        def deploy_model(model_id: str):
            print(f"Deploying model {model_id}")
    		
    		# 파이썬 코드와 유사한 구문을 사용하여 연결 가능
        model_id = train_model()
        deploy_model(model_id)
    ```



    * **해당 코드가 작동되는 방식?**
      * decorated train\_model 함수 호출 → 해당 태스크를 위한 새로운 오퍼레이터 인스턴스 생성
      * 함수의 return 문에서 값을 XCom으로 자동 등록
      * decarated deploy\_model 함수 호출 → 오퍼레이터 인스턴스 생성 + model\_id 출력 함께 전달

    <figure><img src="images/Chapter 5 태스크 간 의존성 정의하기/Untitled 14.png" alt=""><figcaption></figcaption></figure>

#### Taskflow API를 사용하지 않는 경우

* 단점)
  * Taskflow는 PythonOperator를 사용하여 구현되는 파이썬 태스크에서만 사용 가능
  * 다른 오퍼레이터와 관련 태스크는 일반 API를 이용하여 태스크와 의존성을 정의해야함
  * 두 스타일을 혼합하여 사용할 경우 주의하지 않으면 코드가 복잡해 보일 수 있음

## 요약

* Airflow 기본 태스크 의존성을 이용해 Airflow DAG에서 선형 태스크 의존성 및 팬인/팬아웃 구조 정의 가능
* BranchPythonOperater를 사용하면 DAG에 브랜치를 구성하여 특정 조건에 따라 여러 실행 경로 선택 가능
* 조건부 태스크를 사용해 특정 조건에 따라 의존성 태스크를 실행 가능
* DAG 구조에서 브랜치 및 조건을 명시적으로 코딩하면 DAG 실행 방식을 해석하는데 도움이 됨
* Airflow 태스크의 트리거는 트리거 규칙에 의해 제어되며, 다양한 상황에 대응할 수 있도록 구성 가능
* XCom을 사용하여 태스크 간에 상태 공유 가능
* Taskflow API는 파이썬 태스크가 많은 DAG를 단순화 하는 데 도움이 됨
