# Oracle Custom Operator Example

## 1. Build Oracle Docker(Container) Image

![](Images/dockerfile_oracle.png)<br>

    1. Input dockerfile path : proj.ora122010
    
    2. Import file instantclient-basiclite-linux.x64-12.2.0.1.0.zip into Repository
    
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

    COPY  instantclient-basiclite-linux.x64-12.2.0.1.0.zip /tmp/ instantclient-basiclite-linux.x64-12.2.0.1.0.zip
    RUN mkdir -p ${GOROOT} && \
         unzip /tmp/ instantclient-basiclite-linux.x64-12.2.0.1.0.zip -d ${GOROOT}

    ARG ORACLIENT=/goroot/instantclient_12_2
    ENV LD_LIBRARY_PATH=${ORACLIENT}/lib:${LD_LIBRARY_PATH}

    RUN python3 -m pip --no-cache install tornado==5.0.2 && \
         python3 -m pip --no-cache install pandas && \
         python3 -m pip --no-cache install cython

    RUN python3 -m pip --no-cache install cx_Oracle

    RUN groupadd -g 1972 vflow && useradd -g 1972 -u 1972 -m vflow
    USER 1972:1972
    WORKDIR /home/vflow
    ENV HOME=/home/vflow
    
    4. Write Tags.json
    {
        "opensuse": "",
        "python36": "",
        "tornado": "5.0.2",
        "oracle": "12.2.0.1.0"
    }

## 2. Oracle Pipeline
### 2-1. Ingest Oracle into Files
![](Images/pipeline_readOracle.png)<br>
Constant Generator --> Python3(Read Oracle) --> To File --> Write File --> Graph Terminator<br>

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


### 2-2. Ingest Files into Oracle
![](Images/pipeline_writeOracle.png)<br>
Read File --> From File --> Python3(Write Oracle) --> Wiretap --> Graph Terminator

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

