# 話題の ChatGPT + LangChain で、SQL データベースに自然言語でクエリしてみる

## はじめに

本記事では、OpenAI の ChatGPT と LangChain の API を使用し、自然言語で SQL データベースに問い合わせを行う方法を紹介します。

具体的には、SQL データベースに対して自然言語で問い合わせをすると、自然言語で結果が返ってくる、というものです。

ChatGPT と LangChain を使用することで、下記のような複数ステップの仕事を非常に簡単に実行させることができます。

1. 自然言語による問い合わせ文に対応する SQL クエリを生成させる
1. 生成された SQL 文を SQL データベースに対して実行する
1. SQL クエリの実行結果を自然言語で説明する回答を生成させる

この方法を使用することで、SQL の知識がない人でも簡単にデータベースにアクセスできます。

## 必要なもの

本ブログでは実行環境として Google Colab 上で Python コードを書き、ChatGPT と LangChain の Python API を呼び出すこととします。

- Google Colab アカウント
- ChatGPT API キー
- SQL データセット

### ChatGPT API キーの取得方法

ChatGPT API キーを取得するには、OpenAI の Web サイトに登録する必要があります。登録後、API キーを取得してください。API キーは、後で使用するので、大切に保管してください。

### LangChain を使うには？

LangChain はライブラリですので、使用するのに特に API キーのようなものは必要ありません。詳しくは LangChain の GitHub リポジトリを参照してください。

https://github.com/hwchase17/langchain

### 使用するデータセット

本ブログでは、データセットとしてアメリカの赤ちゃんにつける名前の SQL データセットを使用することとします。

アメリカの赤ちゃんにつける名前のデータセットは、Kaggle から入手することができます。このデータセットには、アメリカで生まれた赤ちゃんの名前、生年月日、性別、出生地の情報が含まれており、年次データが収集されています。データセットは CSV ファイル形式で提供され、無料でダウンロードできます。

https://www.kaggle.com/datasets/kaggle/us-baby-names

また、上記のデータセットには SQLite 形式の DB ファイルが提供されているため、後述する LangChain の API にそのまま読み込ませることができます。

この SQL データセットには、下記のような SQL テーブルがあります。

```
CREATE TABLE StateNames (
    Id INTEGER PRIMARY KEY,
    Name TEXT,
    Year INTEGER,
    Gender TEXT,
    State TEXT,
    Count INTEGER);
CREATE INDEX statenames_name_idx  ON StateNames (Name);
CREATE INDEX statenames_year_idx  ON StateNames (Year);
CREATE INDEX statenames_state_idx ON StateNames (State);
...
```

## Google Colab での Python コードの実行

では、Google Colab を開き、Python コードを書くためのノートブックを作成します。はじめに、以下のように必要なライブラリをインポートします。

```
%%writefile requirements.txt
openai
langchain
```

```
%pip install -r requirements.txt
```

```
import platform
import os
import openai
from langchain import OpenAI, SQLDatabase, SQLDatabaseChain
```

### SQL データセットのマウント

SQL データセットは、Google ドライブに格納して Google Colab から Python コードでアクセスすることができます。この場合、Google Colab の環境を使って、Python のライブラリをインストールし、Google ドライブとの接続を設定する必要があります。

```
from google.colab import drive
drive.mount('/content/drive')
```

その後、LangChain の API で、Google ドライブ内のデータセットを読み込んでおきます。これで、SQL 文を生成するためのコードを記述することができます。

```
db = SQLDatabase.from_uri("sqlite:////content/drive/My Drive/Colab Notebooks/SQLNotebooks/data/database.sqlite")
```

### LangChain における LLM のセットアップ

次に、ChatGPT による自然言語でのクエリ処理を実行するためのコードを書きます。

まず、OpenAI の API Key をセットして、LangChain で使用する LLM を準備します。

```
os.environ["OPENAI_API_KEY"] = '(Your API Key is here)'
openai.api_key = os.getenv("OPENAI_API_KEY")
llm = OpenAI(temperature=0)
```

上記のコードでは、ChatGPT の `text-davinci-003` エンジンを使用して、クエリを生成します。

その他のモデル名を指定したり、最大トークン数などのパラメータも指定することもでき、生成されるテキストの質を制御するために使用されます。

上記ではサンプリング温度以外、デフォルトとしています。サンプリング温度は、生成されるテキストにおける単語の選択に関係していますが、0 という、一番低いサンプリング温度を指定しているため、最も出現頻度が高い単語が選択されるようになります。

### SQLDatabaseChain のセットアップ

次に、LLM に与えるプロンプトのテンプレートを準備します。

```
_DEFAULT_TEMPLATE = """Given an input question, first create a syntactically correct {dialect} query to run, then look at the results of the query and return the answer.
Use the following format:

Question: "Question here"
SQLQuery: "SQL Query to run"
SQLResult: "Result of the SQLQuery"
Answer: "Final answer here"

Only use the following tables:

{table_info}

Question: {input}"""
```

```
from langchain import PromptTemplate
PROMPT = PromptTemplate(
    input_variables=["input", "table_info", "dialect"], template=_DEFAULT_TEMPLATE
)
```

上記は SQLDatabaseChain のドキュメントにあるプロンプト例をベースにしています。詳しくは下記をご参照ください。

https://python.langchain.com/en/latest/modules/chains/examples/sqlite.html

前述の LLM とプロンプトテンプレート、SQL データセットを与えて SQLDatabaseChain をセットアップします。

```
db_chain = SQLDatabaseChain(llm=llm, database=db, prompt=PROMPT, verbose=True, return_intermediate_steps=True)
```

これで SQL データベースへ自然言語で問い合わせる準備ができました。

### SQL データベースへ自然言語で問い合わせる

