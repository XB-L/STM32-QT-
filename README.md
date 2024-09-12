<!--
作者: 嵌入式实验室
时间: 2024年9月11日
-->

# QT上位机程序设计

## 1. 介绍
![Alt text](1.png)

## 2. 数据传输格式
通信的数据格式为JSON，QT自带JSON数据解析类，STM32需要向上位机发送指定JSON格式的数据如：`"{ \"name\": \"John\", \"age\": 30, \"city\": \"New York\" }"`。  QT通过解析即可得到每个标识符对应的数据。


以下是QT解析JSON数据示例，其中str为串口接收到的字符串数据。

```cpp
void Widget::GetDataFromMCU(QString str)
{
    QJsonDocument jsonDoc = QJsonDocument::fromJson(str.toUtf8());
    if (!jsonDoc.isNull()) {
        if (jsonDoc.isObject()) {
            QJsonObject jsonObj = jsonDoc.object();
            Data->MotorLeftBack = jsonObj["a"].toInt();
            Data->MotorLeftFront = jsonObj["b"].toInt();
            Data->MotorRightBack = jsonObj["c"].toInt();
            Data->MotorRightFront = jsonObj["d"].toInt();
            Data->RealTimeDtc = jsonObj["e"].toInt();
            Data->TrackLine1 = jsonObj["f"].toInt();
            Data->TrackLine2 = jsonObj["g"].toInt();
            Data->TrackLine3 = jsonObj["h"].toInt();
            Data->TrackLine4 = jsonObj["i"].toInt();
            qDebug() << "MotorLeftBack:" << Data->MotorLeftBack;
            qDebug() << "MotorLeftFront:" << Data->MotorLeftFront;
            qDebug() << "MotorRightBack:" << Data->MotorRightBack;
            qDebug() << "MotorRightFront:" << Data->MotorRightFront;
            qDebug() << "RealTimeDtc:" << Data->RealTimeDtc;
            qDebug() << "TrackLine1:" << Data->TrackLine1;
            qDebug() << "TrackLine2:" << Data->TrackLine2;
            qDebug() << "TrackLine3:" << Data->TrackLine3;
            qDebug() << "TrackLine4:" << Data->TrackLine4;
        } else {
            qDebug() << "JSON document is not an object.";
        }
    } else {
        qDebug() << "Failed to parse JSON.";
    }
    m->SetMCUData(Data);
}
```

## 3. STM32需要向上位机发送的数据
``` cpp
typedef struct DataFromMCU
{
    int MotorLeftFront;   //b
    int MotorRightFront;  //d
    int MotorLeftBack;    //a
    int MotorRightBack;   //c
    int RealTimeDtc; //e
    int TrackLine1;  //f
    int TrackLine2;  //g
    int TrackLine3;  //h
    int TrackLine4;  //i
}DataFromMCU,*PDataFromMCU;


```
为了提高数据传输效率，用一个标识符代表qt中的每个控件的数据，比如STM32发送`printf("{ \"a\": 30, \"b\": 31, \"c\": 32, \"d\": 33, \"e\": 11, \"f\": 1, \"g\": 0, \"h\": 1, \"i\": 0 }");`
则QT界面显示为：
![Alt text](image.png)

## 4. 上位机需要向STM32发送的数据

``` cpp
typedef struct DataToMCU
{
    int CtlFlag;//1 遥控，2跟随，3循迹  a
    int GoFlag;//1 前进， 2右移， 3后退，4左移，5左转，6右转  b
    int SeekBar;//超声波跟随距离 c
}DataToMCU,*PDataToMCU;
```
> 同样采用JSON数据格式，为了提高数据传输效率，以a,b,c依次代表这三个变量，QT中使用定时器每隔100ms向STM32发送这三个数据。



以下是QT使用定时器每100ms向STM32发送JSON格式数据示例代码

``` cpp
void Widget::SendDataToMCU()
{
    static int cnt = 0;
    PDataToMCU data = new DataToMCU;
    data = m->GetQTData();

    // 创建一个定时器
    QTimer *timer = new QTimer(this);

    connect(timer, &QTimer::timeout, this, [this, data]() {
        // 将数据转换为 JSON 格式
        QJsonObject jsonObj;
        jsonObj["a"] = data->CtlFlag;
        jsonObj["b"] = data->GoFlag;
        jsonObj["c"] = data->SeekBar;

        QJsonDocument jsonDoc(jsonObj);
        QString jsonString = jsonDoc.toJson(QJsonDocument::Compact);
        jsonString.append('\n'); // 添加换行符
        qDebug()<<jsonString<<endl;
        // 向串口发送数据
        if (serilaPort->isOpen()) {
            serilaPort->write(jsonString.toUtf8());
        } else {
            qDebug() << "Serial port is not open.";
        }
    });
    timer->start(100);
}
``` 

## 5. STM32中断服务函数：

```cpp
#define BUFFER_SIZE 1024 // 定义缓冲区大小
static char receiveBuffer[BUFFER_SIZE]; // 接收缓冲区
static int bufferIndex = 0; // 缓冲区索引
int CtlFlag = 0;
int GoFlag = 0;
int SeekBar = 0;

int USART2_IRQHandler(void)
{	
    if (USART_GetITStatus(USART2, USART_IT_RXNE) != RESET) // 判断是否接收到数据
    {	      
        char receivedChar = USART2->DR; // 读取数据

        // 如果接收到换行符，表示数据包结束
        if (receivedChar == '\n')
        {
            receiveBuffer[bufferIndex] = '\0'; // 添加字符串结束符
            bufferIndex = 0; // 重置缓冲区索引
            // 解析 JSON 数据
            if (strncmp(receiveBuffer, "{\"a\":", 5) == 0)
            {
                char *ptr = receiveBuffer + 5;
                CtlFlag = atoi(ptr);
                ptr = strstr(ptr, ",\"b\":") + 5;
                GoFlag = atoi(ptr);
                ptr = strstr(ptr, ",\"c\":") + 5;
                SeekBar = atoi(ptr);
            }
        }
        else
        {
            // 将接收到的字符添加到缓冲区
            if (bufferIndex < BUFFER_SIZE - 1)
            {
                receiveBuffer[bufferIndex++] = receivedChar;
            }
        }
    }
    return 0;	
}
```
