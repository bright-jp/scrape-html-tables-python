# PythonでHTMLテーブルをスクレイピングする方法

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.jp/) 

このガイドでは、Beautiful Soup、pandas、Requests を使用して、PythonでHTMLテーブルをスクレイピングする方法を説明します。

- [前提条件](#prerequisites)
- [Webページ構造の理解](#understanding-the-web-page-structure)
- [HTTPリクエストを送信してWebページにアクセスする](#sending-an-http-request-to-access-the-web-page)
- [Beautiful Soupを使用してHTMLを解析する](#parsing-the-html-using-beautiful-soup)
- [データのクリーニングと構造化](#cleaning-and-structuring-the-data)
- [クリーニング済みデータをCSVにエクスポートする](#exporting-cleaned-data-to-csv)

## Prerequisites

Python 3.8 以降がインストールされていることを確認し、[virtual environment](https://docs.python.org/3/library/venv.html) を作成して、以下のPythonパッケージをインストールしてください。

- **[Requests](https://requests.readthedocs.io/en/latest/)**: HTTPリクエストを送信してWebサービスやAPIとやり取りし、データの取得や送信を可能にするライブラリです。
- **[Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)**: HTMLドキュメントを解析するためのツールで、構造化されたナビゲーション、検索、およびWebページからのデータ抽出が可能です。
- **[pandas](https://pandas.pydata.org/)**: スクレイピングしたデータの分析、クリーニング、整理のためのライブラリで、CSVやXLSXなどの形式へのエクスポートをサポートします。

```bash
pip install requests beautifulsoup4 pandas
```

## Understanding the Web Page Structure

このチュートリアルでは、2024年の最新の国別人口データが掲載されている [Worldometer website](https://www.worldometers.info/world-population/population-by-country/) からデータをスクレイピングします。

![HTML table on the web page](https://github.com/luminati-io/scrape-html-tables-python/blob/main/images/image-71-2048x1341.png)

HTMLテーブルを確認するには、テーブルを右クリックして **Inspect** を選択します。これによりDeveloper Toolsパネルが開き、該当するHTMLコードがハイライト表示されます。

![Inspect element with selected element highlighted](https://github.com/luminati-io/scrape-html-tables-python/blob/main/images/image-72-2048x1152.png)

テーブル構造は `<table>` タグ（ID `example2`）から始まり、`<th>` タグで定義されたヘッダーセルと、`<tr>` タグで定義された行で構成されています。各行の中では、`<td>` タグが個々のセルを作成し、データを保持します。

> **Note:**
>
> スクレイピングの前に、Webサイトのプライバシーポリシーおよび利用規約を確認し、データ利用と自動アクセスに関する制限をすべて遵守していることを確認してください。

## Sending an HTTP Request to Access the Web Page

HTTPリクエストを送信してWebページにアクセスするには、Pythonファイル（_eg_ `html_table_scraper.py`）を作成し、`requests`、`BeautifulSoup`、`pandas` パッケージをインポートします。

```python
# import packages
import requests
from bs4 import BeautifulSoup
import pandas as pd
```

次に、スクレイピング対象のWebページURLを定義し、`https://www.worldometers.info/world-population/population-by-country/` に対してGETリクエストを送信します。

```python
# Send a request to the website to get the page content
url = 'https://www.worldometers.info/world-population/population-by-country/'
```

Requests の `get()` メソッドを使用してリクエストを送信し、レスポンスが成功しているかを確認します。

```python
# Get the content of the URL
response = requests.get(url)
 
# Check the status of the response.
if response.status_code == 200:
    print("Request was successful!")
else:
    print(f"Error: {response.status_code} - {response.text}")
```

このコードは指定したURLにGETリクエストを送信し、その後レスポンスのステータスを確認します。`200` のレスポンスは、リクエストが成功したことを示します。

Pythonスクリプトを実行してください。

```bash
python html_table_scraper.py
```

出力は次のようになります。

```
Request was successful!
```

GETリクエストが成功したため、HTMLテーブルを含むWebページ全体のHTMLコンテンツが取得できました。

## Parsing the HTML Using Beautiful Soup

Beautiful Soupは、Webスクレイピングでよくある乱れたHTMLや壊れたHTMLを扱えるように作られています。これにより、以下が可能になります。
- HTMLを解析して人口データのテーブルを特定する
- テーブルヘッダーを抽出する
- 各テーブル行からデータを収集する

解析を開始するために、Beautiful Soupオブジェクトを作成します。

```python
# Parse the HTML content using BeautifulSoup
soup = BeautifulSoup(response.content, 'html.parser')
```

次に、`id` 属性が `"example2"` のHTML内のtable要素を特定します。このテーブルには2024年の国別人口が含まれています。

```python
# Find the table containing population data
table = soup.find('table', attrs={'id': 'example2'})
```

### Collecting Table Headers

テーブルには `<thead>` および `<th>` HTMLタグ内にヘッダーがあります。Beautiful Soupパッケージの `find()` メソッドを使用して `<thead>` タグ内のデータを抽出し、`find_all()` メソッドで全ヘッダーを収集します。

```python
# Collect the headers from the table
headers = []

# Locate the header row within the <thead> tag
header_row = table.find('thead').find_all('th')

for th in header_row:
    # Add header text to the headers list
    headers.append(th.text.strip())
```

このコードは `headers` という空のPythonリストを作成し、`<thead>` HTMLタグを特定して `<th>` HTMLタグ内の全ヘッダーを見つけ、収集した各ヘッダーを `headers` リストに追加します。

### Collecting Table Row Data

各行のデータを収集するために、スクレイピングしたデータを保存する空のPythonリスト `data` を作成します。

```python
# Initialize an empty list to store our data
data = []
```

次に、`find_all()` メソッドを使用してテーブル内の各行のデータを抽出し、Pythonリストに追加します。

```python
# Loop through each row in the table (skipping the header row)
for tr in table.find_all('tr')[1:]:
    # Create a list of the current row's data
    row = []
    
# Find all data cells in the current row
    for td in tr.find_all('td'):
        # Get the text content of the cell and remove extra spaces
        cell_data = td.text.strip()
        
        # Add the cleaned cell data to the row list
        row.append(cell_data)
        
    # After getting all cells for this row, add the row to our data list
    data.append(row)
    
# Convert the collected data into a pandas DataFrame for easier handling
df = pd.DataFrame(data, columns=headers)

# Print the DataFrame to see the number of rows and columns
print(df.shape)
```

このコードは `table` 内で見つかったすべての `<tr>` HTMLタグを、2行目（ヘッダー行をスキップ）から順に反復します。各行（`<tr>`）ごとに、その行のセルのデータを格納する空のリスト `row` を作成します。行の内部では、`find_all()` メソッドを使用してすべての `<td>` HTMLタグ（行内の個々のデータセル）を見つけます。

各 `<td>` HTMLタグについて、`.text` 属性でテキスト内容を抽出し、`.strip()` メソッドで前後の余分な空白を取り除きます。クリーニングされたセルデータは `row` リストに追加されます。現在の行の全セルを処理したら、行全体が `data` リストに追加されます。最後に、収集したデータを `headers` リストで定義された列名を持つpandas DataFrameに変換し、データの形状を表示します。

完全なPythonスクリプトは次のようになります。

```python
# Import packages
import requests
from bs4 import BeautifulSoup
import pandas as pd

# Send a request to the website to get the page content
url = 'https://www.worldometers.info/world-population/population-by-country/'

# Get the content of the URL
response = requests.get(url)

# Check if the request was successful
if response.status_code == 200:
    # Parse the HTML content using Beautiful Soup
    soup = BeautifulSoup(response.content, 'html.parser')

    # Find the table containing population data by its ID
    table = soup.find('table', attrs={'id': 'example2'}) 

    # Collect the headers from the table
    headers = []

    # Locate the header row within the <thead> HTML tag
    header_row = table.find('thead').find_all('th')

    for th in header_row:
        # Add header text to the headers list
        headers.append(th.text.strip())

    # Initialize an empty list to store our data
    data = []

    # Loop through each row in the table (skipping the header row)
    for tr in table.find_all('tr')[1:]:

        # Create a list of the current row's data
        row = []

        # Find all data cells in the current row
        for td in tr.find_all('td'):
            # Get the text content of the cell and remove extra spaces
            cell_data = td.text.strip()

            # Add the cleaned cell data to the row list
            row.append(cell_data)

        # After getting all cells for this row, add the row to our data list
        data.append(row)

    # Convert the collected data into a pandas DataFrame for easier handling
    df = pd.DataFrame(data, columns=headers)

    # Print the DataFrame to see the collected data
    print(df.shape)
else:
    print(f"Error: {response.status_code} - {response.text}")
```

ターミナルで次のコマンドを使用してPythonスクリプトを実行します。

```bash
python html_table_scraper.py
```

出力は次のようになります。

```
(234,12)
```

次に、pandas の `head()` メソッドと `print()` を使用して、抽出したデータの先頭10行を表示します。

```python
print(df.head(10))
```

![Top ten rows from the scraped table](https://github.com/luminati-io/scrape-html-tables-python/blob/main/images/image-73-2048x665.png)

## Cleaning and Structuring the Data

HTMLテーブルからスクレイピングしたデータのクリーニングは、分析における一貫性、正確性、そして利用しやすさのために重要です。生データには欠損値、フォーマットエラー、不要な文字、または誤ったデータ型が含まれる場合があり、これらは信頼できない結果につながる可能性があります。適切にクリーニングすることでデータセットが標準化され、分析に必要な意図した構造に整合します。

このセクションでは、次のデータクリーニング作業を行います。

- 列名の変更
- 行データに存在する欠損値の置換
- カンマを削除し、データ型を正しい形式に変換
- パーセント記号（%）を削除し、データ型を正しい形式に変換
- 数値列のデータ型を変更

### Renaming Column Names

pandas には、列名を更新してより説明的にしたり、扱いやすくしたりするための `rename()` メソッドがあります。`columns` パラメータに辞書を渡すだけで、キーが現在の名前、値が新しい名前を表します。このメソッドを使用して、次の列名を更新します。

- `#` を `Rank` に
- `Yearly change` を `Yearly change %` に
- `World Share` を `World Share %` に

```python
# Rename columns
df.rename(columns={'#': 'Rank'}, inplace=True)
df.rename(columns={'Yearly Change': 'Yearly Change %'}, inplace=True)
df.rename(columns={'World Share': 'World Share %'}, inplace=True)

# Show the first 5 rows
print(df.head())
```

列は次のようになります。

![Column names after renaming](https://github.com/luminati-io/scrape-html-tables-python/blob/main/images/image-74-2048x371.png)

### Replacing Missing Values

欠損値は平均や合計などの計算を歪め、不正確な洞察につながる可能性があります。分析を行う前に、これらの欠損を削除、置換、または適切な値で補完してください。

`Urban Pop %` 列には現在、`N.A.` とラベル付けされた欠損値が含まれています。pandas の `replace()` メソッドを使用して `N.A.` を `0%` に置換します。

```python
# Replace 'N.A.' with '0%' in the 'Urban Pop %' column
df['Urban Pop %'] = df['Urban Pop %'].replace('N.A.', '0%')
```

### Removing Percentage Signs and Convert Data Types

`Yearly Change %`、`Urban Pop %`、`World Share %` の各列には `%` 記号付きの数値（例: `37.0%`）が含まれており、平均や標準偏差の計算などの直接的な数値演算ができません。これを修正するには、`replace()` メソッドで `%` を削除し、`astype()` を使用して値を `float` に変換します。

```python
# Remove the '%' sign and convert to float
df['Yearly Change %'] = df['Yearly Change %'].replace('%', '', regex=True).astype(float)
df['Urban Pop %'] = df['Urban Pop %'].replace('%', '', regex=True).astype(float)
df['World Share %'] = df['World Share %'].replace('%', '', regex=True).astype(float)

# Show the first 5 rows
df.head()
```

このコードは正規表現を用いた `replace()` メソッドで `Yearly Change %`、`Urban Pop %`、`World Share %` 列の値から `%` 記号を削除します。その後、`astype(float)` を使用してクリーニング後の値を `float` データ型に変換します。最後に、`df.head()` でDataFrameの先頭5行を表示します。

出力は次のようになります。

![Top five rows](https://github.com/luminati-io/scrape-html-tables-python/blob/main/images/image-75-2048x371.png)

### Removing Commas and Convert Data Types

現在、`Population (2024)`、`Net Change`、`Density (P/Km²)`、`Land Area (Km²)`、`Migrants (net)` の各列には、カンマを含む数値（_eg_ 1,949,236）が入っています。これにより、分析のための数値演算ができません。

これを修正するには、`replace()` と `astype()` を適用してカンマを削除し、数値を整数データ型に変換します。

```python
# Remove commas and convert to integers
columns_to_convert = [
    'Population (2024)', 'Net Change', 'Density (P/Km²)', 'Land Area (Km²)',
    'Migrants (net)'
]

for column in columns_to_convert:
    # Ensure the column is treated as a string first
    df[column] = df[column].astype(str)

    # Remove commas
    df[column] = df[column].str.replace(',', '')

    # Convert to integers
    df[column] = df[column].astype(int)
```

このコードは、処理対象の列のために `columns_to_convert` というリストを作成します。各列について、`astype(str)` で値を文字列に変換し、`str.replace(',', '')` でカンマを削除してから、`astype(int)` で整数に変換し、数値演算に適した形に整えます。

### Changing Data Types for Numerical Columns

`Rank`、`Med. Age`、`Fert. Rate` の各列は数値を含んでいるにもかかわらずobjectとして保存されています。数値演算ができるように、これらの列を整数または浮動小数点に変換します。

```python
# Convert to integer or float data types and integers

df['Rank'] = df['Rank'].astype(int)
df['Med. Age'] = df['Med. Age'].astype(int)
df['Fert. Rate'] = df['Fert. Rate'].astype(float)
```

このコードは、`Rank` と `Med. Age` 列の値を整数データ型に、`Fert. Rate` の値を浮動小数点（float）データ型に変換します。

最後に、`head()` メソッドを使用して、クリーニング済みデータが正しいデータ型になっていることを確認します。

```python
print(df.head(10))
```

出力は次のようになります。

![Top ten rows of the cleaned data](https://github.com/luminati-io/scrape-html-tables-python/blob/main/images/image-76-2048x665.png)

## Exporting Cleaned Data to CSV

データをクリーニングしたら、今後の分析や共有のためにCSVファイルとして保存します。pandas の `to_csv()` メソッドを使用して、DataFrameを `world_population_by_country.csv` というファイル名でエクスポートします。

```python
# Save the data to a file
filename = 'world_population_by_country.csv'
df.to_csv(filename, index=False)
```

## Conclusion

複雑なWebサイトからのデータ抽出は、難しく時間がかかる場合があります。時間を節約し、作業を簡単にするために、[Bright Data Web Scraper API](https://brightdata.jp/products/web-scraper) の利用をご検討ください。この強力なツールは、事前構築されたスクレイピングソリューションを提供し、最小限の技術知識で複雑なWebサイトからデータを抽出できます。

サインアップして、Web Scraper API の無料トライアルを開始してください。