# Chapter 3. Airflow의 스케줄링

## 3.1 예시: 사용자 이벤트 처리하기

- 시뮬레이션 할 예제 서비스 설명
    - 웹 사이트에서 사용자 동작을 추적하고 사용자가 웹사이트에서 액세스한 페이지를 분석할 수 있는 서비스
    - 사용자들이 접근하는 페이지들과 해당 페이지에서 소비한 시간을 확인
    - 시간이 지남에 따른 사용자 행동 변화를 추적하기 위해 데일리로 통계량 계산

**예제를 위한 API**

```bash
curl -o /tmp/events.json http://10.10.200.9:5000/events
```

해당 api 호출 시 지난 30일 동안의 모든 이벤트 목록 반환

**두 가지 워크 플로우**

1. 사용자 이벤트 가져오기 - BashOperator
2. 통계 및 계산 - PythonOperator

## 3.2 정기적으로 실행하기

```python
dag = DAG(
    dag_id="01_unscheduled", 
		start_date=datetime(2019, 1, 1), # DAG의 시작 날짜
		schedule_interval=None # 스케줄되지 않는 DAG으로 지정 (DAG 트리거를 통해 수동 실행)
)
```

- schedule_interval=None :  디폴트 값
- UI 또는 API를 통해서 수동으로 트리거하면 실행

### 스케줄 간격 정의하기

- schedule_interval="@daily": 매일 자정에 실행하도록 예약
- **정의된 간격 후에 태스크가 시작됨**에 유의
    - ex) start_date=2019-01-01 이고 간격이 @daily 라면 1월 1일 이후 매일 자정에 DAG를 실행하게 된다.
    - 즉, 2019-01-02 자정에 처음 DAG이 실행되고, 2019-01-01에는 실행되지 않음
    - [https://blog.bsk.im/2021/03/21/apache-airflow-aip-39/](https://blog.bsk.im/2021/03/21/apache-airflow-aip-39/) 참고 블로그
    
    <img src="images/Chapter 3 Airflow의 스케줄링/Untitled.png">
    
- 이론적으로는 DAG를 끄기 전까지 영원히 실행
- 기간이 정해진 프로젝트와 같은 종료 일이 필요한 프로세스는 end_date를 주어 특정 날짜 이후에 실행 중지되도록 설정
    - ex) start_date=2019-01-01 , end_date=2019-01-05
        - (여담) 보편적인 회사 업무에서는 start_date만 넣고, end_date를 넣어 실행하는 작업은 없다고 함
    - **Q. 간격이 @daily라서 5일 자정으로부터 하루 간격 뒤인 6일에 실행되고 끝나는것?**
    - → **정의된 간격 후에 태스크가 시작되기 때문에 end_date인 1/5의 작업은 1/6 0시에 실행되고 끝나는 것**

<img src="images/Chapter 3 Airflow의 스케줄링/Untitled 1.png">

### Cron 기반의 스케줄 간격 설정

```bash
# ┌─────── 분 (0 - 59)
# │ ┌────── 시간 (0 - 23)
# │ │ ┌───── 일 (1 - 31)
# │ │ │ ┌───── 월 (1 - 12)
# │ │ │ │ ┌──── 요일 (0 - 6) (일요일 ~ 토요일; 일부 시스템에서 7은 일요일이다.)
# │ │ │ │ │      
# * * * * *

dag = DAG(
    dag_id="04_time_delta",
    schedule_interval='0 0 * * *' 
    #매일 0시에 실행하라는 의미 (=@daily =datetime.timedelta(days=1))
    start_date=dt.datetime(year=2019, month=1, day=1),
    end_date=dt.datetime(year=2019, month=1, day=5),
)
```

**Airflow가 지원하는 다른 매크로**

