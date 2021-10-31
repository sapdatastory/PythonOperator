# ABAP Custom Operator Example

## 1. Docker Image

![](Images/abapdockerfile.png)<br>

    1. Input dockerfile path : proj.abap
    
    2. Load file nwrfcsdk.gz into Repository
    
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
         python3 \
         python3-pip \
         # fatal error: Python.h: No such file or directory
         python3-devel \
         #
         gcc=7 \
         gcc-c++=7 \
         libgthread-2_0-0=2.54.3

    COPY nwrfcsdk.gz /tmp/nwrfcsdk.gz
    RUN mkdir -p ${GOROOT} && \
         tar -xvzf /tmp/nwrfcsdk.gz -C ${GOROOT}

    ENV SAPNWRFC_HOME=/goroot/nwrfcsdk
    ENV LD_LIBRARY_PATH=${SAPNWRFC_HOME}/lib:${LD_LIBRARY_PATH}
    ENV PATH=${SAPNWRFC_HOME}/bin:${PATH}

    RUN python3 -m pip --no-cache install tornado==5.0.2 && \
         python3 -m pip --no-cache install pandas && \
         python3 -m pip --no-cache install cython

    RUN python3 -m pip --no-cache install pyrfc

    RUN groupadd -g 1972 vflow && useradd -g 1972 -u 1972 -m vflow
    USER 1972:1972
    WORKDIR /home/vflow
    ENV HOME=/home/vflow
    
    4. Write Tags.json
    {
        "opensuse": "",
        "python36": "",
        "tornado": "5.0.2",
        "abap": "1.0"
    }

## 2. ABAP Pipeline
### 2-1. Read ABAP and Write File
![](Images/readiq.png)<br>
Constant Generator --> Python3(IQ) --> To File --> Write File --> Graph Terminator<br>

    def on_input(data):
        import sqlanydb
        from pandas import DataFrame

        conn = sqlanydb.connect(uid='DBA', pwd='sqlsql', eng='hana1_iqdemo', dbn='iqdemo', host='hana1.pntdemo.kr:26381')
        cursor = conn.cursor()

        sql = "SELECT * FROM runningtimes"
        cursor.execute(sql)

        rowset = cursor.fetchall()

        df = DataFrame(rowset)
        df.columns = ['ID','HALF','FULL']
        #print(df)
        result = df

        cursor.close()
        #conn.commit()
        conn.close()

        csv = result.to_csv(sep=',', index=False)

        api.send("output", csv)

    api.set_port_callback("input", on_input)

### 2-2. Read File and Write Oracle
![](Images/writeiq.png)<br>
Read File --> From File --> Python3(IQ) --> Wiretap --> Graph Terminator

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

        conn = sqlanydb.connect(uid='DBA', pwd='sqlsql', eng='hana1_iqdemo', dbn='iqdemo', host='hana1.pntdemo.kr:26381')
        cursor = conn.cursor()

        cursor.executemany(sql, rows)

        cursor.close()
        conn.commit()
        conn.close()

        result = {"Number of Rows": str(len(rows))}
        api.send("output1", api.Message(result))

    api.set_port_callback("input1", on_input)

