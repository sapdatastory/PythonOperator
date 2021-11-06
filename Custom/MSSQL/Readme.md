# MSSQL Custom Operator 예제

## 1. Build MSSQL Docker(Container) Image

![](Images/dockerfile_mssql.png)<br>

    1. Input dockerfile path : proj.mssql
    
    2. Load file ora122010.zip into Repository
    
    3. Write Dockerfile
    FROM opensuse/leap:15.1
    ARG GOPATH=/gopath
    ARG GOROOT=/goroot
    ENV GOROOT=${GOROOT}
    ENV GOPATH=${GOPATH}
    ENV PATH=${GOROOT}/bin:${GOPATH}/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

    RUN zypper --non-interactive update && \
         # Install tar, gzip, python3, pip3, gcc and libgthread
          zypper --non-interactive install --no-recommends --force-resolution \
         tar \
         gzip \
         unzip \
         python3 \
         python3-pip \
         python3-devel \
         gcc=7 \
         gcc-c++=7 \
         libgthread-2_0-0=2.54.3 \
         libaio

    #COPY ora122010.zip /tmp/ora122010.zip
    #RUN mkdir -p ${GOROOT} && \
    #     unzip /tmp/ora122010.zip -d ${GOROOT}

    #ARG ORACLIENT=/goroot/instantclient_12_2
    #ENV LD_LIBRARY_PATH=${ORACLIENT}/lib:${LD_LIBRARY_PATH}

    RUN python3 -m pip --no-cache install tornado==5.0.2 && \
         python3 -m pip --no-cache install pandas && \
         python3 -m pip --no-cache install cython

    # MSSQL
    RUN python3 -m pip --no-cache install pymssql

    RUN groupadd -g 1972 vflow && useradd -g 1972 -u 1972 -m vflow
    USER 1972:1972
    WORKDIR /home/vflow
    ENV HOME=/home/vflow

    4. Write Tags.json
    {
        "opensuse": "",
        "python36": "",
        "tornado": "5.0.2",
        "z_mssql": ""
    }

## 2. MSSQL Pipeline
### 2-1. Ingest MSSQL into Files
![](Images/pipeline_readMSSQL.png)<br>
Constant Generator --> Python3(Read MSSQL) --> To File --> Write File --> Graph Terminator<br>

    def on_input(data):
        import cx_Oracle
        import pandas as pd

        connection = cx_Oracle.connect(user="userid", password="userpw",
                                       dsn="xxx.xxx.xxx.xxx:1234/servicename")

        sql = """SELECT *
                 FROM customers"""
        result = pd.read_sql(sql, connection)

        connection.close()

        csv = result.to_csv(sep=',', index=False)
        api.send("output", csv)

    api.set_port_callback("input", on_input)


### 2-2. Ingest Files into MSSQL
![](Images/pipeline_writeMSSQL.png)<br>
Read File --> From File --> Python3(Write MSSQL) --> Wiretap --> Graph Terminator

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