- @None : 외부적으로 트리거된 DAG을 통해 실행
- @once : 1회만 실행하도록 스케줄 (start_date에 맞춰 실행되고, end_date는 무엇을 주던 상관 x)
- @hourly : 매시간 1회 실행 (cron = ‘0 * * * *’)
- @daily : 매일 자정에 1회 실행 (cron = ‘0 0 * * *’)
- @weekly : 매주 일요일 자정에 1회 실행 (cron = ‘0 0 * * 0’)
- @monthly : 매월 1일 자정에 1회 실행 (cron = ‘0 0 1 * *’)
- @yearly : 매년 1월 1일 자정에 1회 실행 (cron = ‘0 0 1 1 *’)

### 빈도 기반의 스케줄 간격 설정하기

- 3일마다 실행하려면 cron식으로는 한계가 있음
- timedelta 인스턴스를 사용

```python
dag = DAG(
    dag_id="04_time_delta",
    schedule_interval=dt.timedelta(days=3),
    start_date=dt.datetime(year=2019, month=1, day=1),
    end_date=dt.datetime(year=2019, month=1, day=5),
)
```

- 다음과 같이 설정하면 DAG가 시작 시간으로 부터 3일마다 실행된다
    - ex. start_date가 2019/01/01이기 때문에 2019년 1월 4일, 7일, 10일 …)
    - dt.timedelta(minutes=10) : 10분마다, dt.timedelta(hours=2) : 2시간마다

## 3.3 데이터 증분 처리하기

> **데이터 증분 처리** : 일정 시간마다 새롭게 추가된 데이터만 처리
> 

**문제**

1. DAG이 매일 사용자 이벤트 전체에 대해 다운로드하고 계산하는 것은 비효율적
2. 지난 30일 동안의 이벤트만 다운로드 → 그 이상 과거 특정 시점의 이벤트는 존재하지 않음

**대안**

1. 데이터를 순차적으로 가져오도록 DAG을 변경
2. 스케줄 간격에 해당하는 일자의 이벤트만 로드하고 새 이벤트만 통계를 계산

<img src="images/Chapter 3 Airflow의 스케줄링/Untitled 2.png">

> 즉, API 호출 시 특정 날짜에 대한 이벤트를 가져오도록 조정한다.
> 

### 실행 날짜를 사용하여 동적 시간 참조하기

- Airflow는 태스크가 실행되는 특정 간격을 정의할 수 있는 매개변수를 제공함
- `execution_date` : 스케줄 간격으로 실행되는 시작 시간을 나타내는 타임스탬프 (DAG를 시작하는 시간의 특정 날짜가 아님!)
- `next_execution_date` : 스케줄 간격 종료 시간

<img src="images/Chapter 3 Airflow의 스케줄링/Untitled 3.png">

**ex) 쇼핑몰을 운영하고 있다고 가정**

2021년 1월 3일의 일일 매출을 계산하고 싶다면, 3일이 끝나고 하루가 넘어가는 1월 4일 자정(0시)에 **01/03 00:00:00 ~ 01/03 23:59:59**에 발생한 매출을 계산해야 한다. 

여기서 작업 대상이 되는 데이터는 1월 3일자 데이터(=`execution_date`)가 되고, 실제 작업이 실행되는 시간은 1월 4일 0시가 된다.

**특정 날짜 지정을 위해 템플릿 사용하기**

```python
fetch_events = BashOperator(
    task_id="fetch_events",
    bash_command=(
         "mkdir -p /data && "
         "curl -o /data/events.json "
         "http://localhost:5000/events?"
         "start_date=**{{execution_date.strftime('%Y-%m-%d')}}**"  # Jinja 템플릿으로 형식화된 execution_date 삽입
         "&end_date=**{{next_execution_date.strftime('%Y-%m-%d')}}**" # next_execution_date로 다음 실행 간격의 날짜를 정의
    ),
    dag=dag,
)
```

