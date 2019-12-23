# Line Chat OP
在 https://notify-bot.line.me/ 登錄之後，右上角個人頁面->發行存取權杖（開發人員用）->選擇聊天室->發行
就可以拿到token
這個token可以用在curl 例如
```
curl -X POST -i https://notify-api.line.me/api/notify -F "message=test 專案發佈成功" -H "Content-Type:multipart/form-data" -H "Authorization:Bearer $token"
```
這樣就可以收到line notify的通知。