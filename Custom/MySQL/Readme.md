# MySQL Custom Operator 예제

## 1. Build MySQL Docker(Container) Image

![](Images/dockerfile_mysql.png)<br>

    1. Input dockerfile path : proj.z_mysql_sles
    
    2. Write Dockerfile
    # FROM $com.sap.python36
    FROM $com.sap.sles.base
    RUN python3 -m pip install --user pandas
    RUN python3 -m pip install --user numpy
    RUN python3 -m pip install --user sklearn
    RUN python3 -m pip install --user statsmodels
    RUN python3 -m pip install --user pmdarima
    RUN python3 -m pip install --user boto3
    RUN python3 -m pip install --user PyMySQL

    3. Write Tags.json
    {
        "z_mysql_sles": ""
    }

## 2. MySQL Pipeline
### 2-1. Ingest MySQL into Files
![](Images/pipeline_readMySQL.png)<br>
Constant Generator --> Python3(Read MySQL) --> To File --> Write File --> Graph Terminator<br>

    def on_input(data):
        import pymysql
        import pandas as pd

        conn = pymysql.connect(
                user='userid', 
                passwd='userpw', 
                host='xxx.xxx.xxx.xxx', 
                port=31725, 
                db='dbname',
                charset='utf8'
        )

        sql = "select * from QA_EMP"

        cursor = conn.cursor()
        cursor.execute(sql)
        row = cursor.fetchall()
        #print(row)

        df = pd.DataFrame(row)
        #df.columns = ['ID','HALF','FULL']
        #print(df)
        result = df

        cursor.close()
        conn.close()

        csv = result.to_csv(sep=',', index=False)
        api.send("output", csv)

    api.set_port_callback("input", on_input)

### 2-2. Ingest Files into MySQL
![](Images/pipeline_writeMySQL.png)<br>
Read File --> From File --> Python3(Write MySQL) --> Wiretap --> Graph Terminator

    from io import StringIO
    import pandas as pd
    import sqlanydb

    def on_input(msg):

        data = StringIO(msg.body.decode("utf-8"))

        df = pd.read_csv(data, sep=';')
        rows = df.values.tolist()
        #print(rows)

        # IQ
        parms = ("?," * len(rows[0]))[:-1]
        sql = "INSERT INTO runningtimes VALUES (%s)" % (parms)
        #print(sql)

        conn = sqlanydb.connect(uid='User', pwd='Password', eng='EngineName', dbn='DBName', host='xxx.xxx.xxx.xxx:2638')
        cursor = conn.cursor()

        cursor.executemany(sql, rows)

        cursor.close()
        conn.commit()
        conn.close()

        result = {"Number of Rows": str(len(rows))}
        api.send("output1", api.Message(result))

    api.set_port_callback("input1", on_input)

