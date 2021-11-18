# 대용량 데이터 배치 처리 예제

<img src="images/largedata_pipeline.png" width="550" height="130"/>

<img src="images/largedata_1.png" width="550" height="130"/>
<img src="images/largedata_2" width="550" height="130"/>
<img src="images/largedata_3" width="550" height="130"/>
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
<img src="images/largedata_4" width="550" height="130"/>
