---
title:  [GitHub Actions + Google Spread Sheet] 깃헙의 CSV 파일이 변경되었을 때, 구글 스프레드 시트로 싱크 자동화
excerpt: "GitHub Action과 Google Sheet를 이용해서 CSV 파일 동기화 자동화"

categories:
  - 개발
tags:
  - [GitHub, GitHub Actions, Google Sheet, Apps Scriot, Python]

toc: true
toc_sticky: true
 
date: 2024-05-06
last_modified_at: 2024-05-06
---



# [GitHub Actions + Google Sheet] 깃헙의 CSV 파일이 변경되었을 때, 구글 스프레드 시트로 싱크 자동화

CSV는 개발자가 읽고 쓰기는 편하지만, 기획자가 함수나 Visualization을 하기 어렵습니다.  



하지만 GitHub으로 CSV 원본 파일을 저장하되, 구글 스프레드 시트로 자동으로 동기화하고 하고 ImportRange 함수로 가공해서 쓰면 두 이점을 모두 챙길 수 있습니다.



그래서 GitHub Action으로 GitHub에 있는 CSV를 Google Sheet로 동기화하는 방법을 알려드립니다.



## 개발 도구

- GitHub Actions
- Google Sheet
- AppsScript
- Python3



## 결과물 예시

최종 결과물은 이렇게 동작합니다.

#### 1. CSV 파일을 추가해서 새로운 커밋을 올린다.
![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/b1d9cad9-0e4c-48a7-af98-d0c19e76ddc4)

#### 2. 그러면 구글 시트에 방금 올린 CSV 파일이 추가된다.
![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/6346d18b-a788-4010-900c-c013122ef04d)

#### 3. 값을 변경한다.
![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/a285d146-3e42-49d7-bfd3-99b28ff7d746)

#### 4. 구글 시트의 값도 변경된다.
![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/e8d01c95-1916-4868-9137-ffd7727da077)

## 구조
![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/c5e7bd11-7386-48f0-bdb9-406299ddc493)



#### 1. GitHub (push)

#### 2. GitHub Actions (workflow)

#### 3. Python

#### 4. Google Sheet (Apps Script)



전체적인 동작 방식은 이렇습니다.

1. GitHub에 CSV 파일을 변경/추가하고 push한다.

2. GitHub Actions가 push를 감지해서 Python 코드를 호출한다.

3. Python 코드에서는 변경된 CSV 파일을 찾아서, Apps Script를 통해 Google Sheet로 전송한다.

   

## 제작
#### 1. Apps Script 코드를 만듭니다.
<img src="https://github.com/choeehb/choeehb.github.io/assets/17942921/8b7bef88-1a22-4ec0-a1db-f0cd1f7ed66f" alt="image" />



Apps Script는 웹앱처럼 사용할 수 있습니다.
{csv 파일명: csv 파일 텍스트} 형태의 데이터를 페이로드로 넘겨받습니다.



아래와 같이 스크립트를 입력하고 저장합니다.

``` javascript
function doPost(e) {
  // e.parameter에서 JSON 문자열을 받아옵니다.
  var payload = JSON.parse(e.postData.contents);
  
  // 스프레드시트를 불러옵니다.
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var updatedSheets = []; // 업데이트된 시트 이름을 저장할 배열

  // payload에서 각 파일명과 내용을 가져와 해당 시트만 제거하고 다시 만듭니다.
  Object.keys(payload).forEach(function(filename) {
    var content = payload[filename];
    
    // 파일명에 해당하는 시트를 찾습니다.
    var sheet = ss.getSheetByName(filename);
    if (sheet !== null) {
      // 시트가 존재하면 제거
      ss.deleteSheet(sheet);
    }
    // 새로운 시트를 만들고 이름을 설정
    sheet = ss.insertSheet(filename);
    updatedSheets.push(filename); // 업데이트된 시트 이름을 배열에 추가
    
    // CSV 내용을 파싱하여 각 행을 추가
    var data = Utilities.parseCsv(content);
    sheet.getRange(1, 1, data.length, data[0].length).setValues(data);
  });
  
  // 처리가 성공했다는 응답을 반환하면서 업데이트된 시트의 이름을 포함
  var responseMessage = "Sheets updated successfully: " + updatedSheets.join(", ");
  return ContentService.createTextOutput(responseMessage);
}
```

