# 채용 플랫폼 Log 데이터 분석

## 프로젝트 개요

이 프로젝트는 코드잇 데이터 분석 중급 프로젝트이며 2가지의 기업 데이터 중 주제1. 국내 채용시장 및 채용 플랫폼을 선택하여
프로젝트를 진행하였음. 프로젝트를 진행하는데 앞써 총 팀원은 6인이었으며 본인은 팀장 역할을 수행하였음.
각자 EDA를 진행하며 가설 선정 및 프로덕트 데이터 분석 방법론을 적용하여 프로젝트를 진행하기로 하였으며 본인은 퍼널, 경로 분석 및 
Log Data set을 집중적으로 EDA하며 인사이트를 도출을 담당하였음.

## Step 1. 문제의식
- 제공받은 데이터셋 Job, JobAddress, JobBookmark, Application, company_funding, log(22년, 23년)
  - Job(채용공고 마스터 테이블)
  - JobAddress(채용공고 주소지 정보 테이블)
  - JobBookmark(채용 북마크 트랜젝션 테이블)
  - Application(지원서 마스터 테이블)
  - company_funding(기업 투자 정보 테이블)
  - log (사용자 로그 테이블)
 
- 제공받은 데이터 셋 중 `사용자 로그 테이블`의 경우
  - 채용정보 조회
  - 채용 기업 페이지 조회
  - 채용 기업의 구성원 프로필 조회
  - 지원서 업데이트(개인정보 이슈상, 어떤 내용이 업데이트되었는지는 미제공)
  - 채용공고 북마크
의 데이터들이 제공되었지만 log 데이터셋 내 URL(Path)의 메타데이터가 존재하지 않아 모든 경로는 추측으로 분석을 진행.
우선 Log 데이터셋을 분석하면서 URL Path 마다 임의의 메타데이터를 생성하여 경로를 분석하고자 함.

__임의로 작성한 메타데이터는 발표자료에서 확인 가능합니다.__

## Step 2. 데이터 탐색 및 EDA

한 채용플랫폼의 22년 1월부터 23년 12월까지의 사용자 로그가 저장되어 있음. 
기존 연도별로 분리되어 있는 로그를 하나로 concat하여 사용.

```python
log = pd.concat([log23, log22])

```

```python
log.info()
# Data columns (total 6 columns):
#   Column         Dtype
# ---  ------         -----
#  0   user_uuid      object
#  1   URL            object
#  2   timestamp      object
#  3   date           object
#  4   response_code  int64
#  5   method         object
#  dtypes: int64(1), object(5)

log.shape
# (17241907, 6)
```
| user_id | timestamp          | date       | URL         | response_code | method |
|---------|--------------------|------------|-------------|---------------|--------|
| 유저id  |  로그 생성 시점     | 로그 생성일  |  로그 경로  | HTTP 응답 코드 | HTTP 요청 메소드 |


#### Tiemstamp 전처리
``` python
# UTC 문자열 제거
log['datetime'] = log['timestamp'].str.replace(' UTC','')

#마이크로 초 단위 제거
log['datetime_kst'] = pd.to_datetime(log['datetime'].astype(str).str.split('.').str[0], format="%Y-%m-%d %H:%M:%S")

# 한국 시간(KST)으로 변환 (UTC+9)
log['datetime_kst'] = pd.to_datetime(log['datetime_kst']) + timedelta(hours=9)
```
- Tiemstamp value에 포함되어 있는 UTC 문자열 제거
- 마이크로 초 단위가 포함된 경우와 포함되지 않은 경우가 있었으며 고해상도의 시간은 불필요하다고 판단했으며, 데이터의 용량을 조금이라고 줄이고자 마이크로 초 단위는 제거
- 9시간 더하여 KST 시간에 맞춤
 
#### 결측치 확인
![image](https://github.com/user-attachments/assets/f50d40c8-8d2e-48fe-b147-a67ea70f5b13)

``` python
log.isna().sum()
# user_uuid		  0
# URL	          644156
# timestamp	    0
# date	        0
# response_code	0
# method      	0
# datetime	    0
# datetime_kst	0
```
- URL 컬럼을 제외한 타 컬럼들은 결측치 부재
- 총 1,700만 행 중 644,156 행(3.7%)의 결측치가 존재함

#### 결측치 상세 분석
![image](https://github.com/user-attachments/assets/7c9c10c0-725e-4bdf-9c26-9a4bf8a706d1)
```python
log[(log['response_code'] == 200) & (log['URL'].isna())]
log[(log['response_code'] != 200) & (log['URL'].isna())] 
```
- response_code == 200이면서 URL이 NaN인 경우
  - 행 수: 636,351 rows
  - 의미: 정상 응답인데도 URL이 누락된 경우가 63만여 건.
 
- response_code != 200이면서 URL이 NaN인 경우
  - 행 수: 7,805 rows
  - 의미: 비정상 응답(예: 404, 500 등)에서 URL이 누락된 경우가 7천여 건
 
```python
log[log['URL'].isna()]['datetime_kst'].agg(['count', 'nunique', 'max', 'mean'])
# count        644156             
# nunique      631293            
# min          2022-01-01 00:03:08
# max          2023-12-31 23:45:47
```
![image](https://github.com/user-attachments/assets/dcdf8055-4ad4-4835-b072-3cd161e2cf7e)
- 비정상 응답을 좀 더 세분화해서 보기 위해 응답 코드별로 분리하여 탐색
- 특정 시간에 결측치가 다량으로 발생했을 가능성에 `datetime_kst` 컬럼과 함께 탐색
  - 전체 결측 : 644,156
  - unique : 643,902
  - URL 누락이 특정 시점에 집중된 게 아니라, 시간 전반에 걸쳐 고르게 퍼져 있음.

전체 결측치와 `datetime_kst`의 유의미한 차이를 보이지 않음. 이는 특정 시간대에 결측치가 다량을 발생하지 않았다는 것을 의미한다고 판단

#### 결측처리
- 결측치들을 확잉했을 때 1,700만 행 중 정상 응답이면서 결측인 데이터는 63만 행.
- 63만 행의 결측치가 발생했다는 것에 지속적인 의문이 생김.
- 직접 로그의 흐름을 살펴보며 결측이 발생 이유를 조사하기로 결정

![1팀_중급프로젝트](https://github.com/user-attachments/assets/8be4b241-6fb2-464a-aea5-6158d2cb4702)

