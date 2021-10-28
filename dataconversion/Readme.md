# Python Operator에서 Data Type 처리 예제

### 1. File to Python 처리
#### 1.1 message Data Type
![](/dataconversion/images/1.FilePython.png)<br>

    from io import StringIO
    import pandas as pd

    def on_input(msg):

        data = StringIO(msg.body.decode("utf-8"))

        df = pd.read_csv(data, sep=';')
        result = df
        csv = result.to_csv(sep=',', index=False)

        api.send("output1", api.Message(attributes=msg.attributes, body=csv))

    api.set_port_callback("input1", on_input)

#### 1.2 string Data Type
![](/dataconversion/images/2.FilePython.png)<br>

    from io import StringIO
    import pandas as pd

    def on_input(msg):

        data = StringIO(msg)

        df = pd.read_csv(data, sep=';')
        result = df
        csv = result.to_csv(sep=',', index=False)

        api.send("output1", csv)

    api.set_port_callback("input1", on_input)

### 2. HANA to Python 처리

#### 2.1 Table Consumer Operator - table Type
![](/dataconversion/images/3.HanaPython.png)<br>
    
    from io import StringIO
    import pandas as pd

    def on_input(msg):

        data = StringIO(msg.body)

        df = pd.read_csv(data, sep=',')
        result = df
        csv = result.to_csv(sep=',', index=False)

        api.send("output1", api.Message(attributes=msg.attributes, body=csv))

    api.set_port_callback("input1", on_input)

#### 2.2 HANA Client Operator - message Type
![](/dataconversion/images/4.HanaPython.png)<br>
    
    from io import StringIO
    import pandas as pd

    def on_input(msg):

        data = pd.DataFrame(msg.body)

        #df = pd.read_csv(data, sep=',')
        result = data
        csv = result.to_csv(sep=',', index=False)

        api.send("output1", api.Message(attributes=msg.attributes, body=csv))

    api.set_port_callback("input1", on_input)

#### 2.3 HANA Client Operator - message.table Type
![](/dataconversion/images/5.HanaPython.png)<br>

from io import StringIO
import pandas as pd

        def on_input(msg):

            #data = StringIO(msg.body.decode("utf-8"))
            #data = StringIO(msg.body)
            data = pd.DataFrame(msg.body)

            #df = pd.read_csv(data, sep=',')
            result = data
            csv = result.to_csv(sep=',', index=False)

            api.send("output1", api.Message(attributes=msg.attributes, body=csv))
            #api.send("output1", csv)

        api.set_port_callback("input1", on_input)