- {{variable_name}} 구문은 Airflow의 Jinja 템플릿을 사용하는 예임 (4장에서 자세히)
- 해당 구문을 사용해서 DAG 실행 날짜를 참조하고 문자열 형식으로 반환 형식을 지정
- 더욱 축약된 매개변수도 제공함
- `ds` : execution_date를 YYYY-MM-DD로 반환
- `ds_nodash` : execution_date를 YYYYMMDD로 반환
    - 즉, `{{execution_date.strftime('%Y-%m-%d')}}` == `{{ds}}`

```python
fetch_events = BashOperator(
    task_id="fetch_events",
    bash_command=(
         "mkdir -p /data && "
         "curl -o /data/events.json "
         "http://localhost:5000/events?"
         "start_date=**{{ds}}**"  # ds는 YYYY-MM-DD 형식의 execution_date 제공
         "&end_date=**{{next_ds}}**" # next_ds는 next_execution_date에 대해 YYYY-MM-DD 형식으로 제공
    ),
    dag=dag,
)
```

### 데이터 파티셔닝

- 파티셔닝 : 데이터 세트를 더 작고 관리하기 쉬운 조각으로 나누는 작업. 데이터 저장 및 처리 시스템에서 일반적인 전략
- 파티션 : 데이터 세트의 작은 부분

```python
fetch_events = BashOperator(
    task_id="fetch_events",
    bash_command=(
        "mkdir -p /data/events && "
        "curl -o /data/events/**{{ds}}.json** " # 날짜별로 개별 파일에 이벤트 데이터 쓰기
        "http://events_api:5000/events?"
        "start_date={{ds}}&"
        "end_date={{next_ds}}"
    ),
    dag=dag,
)

def _calculate_stats(****context**): # 모든 콘텍스트 변수 수신
    """Calculates event statistics."""
    input_path = context["templates_dict"]["input_path"]
    output_path = context["templates_dict"]["output_path"]

    events = pd.read_json(input_path)
    stats = events.groupby(["date", "user"]).size().reset_index()

    Path(output_path).parent.mkdir(exist_ok=True)
    stats.to_csv(output_path, index=False)

calculate_stats = PythonOperator(
    task_id="calculate_stats",
    python_callable=_calculate_stats,
    **templates_dict**={  # 템플릿 되는 값을 전달
        "input_path": "/data/events/{{ds}}.json",
        "output_path": "/data/stats/{{ds}}.csv",
    },
    # Required in Airflow 1.10 to access templates_dict, deprecated in Airflow 2+.
    # provide_context=True,
    dag=dag,
)
```

- 실행 날짜별로 다운로드 되는 데이터를 각각 따로 저장
- 데이터를 로드하고 통계내는 부분도 전체가 아닌 파티션 데이터셋에 대해서만 처리하도록
입출력 경로를 변경해준다.
- PythonOperator에서 템플릿을 구현하기 위해 오퍼레이터의 templates_dict 매개변수를 사용하여 템플릿화해야 하는 모든 인수를 전달해야 한다.
- 그 후, Airflow에 의해 _ calculate_stats 함수에 전달된 콘텍스트 개체(**context)에서 함수 내부의 템플릿 값을 확인할 수 있다.

## 3.4 Airflow 실행 날짜 이해

- 시점 기반(cron), 간격기반(airflow)

### 고정된 스케줄 간격으로 태스크 실행

<img src="images/Chapter 3 Airflow의 스케줄링/Untitled 4.png">

- 간격 기반의 시간 표현에서는 해당 간격의 시간 슬롯이 경과되자마자 해당 간격동안 DAG가 실행된다.
    - 첫 번째 간격은 2019-01-01 23:59:59 직후 실행
    - 두 번째 간격은 2019-01-02 23:59:59 직후 실행
- 간격 기반 접근은 작업이 실행되는 시간 간격(시작 및 끝)을 정확히 알고 있기 때문에 증분 데이터 처리 유형을 수행하는 데 적합하다.
- `execution_date` 는 DAG이 실제 실행되는 시점이 아니라 **예약된 간격의 시작 날짜**
    - 변수명으로 인해 실행 날짜라고 오해하기 쉬움
    - 실제 실행되는 시점이 1/4이면 전날의 간격을 실행하고 있기 때문에 execution_date는 1/3

