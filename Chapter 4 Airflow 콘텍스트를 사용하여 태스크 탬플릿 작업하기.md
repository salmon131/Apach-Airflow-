# Chapter 4. Airflow 콘텍스트를 사용하여 태스크 템플릿 작업하기

## 4.1 Airflow로 처리할 데이터 검사하기

* 파이프라인을 구축하기 전, 접근 방식에 대한 기술적 계획을 세우는 것이 중요하다.
* 데이터로 무엇을 하려는지에 따라 빈도, 데이터의 크기, 형식, 소스 유형 등을 고려해야한다.
*   시뮬레이션 할 예제 서비스 설명

    * 주식 시장 예측을 위해 회사의 페이지 뷰 증가 / 감소를 파악하고 감성 분석을 수행
    * 위키피디어에서 매 시간 페이지 뷰 덤프 하나를 다운로드
    * 압축 해제 후 페이지 뷰 데이터를 확인



    <figure><img src="images/Chapter 4 Airflow 콘텍스트를 사용하여 태스크 탬플릿 작업하기/Untitled.png" alt=""><figcaption></figcaption></figure>

## 4.2 태스크 콘텍스트와 Jinja 템플릿 작업

### 오퍼레이터의 인수 템플릿 작업

```python
dag = DAG(
    dag_id="listing_4_01",
    start_date=airflow.utils.dates.days_ago(3),
    schedule_interval="@hourly",
)

get_data = BashOperator(
    task_id="get_data",
    bash_command=(
        "curl -o /tmp/wikipageviews.gz "
        "https://dumps.wikimedia.org/other/pageviews/"
        "{{ execution_date.year }}/" #이중 중괄호는 런타임 시에 삽입될 변수를 나타낸다.
        "{{ execution_date.year }}-{{ '{:02}'.format(execution_date.month) }}/"
        "pageviews-{{ execution_date.year }}"
        "{{ '{:02}'.format(execution_date.month) }}"
        "{{ '{:02}'.format(execution_date.day) }}-"
        "{{ '{:02}'.format(execution_date.hour) }}0000.gz" #모든 파이썬 변수 또는 표현식에 대해 제공할 수 있다.
    ),
    dag=dag,
)
```

* execution\_data는 **작업 런타임 시에 사용**할 수 있는 변수이다.
  * 런타임시에 값이 들어오기 때문에 프로그래밍 할 때는 값을 알 수 없다.