具体的な問い合わせの例として、カリフォルニア州で女の子の赤ちゃんに対する最も人気のある名前について調べてみたいと思います。そこで、以下のような質問を SQL データベースに対して問い合わせることとします。

> **_What are the most popular names for female babies in California? Provide top 10 names by total babies in descending order._**

具体的な問い合わせ方法は、上記の問い合わせ文を SQLDatabaseChain の API に対してそのまま与えるだけです。

```
result = db_chain("What are the most popular names for female babies in California? Provide top 10 names by total babies in descending order")
```

上記に対する出力結果はちょっと読みづらいので、後ほど紹介することとし、先に抜粋したものを紹介します。

まず、ChatGPT は、次のような SQL クエリを生成しました。

> SELECT name, total_babies FROM gendered_names WHERE state = 'CA' AND primary_sex = 'F' ORDER BY total_babies DESC LIMIT 10;

LangChain がこの SQL クエリを実行し、データベースからの結果を ChatGPT に与えます。

下記が生成された SQL クエリと実行結果になります。

```
import pprint
pp = pprint.PrettyPrinter(indent=4)

pp.pprint(result["intermediate_steps"])
```

```
[   " SELECT Name, SUM(Count) AS TotalCount FROM StateNames WHERE Gender = 'F' "
    "AND State = 'CA' GROUP BY Name ORDER BY TotalCount DESC LIMIT 10;",
    "[('Jennifer', 174053), ('Mary', 151993), ('Jessica', 134163), "
    "('Elizabeth', 127636), ('Patricia', 116572), ('Linda', 114321), "
    "('Michelle', 112123), ('Maria', 101439), ('Stephanie', 99685), ('Lisa', "
    '94281)]']
```

そして、下記が ChatGPT による自然言語での回答結果です。

```
pp.pprint(result['result'])
```

```
(' The top 10 most popular names for female babies in California are Jennifer, '
 'Mary, Jessica, Elizabeth, Patricia, Linda, Michelle, Maria, Stephanie, and '
 'Lisa.')
```

上記から、カリフォルニアで最も人気な女の子の赤ちゃんにつける名前は、ジェニファーちゃんであることが判明しました。

| 順位 | 名前         |
| ---- | ------------ |
| 1 位 | ジェニファー |
| 2 位 | メリー       |
| 3 位 | ジェシカ     |
| ...  | ...          |

LangChain API の実際の出力結果が下記になります。きちんと、前述のプロンプトテンプレートに沿った回答になっていることが分かります。

```
> Entering new SQLDatabaseChain chain...
What are the most popular names for female babies in California? Provide top 10 names by total babies in descending order
SQLQuery: SELECT Name, SUM(Count) AS TotalCount FROM StateNames WHERE Gender = 'F' AND State = 'CA' GROUP BY Name ORDER BY TotalCount DESC LIMIT 10;
SQLResult: [('Jennifer', 174053), ('Mary', 151993), ('Jessica', 134163), ('Elizabeth', 127636), ('Patricia', 116572), ('Linda', 114321), ('Michelle', 112123), ('Maria', 101439), ('Stephanie', 99685), ('Lisa', 94281)]
Answer: The top 10 most popular names for female babies in California are Jennifer, Mary, Jessica, Elizabeth, Patricia, Linda, Michelle, Maria, Stephanie, and Lisa.
> Finished chain.
```

もちろん下記のように、同じ質問を日本語で問い合わせても同様の結果が得られます。

```
resultJP = db_chain("カリフォルニアで女の子の赤ちゃんに対する最も人気のある名前はなんですか？赤ちゃんの合計数で上位１０個の名前を降順に表示してください。")
```

```
> Entering new SQLDatabaseChain chain...
カリフォルニアで女の子の赤ちゃんに対する最も人気のある名前はなんですか？赤ちゃんの合計数で上位１０個の名前を降順に表示してください。
SQLQuery: SELECT Name, SUM(Count) AS TotalCount FROM StateNames WHERE Gender = 'F' AND State = 'CA' GROUP BY Name ORDER BY TotalCount DESC LIMIT 10;
SQLResult: [('Jennifer', 174053), ('Mary', 151993), ('Jessica', 134163), ('Elizabeth', 127636), ('Patricia', 116572), ('Linda', 114321), ('Michelle', 112123), ('Maria', 101439), ('Stephanie', 99685), ('Lisa', 94281)]
Answer: カリフォルニアで女の子の赤ちゃんに対する最も人気のある名前はJennifer、Mary、Jessica、Elizabeth、Patricia、Linda、Michelle、Maria、Stephanie、そしてLisaです。
> Finished chain.
```

以上が、ChatGPT と LangChain を使用して自然言語で SQL データベースに問い合わせる方法の具体的な例となります。

## まとめ

今回のブログでは、ChatGPT と LangChain を使用して、SQL データセットに自然言語で問い合わせる方法を紹介しました。

この方法は、SQL データベースに直接アクセスする必要がある場合や、データの加工や集計が必要な場合に有用です。また、ChatGPT と LangChain を使うことで、ChatGPT が SQL データベースのスキーマの情報からカラムの意味を推測して適切な SQL 文を生成してくれるため、SQL の知識がなくてもデータベースを扱うことができます。

実は、前述のデータセットは非常に有名なものであるため、わざわざ LangChain を使用せずとも ChatGPT はカリフォルニアで女の子の赤ちゃんに対する最も人気のある名前について答えることができてしまうものです。

そこで、ぜひ読者の方には ChatGPT がまだ直接答えることができないであろう SQL データセットに対して ChatGPT と LangChain を用いて自然言語で問い合わせてみて頂きたいと思います。本ブログが参考になれば幸いです。
