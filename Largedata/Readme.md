# 대용량 데이터 배치 처리 예제

파일 데이터를 읽어서 Python에서 배치 처리하는 매커니즘에 대해 설명합니다.

<img src="images/largedata_pipeline.png" width="620" height="130"/>

Structured File Comsumer Operator에서 

<img src="images/largedata_1.png" width="500" height="380"/>
<img src="images/largedata_2.png" width="450" height="330"/>
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

<img src="images/largedata_4.png" width="350" height="630"/>
