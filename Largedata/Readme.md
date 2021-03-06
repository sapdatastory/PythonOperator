# 대용량 데이터의 배치 처리 매커니즘

대용량 파일 데이터의 배치 처리 파이프라인<br>
<img src="images/largedata_pipeline.png" width="620" height="130"/>

Structured File Comsumer Operator<br>
User Defined Fetch Size를 False(default)로 설정하면, 실행 시 자동으로 10 ~ 1000 사이로 Batch Size를 설정합니다.<br>
만약, User Defined Fetch Size를 True로 설정하면, Maxium Fetch Size 값을 사용하여 실행합니다.<br>

<img src="images/largedata_1.png" width="500" height="380"/>

Table to Message Coverter Operator<br>
Force batch size를 True로 설정하면, Batch size 값을 사용하여 실행합니다.<br>

<img src="images/largedata_2.png" width="420" height="330"/>

Python3 Operator 스크립트<br>
input1 포트로 입력되는 Batch Size의 크기(Dataframe.shape)를 출력합니다.<br>

<img src="images/largedata_3.png" width="750" height="380"/>

```shell
import pandas as pd
import io

df = pd.DataFrame()

def on_input(data):
    global df
    
    body = io.StringIO(data.body)
    dfchunks = pd.read_csv(body, sep=",", header=0)

    api.send("output1", str(dfchunks.shape))
    
    df = df.append(dfchunks, ignore_index=True)
    
    if data.attributes['message.lastBatch']:
        api.send("output1", str(df.shape))
    
api.set_port_callback("input1", on_input)
```

Wiretap Oprator<br>
Python3 Operator 포트에서 출력되는 값을 표시합니다.<br>

<img src="images/largedata_4.png" width="450" height="630"/>