* 이중 중괄호는 Jinja 템플릿 문자열을 나타낸다.
* Airflow는 날짜 시간에 [Pendulum](https://pendulum.eustace.io/) 라이브러리를 사용하는데, execution\_data는 Pendulum 라이브러리의 datetime 객체이기 때문에 파이썬 datetime의 호환 객체이다.
* pendulum은 파이썬의 datetime과 동일하게 작동하기 때문에 같은 결과를 얻는다.

```python
>>> from datetime import datetime
>>> import pendulum
>>> datetime.now().year
2022
>>> pendulum.now().year
2022
```

* 따라서 execution\_date의 month, day, hour를 얻을 수 있는 것이다.

### 템플릿에 무엇이 사용 가능할까요?

&#x20;

<figure><img src="images/Chapter 4 Airflow 콘텍스트를 사용하여 태스크 탬플릿 작업하기/Untitled 1.png" alt=""><figcaption></figcaption></figure>

* execution\_date 뿐만 아니라 더 많은 변수가 템플릿화 될 수 있다.
* 태스크 콘텍스트 reference 설명 (version 2.4.1 기준)

| 키                                    | 설명                                                            | 예시                                                                                    |
| ------------------------------------ | ------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| data\_interval\_start                | 태스크 스케줄 간격의 시작 날짜 / 시간 (기존의 execution\_date)                  | DateTime(2022, 10, 23, 0, 0, 0, tzinfo=Timezone('UTC'))                               |
| data\_interval\_end                  | 태스크 스케줄 간격의 종료 날짜 / 시간 (기존의 next\_execution\_date)            | DateTime(2022, 10, 24, 0, 0, 0, tzinfo=Timezone('UTC'))                               |
| logical\_date                        | 태스크 스케줄 간격의 시작 날짜 / 시간 (기존의 execution\_date)                  | 자동실행이랑 트리거랑 결과 값 다름.. (확인 필요) 자동실행은 data\_interval\_start와 같고, 트리거는 트리거 작동 시간임        |
| ds                                   | %Y-%m-%d 형식의 logical\_date                                    | “2018-01-01”                                                                          |
| ds\_nodash                           | %Y%m%d 형식의 logical\_date                                      | “20180101”                                                                            |
| ts                                   | %Y-%m-%dT%H:%M:%S%z 형식의 logical\_date                         | ”2018-01-01T00:00:00+00:00”                                                           |
| ts\_nodash\_with\_tz                 | %Y%m%dT%H%M%S%z 형식의 logical\_date                             | ”20180101T000000+0000”                                                                |
| ts\_nodash                           | %Y%m%dT%H%M%S 형식의 logical\_date                               | ”20180101T000000”                                                                     |
| prev\_data\_interval\_start\_success | 직전 성공한 태스크 스케줄 간격의 시작 날짜 / 시간                                 | “2022-10-23T00:00:00+00:00”                                                           |
| prev\_data\_interval\_end\_success   | 직전 성공한 태스크 스케줄 간격의 종료 날짜 / 시간                                 | “2022-10-24T00:00:00+00:00”                                                           |
| prev\_start\_date\_success           | 직전 성공한 태스크 스케줄 간격의 실제 실행 날짜 / 시간                              | 2022-10-24T06:09:54.429103+00:00                                                      |
| dag                                  | 현재 DAG 개체                                                     | DAG object                                                                            |
| task                                 | 현재 오퍼레이터                                                      | PythonOperator object                                                                 |
| macros                               | airflow.macros 모듈                                             | macro module                                                                          |
| task\_instance                       | 현재 TaskInstance 객체                                            | TaskInstance object                                                                   |
| ti                                   | task\_instance와 동일한 현재 TaskInstance 객체                        | TaskInstance object                                                                   |
| params                               | 태스크 콘텍스트에 대한 사용자 제공 변수                                        | {}                                                                                    |
| var                                  | Airflow 전역에 대한 사용자 지정 변수                                      | https://dydwnsekd.tistory.com/65                                                      |
| var.value.{my\_var}                  | bashoperator에서의 value 형태의 사용자 지정 변수 호출                        | my\_var 변수의 값                                                                         |
| var.json.{my\_var}.{key}             | bashoperator에서의 json 형태의 사용자 지정 변수 호출                         | my\_var 변수의 key 값에 대한 value 값                                                         |
| conn.my\_conn\_id                    |                                                               |                                                                                       |
| task\_instance\_key\_str             | 현재 TaskInstance의 고유 식별자 ({dag\_id}**{task\_id}**{ds\_nodash}) | “dag\_id\_\_task\_id\_\_20190101” “listing\_4\_03\_\_print\_context\_\_20221024”      |
| conf                                 | Airflow 구성                                                    | airflow.configuration. AirflowConfigParser object                                     |
| run\_id                              | DagRun의 run\_id (일반적으로 접두사 + datetime으로 구성된 키)                | “scheduled\_\_2022-10-22T00:00:00+00:00” “manual\_\_2022-10-24T06:09:53.925423+00:00” |
| dag\_run                             | 현재 DagRun 개체                                                  | DagRun object                                                                         |
| test\_mode                           | Airflow가 테스트 모드에서 실행 중인지 여부                                   | True or False                                                                         |

* macros 모듈

| 키                 | 설명                         | 예시                                               |
| ----------------- | -------------------------- | ------------------------------------------------ |
| macros.datetime   | datetime.datetime과 같음      |                                                  |
| macros.timedelta  | datetime.timedelta와 같음     |                                                  |
| macros.dateutil   | datetuil과 같음               |                                                  |
| macros.time       | datetime.time과 같음          |                                                  |
| macros.uuid       | uuid와 같음                   |                                                  |
| macros.random     | random.random과 같음          | 0\~1 사이의 랜덤 float 값                              |
| macros.ds\_add    | YYYY-MM-DD 포멧의 날짜에 일수를 더해줌 | ds\_add('2015-01-01', 5)                         |
| '2015-01-06'      |                            |                                                  |
| macros.ds\_format | 날짜 형식 변환                   | ds\_format('2015-01-01', "%Y-%m-%d", "%m-%d-%y") |
| '01-01-15'        |                            |                                                  |

### PythonOperator 템플릿

* BashOperator는 런타임에 자동으로 템플릿이 지정되는 bash\_command 인수에 문자열을 제공한다.
* 그에 반해 PythonOperator는 런타임 콘텍스트로 템플릿화할 수 있는 인수를 사용치 않고 별도로 런타임 콘텍스트를 적용할 수 있는 python\_callable 인수를 사용한다.
* python\_callable에 오는 것은 문자열이 아니라 함수이기 때문에 함수에서 키워드 인수를 받아서 사용할 수 있다.
* 즉 **함수 내의 코드를 자동으로 템플릿화 할 수 없기 때문**에 직접 인수로 전달해주어야한다.

```python
def templated_test(d1):
    print("{{ ds }}")
    print("ds test:", d1)

# 해당 함수를 PythonOperator로 수행해도 첫번째 프린트문은 '{{ ds }}'만 출력됨
```

```python
def _print_context(****kwargs**): 
    print(context)

def _print_context(****context**): 
    # 키워드 인수인 **kwargs 이름을 태스크 콘텍스트를 저장하려는 의도를 표현하기 위해 context로 변경
    print(context)

print_context = PythonOperator(
    task_id="print_context", 
    python_callable=_print_context, 
    dag=dag
)
```

* 콘텍스트 변수는 모든 콘텍스트 변수의 집합이며 현재 실행되는 태스크의 시작 및 종료 날짜 시간 인쇄와 같이 태스크 실행 간격에 대해 다양한 동작을 제공할 수 있다.

```python
def _print_context(****context**):
    start = context["execution_date"] #context로 부터 execution_date 추출
    end = context["next_execution_date"]
    print(f"Start: {start}, end: {end}")

print_context = PythonOperator(
    task_id="print_context", python_callable=_print_context, dag=dag
)

# 출력 예시:
# Start: 2019-07-13T14:00:00+00:00, end: 2019-07-13T15:00:00+00:00
```

* 명시적으로 context 변수를 키워드 인수로 설정할 수 있다.
* execution\_date라는 인수가 필요하다는 것을 선언하였기 때문에 context 인수에서 캡처되지 않는다.

```python
def _get_data(execution_date, ****context**): #execution_date라는 인수가 필요하다는 것을 선언
    year, month, day, hour, *_=execution_date.timetuple()
		.....

-> context["execution_date"] 처럼 사용하지 않고 바로 execution_date 사용 가능
```

### PythonOperator에 변수 제공

*   PythonOperator는 콜러블 함수에서 추가 인수를 제공하는 방법도 지원한다.

    * 첫 번째 방법 : `op_args` 인수 사용
      *   오퍼레이터를 실행하면 op\_args에 제공된 리스트의 각 값이 콜러블 함수에 전달된다.

          * \_get\_data("/tmp/wikipageviews.gz") 처럼 호출한 것과 동일한 결과

          ```python
          def _get_data(output_path, **context): #output_path라는 인수가 필요하다는 것을 선언
              (생략)

          get_data = PythonOperator(
              task_id="get_data",
              python_callable=_get_data,
              **op_args**=["/tmp/wikipageviews.gz"], #op_args를 사용하여 콜러블 함수에 추가 변수 제공
              dag=dag,
          )
          ```

    ```python
    def _get_data(*output_path, **context): #output_path라는 인수가 필요하다는 것을 선언
        (생략)

    get_data = PythonOperator(
        task_id="get_data",
        python_callable=_get_data,
        **op_args**=["/tmp/wikipageviews.gz","/tmp/wikipageviews1.gz"], #op_args를 사용하여 콜러블 함수에 추가 변수 제공
        dag=dag,
    )
    ```

    * 두 번째 방법 : `op_kwargs` 인수 사용
      *   op\_args와 유사하게 op\_kwargs의 모든 값은 콜러블 함수에 전달되지만, 여기서는 **키워드 인수로 전달**된다.

          * \_get\_data(output\_path = "/tmp/wikipageviews.gz") 처럼 호출한 것과 동일한 결과

          ```python
          def _get_data(output_path, **context): #output_path라는 인수가 필요하다는 것을 선언
              (생략)

          get_data = PythonOperator(
              task_id="get_data",
              python_callable=_get_data,
              **op_kwargs**={"output_path" : "/tmp/wikipageviews.gz"}, #op_kwargs에 주어진 명령어가 호출 가능한 키워드 인수로 전달됨
              dag=dag,
          )
          ```

### 템플릿의 인수 검사하기

* Airflow UI는 템플릿 인수 오류를 디버깅하는데 유용하다.
* Task의 Details의 Rendered Template을 보면 템플릿 인수 값을 검사할 수 있다.

<figure><img src="images/Chapter 4 Airflow 콘텍스트를 사용하여 태스크 탬플릿 작업하기/Untitled 2.png" alt=""><figcaption></figcaption></figure>

<figure><img src="images/Chapter 4 Airflow 콘텍스트를 사용하여 태스크 탬플릿 작업하기/Untitled 3.png" alt=""><figcaption></figcaption></figure>

**UI 확인의 단점**

* 렌더링된 템플릿 보기는 Airflow에서 **작업이 스케줄 될때까지 기다린 후** 확인 가능.

**CLI로 확인하기**

* CLI로 확인하면 작업을 실행하지 않고도 UI에 표시된 것과 동일한 결과를 확인 가능.
* 메타스토어에도 아무것도 등록되지 않음

```bash
airflow tasks render [dag id] [task id] [desired execution date]
ex. airflow tasks render listing_4_03 print_context 2022-10-25T08:27:33.038119+00:00
```

## 4.3 다른 시스템과 연결하기

> 시간별 위키피디아 페이지 뷰를 처리하여 사용하는 사례 진행

<figure><img src="images/Chapter 4 Airflow 콘텍스트를 사용하여 태스크 탬플릿 작업하기/Untitled 4.png" alt=""><figcaption></figcaption></figure>

* 페이지 뷰를 추출하여 Postgres 데이터 베이스에 기록한다.
* 이때, 데이터베이스에 직접 쓰는 작업은 PostgresOperator로 수행한다.

### Airflow가 태스크 간 데이터를 전달하는 방식

1. 메타스토어를 사용 (XCom: 5장에서 다룸)
   1. airflow는 XCom이라는 기본 메커니즘을 제공하여 Airflow 메타스토어에서 pickle 개체를 저장하고 나중에 읽을 수 있다.
   2. airflow의 메타스토어(일반적으로 MySQL 또는 Postgres 데이터베이스)는 크기가 한정되어 있고, 피클링 된 객체는 메타스토어의 blob에 저장되기 때문에 문자열 몇 개 같은 작은 데이터 전송 시에 적용하는 것이 좋다.
2. 영구적인 위치 (ex. 디스크나 데이터베이스)에 태스크 결과를 기록
   1. 큰 데이터를 태스크 간 전송하기 위해 적용하는 것이 좋다.
   2. 향후 더 많은 페이지 처리로 데이터 크기가 커질 수 있을 때도 XCom 대신 디스크에 결과를 저장하는 것이 좋다.

```python
write_to_postgres = PostgresOperator(
    task_id="write_to_postgres",
    postgres_conn_id="my_postgres", # 연결에 사용할 인증 정보의 식별자
    sql="postgres_query.sql", # SQL 쿼리 또는 SQL 쿼리를 포함하는 파일의 경로
    dag=dag,
)
```

* Airflow는 `postgres_conn_id` 등 자격증명 식별자들을 메타스토어에 저장하고 관리하며 운영자가 필요할때 가져와서 쓸 수 있다.

**CLI를 사용하여 자격 증명 등록**

```bash
airflow connections add \
--conn-type postgres \
--conn-host localhost \
--conn-login postgres \
--conn-password mysecretpassword \
my_postgres # <- 연결 식별자
```

그러면 연결이 UI에 생성된다.

**UI에서 자격 증명 등록**

Admin > Connections > `+`

<figure><img src="images/Chapter 4 Airflow 콘텍스트를 사용하여 태스크 탬플릿 작업하기/Untitled 5.png" alt=""><figcaption></figcaption></figure>

### 훅 (Hook)

* PostgresOperator는 Postgres와 통신하기 위해 훅hook이라고 불리는 것을 인스턴스화 한다.
* 인스턴스화된 훅은 연결 생성, Postgres에 쿼리를 전송하고 연결에 대한 종료 작업을 처리한다.
* 여기서 **오퍼레이터는 사용자의 요청을 훅으로 전달하는 작업만 담당**한다.
  * 파이프라인을 구축할 때는 훅은 오퍼레이터 내부에서 동작하기 때문에 신경 쓸 필요는 없다.

<figure><img src="images/Chapter 4 Airflow 콘텍스트를 사용하여 태스크 탬플릿 작업하기/Untitled 6.png" alt=""><figcaption></figcaption></figure>

## 4장 요약

* 오퍼레이터의 일부 인수는 템플릿화 할 수 있다.
* 템플릿 작업은 런타임에 실행된다.
* PythonOperator 템플릿은 다른 오퍼레이터와 다르게 작동한다. 변수를 호출 가능하도록 구성해서 전달해야 한다.
* 템플릿 인수의 결과는 airflow tasks render로 확인할 수 있다.
* 오퍼레이터는 훅을 통해 다른 시스템과 통신할 수 있다.
* 오퍼레이터는 **무엇**을 해야하는지 기술하고, 훅은 작업 **방법**을 결정한다.
