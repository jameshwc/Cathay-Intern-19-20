# AWS Lambda (with SES and CloudWatch Logs)

## Lambda with SES
> 在Lambda裡使用Python （當然，其他語言亦可）和SES服務寄信通知

+ 要先在SES裡驗證收信人和發信人的信箱
![](https://i.imgur.com/zeBdhbo.png)

+ 在Lambda頁面設置好環境變數（MYAWS_REGION、MYAWS_ACCESS_KEY_ID、MYAWS_SECRET_ACCESS_KEY、RECIPIENT、SENDER）
+ 如果手上的IAM都沒有SES的權限，只有自己有的話，可以在*call boto3.client*時加入ACCESS_KEY_ID和SECRET_ACCESS_KEY。如果沒加，會帶入Lambda預設的IAM。
    ![](https://i.imgur.com/2XcroJ0.png)
    - 這裡有一個坑，環境變數名字不能命名為*AWS_ACCESS_KEY_ID*或*AWS_SECRET_ACCESS_KEY*，因為這恰好是Lambda會幫忙填好的資訊（即預設的Lambda IAM），同名的結果會導致讀不到其他環境變數（原因未知，但不要撞名就對了）


```python
import os
import json
import boto3
from botocore.exceptions import ClientError

def send_mail(subject:str, content:str, defaultIAM:bool=False):
    SENDER = os.environ["SENDER"]
    RECIPIENT = os.environ["RECIPIENT"]
    AWS_REGION = os.environ["AWS_REGION"]
    SUBJECT = subject
    CHARSET = "utf-8"
    if not defaultIAM:
        AWS_ACCESS_KEY_ID = os.environ["AWS_ACCESS_KEY_ID"]
        AWS_SECRET_ACCESS_KEY = os.environ["AWS_SECRET_ACCESS_KEY"]
        client = boto3.client('ses',region_name=AWS_REGION, aws_access_key_id=AWS_ACCESS_KEY_ID, aws_secret_access_key=AWS_SECRET_ACCESS_KEY)
    else:
        client = boto3.client('ses', region_name=AWS_REGION)
    try:
        response = client.send_email(
            Destination={
                'ToAddresses': [
                    RECIPIENT,
                ],
            },
            Message={
                'Body': {
                    'Html': {
                        'Charset': CHARSET,
                        'Data': content,
                    },
                    'Text': {
                        'Charset': CHARSET,
                        'Data': content,
                    },
                },
                'Subject': {
                    'Charset': CHARSET,
                    'Data': SUBJECT,
                },
            },
            Source=SENDER,
        )
    except ClientError as e:
        print(e.response['Error']['Message'])
    else:
        print("Email sent! Message ID:"),
        print(response['MessageId'])
    
def lambda_handler(event, context=None):
    send_mail("SES test", str(event))
```
*send_mail* 改寫自[doc](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/send-using-sdk-python.html)

## Lambda with Slack
+ 在Lambda頁面設置好環境變數（SLACK_URL）
![](https://i.imgur.com/czU1w1O.png)
+ 因呼叫slack API會使用requests這個library，但其並非python內建之函式庫，也沒辦法pip install，解決方案有二：
    + ```from botocore.vendored import requests```，其用法和requests一樣，但該函式庫[已在19年10月被移除](https://github.com/boto/botocore/pull/1829)，故最新的python3.8並不支援，只能使用python3.6/3.7
    + ```pip install -t requests .```再將requests的包和其他程式碼打包成zip上傳到Lambda
```python
import json
from botocore.vendored import requests # or import requests
import os

def slack_notify(url:str, content:str):
    dict_headers = {'Content-type': 'application/json'}
    dict_payload = {'text': content}
    json_payload = json.dumps(dict_payload)
    resp = requests.post(url, data=json_payload, headers=dict_headers)

def lambda_handler(event, context):
    url = os.environ['SLACK_URL']
    slack_notify(url, str(event))
```

## Lambda with CloudWatch Logs
- 資料會被用zip壓縮，並以base64編碼，所以要先解碼
- 一開始拿到的資料應該會長：
```jsonld=
{
  "awslogs": {
    "data": "ewogICAgIm1lc3NhZ2VUeXBlIjogIkRBVEFfTUVTU0FHRSIsCiAgICAib3duZXIiOiAiMTIzNDU2Nzg5MDEyIiwKICAgICJsb2dHcm91cCI6I..."
  }
}
```
解碼後長：
```jsonld=
{
    "messageType": "DATA_MESSAGE",
    "owner": "123456789012",
    "logGroup": "/aws/lambda/echo-nodejs",
    "logStream": "2019/03/13/[$LATEST]94fa867e5374431291a7fc14e2f56ae7",
    "subscriptionFilters": [
        "LambdaStream_cloudwatchlogs-node"
    ],
    "logEvents": [
        {
            "id": "34622316099697884706540976068822859012661220141643892546",
            "timestamp": 1552518348220,
            "message": "REPORT RequestId: 6234bffe-149a-b642-81ff-2e8e376d8aff\tDuration: 46.84 ms\tBilled Duration: 100 ms \tMemory Size: 192 MB\tMax Memory Used: 72 MB\t\n"
        }
    ]
}
```
請[參考](https://docs.aws.amazon.com/zh_tw/lambda/latest/dg/services-cloudwatchlogs.html)
串接CloudWatch Logs的程式碼：(from 尚禮哥)
```python
import json
from botocore.vendored import requests # or import request
import os
import base64
import gzip

def slack_notify(url:str, content:str):
    dict_headers = {'Content-type': 'application/json'}
    dict_payload = {'text': content}
    json_payload = json.dumps(dict_payload)
    resp = requests.post(url, data=json_payload, headers=dict_headers)

def lambda_handler(event, context):
    url = os.environ['SLACK_URL']
    cloudwatch_event = event["awslogs"]["data"]
    decode_base64 = base64.b64decode(cloudwatch_event)
    decompress_data = gzip.decompress(decode_base64)
    log_data = json.loads(decompress_data)
    slack_notify(url, str(log_data))
```