<img src="images/Chapter 3 Airflow의 스케줄링/Untitled 5.png">

- execution_date는 현재 간격의 시작
- previous_exection_date는 이전 간격의 시작
- next_exection_date는 다음 간격의 시작(=이번 간격의 끝)
- 따라서 현재 간격은 execution_date은 next_execution_date의 조합

## 3.5 과거 데이터 간격을 메꾸기 위해 백필 사용하기

- Airflow는 기본적으로 아직 실행되지 않은 **과거 스케줄 간격을 예약하고 실행**
- 과거 시점 태스크 실행을 피하려면 `catchup=False`

```python
dag = DAG(
    dag_id="09_no_catchup",
    schedule_interval="@daily",
    start_date=dt.datetime(year=2019, month=1, day=1),
    end_date=dt.datetime(year=2019, month=1, day=5),
    **catchup=False**,
)
```

- 가장 최신 스케줄 간격부터 작업 처리하게됨
    - Catchup = True 일 때는 과거의 스케줄 간격을 포함하여 작업 처리를 시작한다 ( = 백필)
    - Catchup = False 일 때는 현재 스케줄 간격부터 작업 처리를 시작한다.

(Chapter%203%20Airflow%E1%84%8B%E1%85%B4%20%E1%84%89%E1%85%B3%E1%84%8F%E1%85%A6%E1%84%8C%E1%85%AE%E1%86%AF%E1%84%85%E1%85%B5%E1%86%BC%209db4be77795b42a5b88517004855f95a/Untitled%206.png)

- ex. 2주, 즉 14일 동안 DAG를 실행한다고 가정해보자.
- 1일 수행 후 2일 차에서 멈추고, 5일 차에서 다시 수행하면`Catchup = True`로 설정한 경우 **수행되지 않았던(혹은 지워진)** 3, 4, 5일차의 DAG RUNs이 모두 실행된다. 이를 원하지 않는 경우 `Catchup = False`로 설정해야 한다.
- 백필링과 재실행 태스크는 자원에 많은 부하를 유발

## 3.6 태스크 디자인을 위한 모범 사례

### 원자성

> **원자성 트랜잭션: 모두 발생하거나 전혀 발생하지 않는다.**
> 
- 하나의 태스크 안에서 두 개의 작업은 원자성을 무너뜨리므로 다수 태스크로 분리하도록 한다.
- 두 태스크 사이에 강한 의존성이 있다면 단일 태스크 내에서 두 작업을 모두 유지하여 하나의 일관된 태스크 단위를 형성하는 것이 나을 수 도 있다.
    - ex. api 인증 토큰 발급 → api 호출 하는 경우

### 멱등성

> **멱등성: 동일한 입력으로 동일한 태스크를 여러 번 호출해도 전체 결과에 효력이 없어야 한다.**
> 
- ex) 동일한 날짜의 데이터를 불러와서 동일한 처리를 하고 기존 파일에 동일한 결과를 덮어쓰며 저장
- 멱등성은 일관성과 장애 처리를 보장

## 3장 요약

- 스케줄 간격을 설정하여 스케줄 간격으로 DAG를 실행할 수 있다.
- 하나의 스케줄 간격 작업은 해당 주기의 작업이 끝나면 시작한다.
- 스케줄 간격은 cron 및 timedelta 식을 사용하여 구성한다.
- 템플릿을 사용하여 변수를 동적으로 할당해 데이터 증분 처리가 가능하다.
- 실행 날짜는 실제 실행 시간이 아니라 스케줄 간격의 시작 날짜를 말하는 것이다.
- DAG는 백필을 통해 과거의 특정 시점에 작업을 실행할 수 있다.
- 멱등성은 동일한 출력 결과를 생성하면서 작업을 재실행할 수 있도록 보장한다.