권한을 추가하기 위해서 편집기에 'appsscript.json' 매니페스트 파일을 띄우고, outhScopes를 추가합니다.

![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/8534afab-0959-4231-b5a9-a51f126ddf04)

```json
  "oauthScopes": [
    "https://www.googleapis.com/auth/script.external_request",
    "https://www.googleapis.com/auth/drive",
    "https://www.googleapis.com/auth/spreadsheets.currentonly",
    "https://www.googleapis.com/auth/spreadsheets"
  ]
```

배포합니다.
![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/afe2af70-42f9-4cb1-9048-b3951de3d5ee)

배포 이후에 나온 URL을 저장해두세요.

이 URL로 csv파일을 담아서 http post 요청을 보내면, 구글 시트가 갱신됩니다.
![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/d2cca045-d642-4da3-af04-34b9b979cdaf)

#### 2. Python 코드를 작성합니다.

파이썬이 할 일
- 현재 브랜치에서 변경된 csv 파일을 찾는다.
- Apps Script로 post 메세지를 보낸다.

코드 이름은 update_sheets.py로 지으세요.

바꿔도 상관은 없는데, 아래 워크플로에서 'update_sheets.py'로 호출합니다

``` python3
import subprocess
import os
import requests

def get_changed_files():
    # 'git diff' 명령을 실행하여 마지막 커밋 이후 변경된 파일들을 가져옵니다.
    changed_files = subprocess.check_output(['git', 'diff', '--name-only', 'HEAD^', 'HEAD']).decode().split()
    # 'sheets' 폴더 내의 .csv 파일만 필터링합니다.
    return [f for f in changed_files if f.startswith('sheets/') and f.endswith('.csv')]

def main():
    changed_csv_files = get_changed_files()
    payload = {}

    for csv_file in changed_csv_files:
        with open(csv_file, 'r', encoding='utf-8') as file:
            content = file.read()
            filename_without_extension = os.path.splitext(os.path.basename(csv_file))[0]
            payload[filename_without_extension] = content

    # Apps Script 웹앱 URL
    url = '아까전에 얻은 URL'
    print('----payload----')
    print(payload)
    # POST 요청으로 데이터 전송
    response = requests.post(url, json=payload)
    print('----response----')
    print(response.text)

if __name__ == '__main__':
    main()
```



#### 3. GitHub Actions Workflow 만들기

이제 GitHub Action으로 push가 발생할 때마다 위에서 만든 파이썬 코드를 호출합니다.
![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/849b65e1-b688-4d33-b5ce-b47a8b13c669)
![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/9bd313ce-4bb0-491f-8843-b6ba0b0057f2)

Actions에 들어가서 새로운 워크플로를 추가합니다.

```yaml
name: Update Google Sheets

on:
  push:
    branches:
      - nightly-main
      - feature/workflow-sheet
    paths:
      - 'sheets/*.csv'  # 'sheets' 폴더 내 CSV 파일에 대한 변경 사항만 감지

jobs:
  update-sheets:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0  # 모든 커밋 히스토리를 가져옴

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install requests

      - name: Identify changed CSV files and update Google Sheets
        run: python update_sheets.py
```



끝
이제 커밋을 하면 저렇게 초록색 체크마크가 뜨면 성공한겁니다.
![image](https://github.com/choeehb/choeehb.github.io/assets/17942921/67d22f32-7c39-449c-ab13-057de14a2cbe)

사소한 문제
- CSV 파일을 지웠을 때, 구글 시트의 시트를 지우지 않는다.
- Apps Script에서 모든 시트를 지우면 문제가 생긴다. 그래서 임시 시트 1개를 미리 만들어 둬야한다.
