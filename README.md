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

#### Log 분리
![image](https://github.com/user-attachments/assets/02f22b0f-d144-4752-9104-fbdaa2177000)
- 해당 시각화는 URL을 '/' 경로별로 분리 후 첫번째 경로를 barplot으로 시각화한 이미지이다. 첫번째 경로를 통해 의미를 추측해보아 API 요청으로 API는 사용자가 프론트엔드를 거치지 않고 곧바로 백엔드에 정보를 호출하는 것이기 때문에 사용자 행동 분석 등의 의미가 희석될 수 있을 것이라고 판단되어 임의의로 API와 유저 행동 로그로 분리할 필요성이 있다고 생각해 로그를 API, 유저 행동 로그로 분리하여 분석을 진행. 
- API로그를 통해 유저 행동을 분석하는 것은 무리가 있어 대부분은 유저 행동 로그로 분석을 진행

#### 세션 부여
- 제공받는 사용자 로그 데이터 셋에는 세션 컬럼이 부재하여 임의의로 세션을 부여하여 분석을 진행
- 사용자별 세션 부여
  - 웹, 앱 등 서비스에 따라 세션 적정 시간이 다름
  - GA(Google Analytics)와 같은 웹 분석 도구에서는 30분으로 선정하기 때문에 이에 맞춰 유저별 세션은 30분으로 부여

```

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

  
#### 경로 추적
![1팀_중급프로젝트](https://github.com/user-attachments/assets/8be4b241-6fb2-464a-aea5-6158d2cb4702)

- 한 유저의 경로를 추척해본 결과 결측치에서 약 10초의 시간 뒤 user_id 경로로 이동, 다시 15초 뒤 결측치로 이동 하는 것을 확인
- 이는 결측치가 아닌 어떠한 페이지 일 수 있을 것이라고 판단.
- user_id는 프로필/프로필 보기의 페이지라고 가정하고 job은 채용공고 페이지라고 가정했을 때
- NaN은 높은 확률로 메인페이지일 수도 있을 것이라고 판단.

#### 메인페이지 경로 시각화 (생키 차트) 
![newplot (3)](https://github.com/user-attachments/assets/64c4356d-532d-487d-9fff-d31d83c4ee4d)
```
# 메인페이지로 가정 후 경로 분석
main_page → @user_id                                      
main_page                                                 
main_page → jobs                                          
main_page → main_page                                     
main_page → @user_id/resume                               
main_page → setting                                        
main_page → @user_id → @user_id → @user_id                 
main_page → @user_id/applications                          
main_page → @user_id → @user_id                            
main_page → @user_id → @user_id → @user_id → @user_id      
main_page → main_page → main_page                          
main_page → @user_id → main_page → @user_id                
main_page → jobs → jobs/id/id_title                        
main_page → @user_id → main_page → jobs                    
main_page → @user_id/applications → @user_id/resume        
main_page → jobs → jobs/id/id_title → jobs/id/id_title     
main_page → @user_id → main_page → main_page               
main_page → @user_id → main_page                           
main_page → setting → setting                              
main_page → @user_id → @user_id/job_offer/received         
```
- 데이터를 제공해준 채용플랫폼에서 URL에 대한 메타데이터를 제공받지 못해 추측으로 경로 분석을 진행하였지만 해당 경로들로 보았을 때 메인페이지로 보는 것이 합당하다고 판단됨
- 이를 통해 인사이트 및 가설을 만들어낼 수는 없었지만 결측치를 제거, 데이터 대체(중앙값, 평균값) 등으로 처리하는 것만 아니라 직접 데이터를 추적하여 결측치를 보존할 수 있는 방법이 있다는 것을 배울 수 있었음.

## Step 3. 퍼널 분석
# 사용자별 상위 20개 전환 경로 분석 

```
companies/company_id/jobs -> jobs/id/apply: 179824회
companies/company_id -> companies/company_id/jobs: 123740회
jobs/id/apply/step1 -> jobs/id/apply/step2: 120587회
jobs/id/apply -> @user_id: 115988회
@user_id -> companies/company_id: 108536회
jobs/id/apply -> jobs/id/apply/step1: 93972회
companies/company_id -> @user_id: 91344회
jobs -> jobs/id/apply: 91148회
jobs/id/apply -> companies/company_id/jobs: 90709회
jobs/id/apply/step2 -> jobs/id/apply/step3: 86667회
... 생략
```

- 가장 먼저 눈에 들어오는 경로는 apply와 하위 경로 step1 ~ 4
- 채용 플랫폼 답게 지원서 작성 페이지가 많이 호출되고 있음을 확인.
- 단계별로 이루어진 페이지이기에 퍼널 분석이 적합하다고 판단됨.

![newplot (5)](https://github.com/user-attachments/assets/b4ba466e-e66b-4f67-a4fd-68d210c2e4ab)


  
