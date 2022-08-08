```python
# 
# Copyright (C) 2019-2021 vdaas.org vald team <vald@vdaas.org>
# 
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
# 
#     https://www.apache.org/licenses/LICENSE-2.0
# 
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# 
```

# Vald Similarity Search using chiVe Dataset

---
※***このnotebookは, 既に[Get Started](https://vald.vdaas.org/docs/tutorial/get-started/)を完了し, Valdの環境構築が完了した方を対象としています.  
まだValdの環境構築がお済みでない方は, 先に[Get Started](https://vald.vdaas.org/docs/tutorial/get-started/)を行うことを推奨します.  
また, データセットとしてchiVeを利用する場合, Vald Agentのdimensionを300に, distance_typeをcosineにすることを推奨するため, [sample-values.yaml](https://github.com/vdaas/vald-demo/blob/main/chive/sample-values.yaml)を用いる or 値を修正したvalues.yamlを用いてValdの構築を行ってください.***  
- *dimension: 300, distance_type: consieの修正例 ([path/to/helm/values.yaml](https://github.com/vdaas/vald/blob/main/example/helm/values.yaml#L45-L49))*:
```yaml
agent:
  ngt:
    dimension: 300
    distance_type: cos
```
---

このnotebookの目的は, vald-python-clientを通じて, Valdの基礎的な動作であるInsert/Search/Update/Removeを体験し, 近似近傍探索を用いた検索の一例を体験することです.  
今回, 検索を行うためのデータセットとして, 日本語単語ベクトルのデータセットである[chiVe](https://github.com/WorksApplications/chiVe)を利用しています.

notebookの概要は以下の通りです:
- Preprocess
  - Install packages
  - Import dependencies
  - Prepare the vector data with chiVe
- Similarity Search with vald-client-python
  - Create gRPC channel
  - Insert/Search/Update/Remove
- Advanced
  - Word Analogies
  
それでは, Valdを利用した近似近傍探索による検索を体験してみましょう!!

---

## Preprocess

Valdを利用するにあたって必要なパッケージやベクトルデータを準備します.

### Install packages

※*動作環境に応じてパッケージのインストールを行ってください.*


```python
!pip install grpcio pymagnitude vald-client-python
```

### Import dependencies

notebookを実行するに当たり, 必要なパッケージをインポートします.


```python
import grpc
import io
import os
import pandas as pd

from pymagnitude import Magnitude
from tqdm.notebook import tqdm
from vald.v1.payload import payload_pb2
from vald.v1.vald import (insert_pb2_grpc,
                          object_pb2_grpc,
                          remove_pb2_grpc,
                          search_pb2_grpc,
                          update_pb2_grpc)
```

### Prepare the vector data with [chiVe](https://github.com/WorksApplications/chiVe)

このnotebookでは, 日本語単語ベクトルとして[chiVe](https://github.com/WorksApplications/chiVe)を用いるため, 予め必要なデータをダウンロードしておくことを推奨します.
```
curl "https://sudachi.s3-ap-northeast-1.amazonaws.com/chive/chive-1.2-mc90.magnitude" -o "chive-1.2-mc90.magnitude"
```

データを読み込み, サンプルとなるクエリを用いてベクトルを表示します.


```python
# NOTE: "___" -> "/path/to/chive-1.2-mc90.magnitude"
vectors = Magnitude("___")
"テスト" in vectors, vectors.query("テスト")
```




    (True,
     array([-0.0114243,  0.0477236,  0.0876566,  0.0197937, -0.1209878,
            -0.0816604,  0.0081722,  0.0903953,  0.0898218,  0.0035701,
            -0.0470737, -0.0887895, -0.0832327, -0.0100294,  0.0481049,
            -0.0430062, -0.0312618, -0.0597037,  0.0346277, -0.0025902,
             0.0157811, -0.0211378, -0.0122685, -0.1061476,  0.0584456,
             0.0059301, -0.0449118, -0.0165369, -0.0534942,  0.0013314,
            -0.0248894,  0.0268765,  0.0280359,  0.0893275,  0.1173892,
             0.0076032, -0.0571333,  0.0437547, -0.0401019, -0.0441953,
            -0.0244796, -0.0321471,  0.0333059, -0.0002985, -0.0581208,
            -0.1002922,  0.0905983,  0.0360072, -0.0278267,  0.1554238,
             0.0393037,  0.0486849, -0.0367415, -0.0060189, -0.0343168,
             0.0046014,  0.0766613, -0.0512379, -0.0722953, -0.0227866,
             0.0112333, -0.0887581, -0.0993438, -0.0066624, -0.0434988,
            -0.0116181,  0.0568109,  0.1101439, -0.025023 ,  0.0429484,
            -0.0504101, -0.04968  ,  0.048752 ,  0.0345578,  0.0172149,
            -0.0531341, -0.0453309, -0.0085477,  0.046678 , -0.0032684,
             0.0774227,  0.0717758,  0.0645902,  0.0695856, -0.0085593,
            -0.0115149,  0.0737886,  0.0353913, -0.0283075,  0.0269228,
             0.0338442, -0.0400569, -0.0817175,  0.0407814, -0.0261412,
            -0.0587236, -0.0763435,  0.0676031, -0.0619034, -0.0093709,
            -0.0685307, -0.0456472, -0.0132658, -0.0284764,  0.0707083,
             0.0024424,  0.1520566,  0.0215057,  0.0699215,  0.0655076,
             0.0439527,  0.0971613, -0.0191589, -0.1494674, -0.0660212,
             0.027601 ,  0.1606499,  0.0686498,  0.0059506,  0.0384091,
            -0.0498702,  0.0194641,  0.02156  , -0.1387536,  0.0205984,
             0.0510442,  0.0668833,  0.1212867, -0.0908945,  0.001041 ,
            -0.1035047, -0.0172008,  0.0696314,  0.0385644, -0.0154698,
             0.0073492, -0.0637545,  0.0951317,  0.0131859, -0.0714227,
             0.0443708,  0.0792128,  0.0221658, -0.0105484, -0.0550355,
            -0.0181468,  0.0018366, -0.0651676,  0.0645388, -0.0827222,
            -0.0477683, -0.07355  , -0.0239801, -0.0879891, -0.064406 ,
             0.0204663, -0.09808  ,  0.0219029,  0.0271339, -0.049638 ,
             0.0703503,  0.0134905, -0.0956024, -0.0290809,  0.0006495,
             0.0192448,  0.049475 , -0.0961541, -0.0697048, -0.0317764,
             0.0715788,  0.0084307,  0.0503979, -0.0235382,  0.0089828,
             0.0273437, -0.0581455,  0.042981 , -0.0625502, -0.1485213,
            -0.0024187,  0.0618574, -0.1114425, -0.0115271,  0.0863034,
            -0.0255691, -0.0205305, -0.0711611, -0.0034462, -0.1044056,
            -0.0046641,  0.0503957, -0.0170538, -0.0472625,  0.0058688,
             0.0533875, -0.0090653,  0.0335468,  0.0435079, -0.0339937,
            -0.0191978,  0.0568108,  0.0291197, -0.0011628, -0.0509131,
             0.0601245,  0.0308244, -0.0710804,  0.0731381, -0.0506741,
            -0.013013 ,  0.1390505,  0.1189048, -0.0409868, -0.0288562,
             0.0789599,  0.0786378,  0.0096255, -0.0614009,  0.0308439,
            -0.0511024,  0.0803731,  0.0007882, -0.0586906,  0.00491  ,
            -0.0336768,  0.0344689, -0.0445655, -0.070559 , -0.0293072,
             0.0213883, -0.0265467, -0.0665999,  0.0555801,  0.039686 ,
             0.0279025,  0.0356918,  0.0059812,  0.002016 ,  0.0360139,
             0.024166 ,  0.0852378,  0.002272 , -0.0208231,  0.0042477,
             0.0035503,  0.0609905, -0.08944  ,  0.0117659, -0.0452567,
             0.0131531, -0.0413104, -0.0168282, -0.0160598, -0.0542108,
             0.0638464, -0.0311752, -0.0886891,  0.0481477, -0.0117365,
             0.0187849, -0.0123002,  0.0321253,  0.082018 , -0.0026071,
             0.0222936,  0.020383 ,  0.0108969, -0.0485949, -0.1412405,
            -0.0365578,  0.0233411, -0.0123404,  0.0427132, -0.0758276,
            -0.0625174, -0.0502814, -0.0076557,  0.0214561, -0.0020813,
            -0.0534879,  0.0448377,  0.022926 , -0.0299312, -0.0878529,
            -0.0654284, -0.1041372,  0.0450328,  0.0583699,  0.1214726,
             0.0485365,  0.0136953, -0.0191399, -0.1057521, -0.0917401,
             0.0249748, -0.0379459,  0.0465482, -0.0345087,  0.1086441],
           dtype=float32))



---

## Similarity Search with vald-client-python

Valdの基礎的な動作であるInsert/Search/Update/Removeを実行すると共に, 前項で準備したベクトルデータを用いて近似近傍探索による検索を行います.

### Create gRPC channel

gRPCによる通信を行うため, Valdが動作している各環境に応じて, 必要なエンドポイントを記述し, channelを作成します.


```python
# NOTE: "___" -> "{host}:{port}"
channel = grpc.insecure_channel("___")
```

### Insert

初めに, Valdにデータを入れるため, Insertを行います.  
Insertを行うため, 先程作成したchannelを用いてInsert用のstubを作成します.


```python
# create stub
istub = insert_pb2_grpc.InsertStub(channel)
```

次に, Insert命令を用いてValdにデータ(id="test")を1件Insertし, 正常に動作が完了するか確認します.


```python
ivec = payload_pb2.Object.Vector(id="test", vector=vectors.query("テスト"))
icfg = payload_pb2.Insert.Config(skip_strict_exist_check=True)
ireq = payload_pb2.Insert.Request(vector=ivec, config=icfg)

istub.Insert(ireq)
```




    name: "vald-agent-ngt-0"
    uuid: "test"
    ips: "127.0.0.1"
    ips: "127.0.0.1"
    ips: "127.0.0.1"
    ips: "127.0.0.1"
    ips: "127.0.0.1"



1件のデータ(id="test")のInsertの動作確認が完了次第, 100,000件のデータをValdにInsertします.  
ここで, Insertの時間短縮のため, 複数のデータを用いてInsertを行うMultiInsertを使用しています.


```python
# Insert 100*1000 vector
count = 100
length = 1000

for c in tqdm(range(count)):
    ireqs = []
    for key, vec in vectors[c*length:(c+1)*length]:
        ivec = payload_pb2.Object.Vector(id=key, vector=vec)
        icfg = payload_pb2.Insert.Config(skip_strict_exist_check=True)
        ireq = payload_pb2.Insert.Request(vector=ivec, config=icfg)
        ireqs.append(ireq)    
    imreq = payload_pb2.Insert.MultiRequest(requests=ireqs)
    istub.MultiInsert(imreq)
```


      0%|          | 0/100 [00:00<?, ?it/s]


### Search

次に, 先程Insertしたデータを使用してSearchを行います.  
Insert時と同様, Search用のstubを作成します.


```python
# create stub
sstub = search_pb2_grpc.SearchStub(channel)
```

SearchのRequestを作成し, stubを用いて, __"テスト"__ に類似したテキストを検索します.

※*検索結果が0件または極端に少ない場合, Valdの自動Indexingにより検索結果が返却されていない可能性があるため, Indexing完了のため数秒待機し, 再度Searchを行ってください.*


```python
svec = vectors.query("テスト")
scfg = payload_pb2.Search.Config(num=10, radius=-1.0, epsilon=0.01, timeout=3000000000)
sreq = payload_pb2.Search.Request(vector=svec, config=scfg)

response = sstub.Search(sreq)
pd.DataFrame(
    [(result.id, result.distance) for result in response.results],
    columns=["id", "distance"])
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>distance</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>test</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>1</th>
      <td>テスト</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>2</th>
      <td>テスト結果</td>
      <td>0.289474</td>
    </tr>
    <tr>
      <th>3</th>
      <td>学年末</td>
      <td>0.438919</td>
    </tr>
    <tr>
      <th>4</th>
      <td>追試</td>
      <td>0.448114</td>
    </tr>
    <tr>
      <th>5</th>
      <td>試験</td>
      <td>0.451455</td>
    </tr>
    <tr>
      <th>6</th>
      <td>模試</td>
      <td>0.455780</td>
    </tr>
    <tr>
      <th>7</th>
      <td>模擬試験</td>
      <td>0.476866</td>
    </tr>
    <tr>
      <th>8</th>
      <td>考査</td>
      <td>0.491180</td>
    </tr>
    <tr>
      <th>9</th>
      <td>期末</td>
      <td>0.505280</td>
    </tr>
  </tbody>
</table>
</div>



また, Valdは既に入力済みのデータに対して, idに紐づくベクトルを用いて検索を行うSearch By IDにも対応しています.  
以下で, 既にInsert済みのid="test"に紐づくベクトルを使用し, 近似近傍探索による検索を行います.


```python
# Search By ID
sireq = payload_pb2.Search.IDRequest(id="test",config=scfg)

response = sstub.SearchByID(sireq)
pd.DataFrame(
    [(result.id, result.distance) for result in response.results],
    columns=["id", "distance"])
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>distance</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>test</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>1</th>
      <td>テスト</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>2</th>
      <td>テスト結果</td>
      <td>0.289474</td>
    </tr>
    <tr>
      <th>3</th>
      <td>学年末</td>
      <td>0.438919</td>
    </tr>
    <tr>
      <th>4</th>
      <td>追試</td>
      <td>0.448114</td>
    </tr>
    <tr>
      <th>5</th>
      <td>試験</td>
      <td>0.451455</td>
    </tr>
    <tr>
      <th>6</th>
      <td>模試</td>
      <td>0.455780</td>
    </tr>
    <tr>
      <th>7</th>
      <td>模擬試験</td>
      <td>0.476866</td>
    </tr>
    <tr>
      <th>8</th>
      <td>考査</td>
      <td>0.491180</td>
    </tr>
    <tr>
      <th>9</th>
      <td>期末</td>
      <td>0.505280</td>
    </tr>
  </tbody>
</table>
</div>



### Update

ここでは, idに紐づくInsert済みのデータを更新するUpdateを行います.  
Updateを行うため, 同様にstubを作成します.


```python
# create stub
ustub = update_pb2_grpc.UpdateStub(channel)
```

id="test"に紐づくデータを __"テスト"__ から __"test"__ のベクトルに更新します.


```python
uvec = payload_pb2.Object.Vector(id="test", vector=vectors.query("test"))
ucfg = payload_pb2.Update.Config(skip_strict_exist_check=True)
ureq = payload_pb2.Update.Request(vector=uvec, config=ucfg)

ustub.Update(ureq)
```




    name: "vald-agent-ngt-0"
    uuid: "test"
    ips: "127.0.0.1"
    ips: "127.0.0.1"
    ips: "127.0.0.1"
    ips: "127.0.0.1"
    ips: "127.0.0.1"



Updateの確認のため, id="test"に紐づくデータに対して検索を行い, __"テスト"__ の結果と異なることを確認します.

※*Insert済みのデータによっては, 同様の結果となる場合もあります.*  
※*Searchの際と同様に, Indexingなどのタイミングによっては値が変更されていない可能性があるため, 時間をおいて再度検索を行ってください.*


```python
# Search By ID
sireq = payload_pb2.Search.IDRequest(id="test", config=scfg)

response = sstub.SearchByID(sireq)
pd.DataFrame(
    [(result.id, result.distance) for result in response.results],
    columns=["id", "distance"])
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>distance</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>test</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>1</th>
      <td>コマンドライン</td>
      <td>0.513686</td>
    </tr>
    <tr>
      <th>2</th>
      <td>PHP</td>
      <td>0.529631</td>
    </tr>
    <tr>
      <th>3</th>
      <td>cygwin</td>
      <td>0.532150</td>
    </tr>
    <tr>
      <th>4</th>
      <td>ソースファイル</td>
      <td>0.532741</td>
    </tr>
    <tr>
      <th>5</th>
      <td>インクルード</td>
      <td>0.535722</td>
    </tr>
    <tr>
      <th>6</th>
      <td>config</td>
      <td>0.539216</td>
    </tr>
    <tr>
      <th>7</th>
      <td>スクリプト</td>
      <td>0.539708</td>
    </tr>
    <tr>
      <th>8</th>
      <td>vim</td>
      <td>0.539814</td>
    </tr>
    <tr>
      <th>9</th>
      <td>インストーラー</td>
      <td>0.550205</td>
    </tr>
  </tbody>
</table>
</div>



### Remove

最後に, 入力されたデータを削除するRemoveを行います.  
Remove用のstubを作成します.


```python
# create stub
rstub = remove_pb2_grpc.RemoveStub(channel)
```

id="test"に紐づくデータを削除します.


```python
rid = payload_pb2.Object.ID(id="test")
rcfg = payload_pb2.Remove.Config(skip_strict_exist_check=True)
rreq = payload_pb2.Remove.Request(id=rid, config=rcfg)

rstub.Remove(rreq)
```




    name: "vald-agent-ngt-2"
    uuid: "test"
    ips: "127.0.0.1"
    ips: "127.0.0.1"
    ips: "127.0.0.1"
    ips: "127.0.0.1"
    ips: "127.0.0.1"



データが削除されたかどうかを確認するため, Existを用いてデータが存在するかどうかをチェックします(データが存在しない場合Errorを返します).


```python
# Exists
ostub = object_pb2_grpc.ObjectStub(channel)

oid = payload_pb2.Object.ID(id="test")
try:
    ostub.Exists(oid)
except grpc._channel._InactiveRpcError as _:
    print("vector is not found")
```

    vector is not found


以上が, Valdの基礎的な動作であるInsert/Search/Update/Removeを用いた検索の例です.

---

## Advanced

近似近傍探索を用いたテキスト検索の実験的な例として, 以下の内容を行います.

- Word Analogies

### Word Analogies

単語のベクトル表現を用いて, 加法/減法を行い, 意味的に類似した単語を検索します.  
例として, ”王"-"男"+"女"="女王"となるベクトル表現が得られ, "女王"に意味的に類似した単語が結果に含まれることを期待するテキスト検索を以下を示します. 


```python
svec = vectors.query("王") - vectors.query("男") + vectors.query("女")
scfg = payload_pb2.Search.Config(num=10, radius=-1.0, epsilon=0.01, timeout=3000000000)
sreq = payload_pb2.Search.Request(vector=svec, config=scfg)
response = sstub.Search(sreq)

pd.DataFrame(
    [(result.id, result.distance) for result in response.results],
    columns=["id", "distance"])
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>distance</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>王</td>
      <td>0.125656</td>
    </tr>
    <tr>
      <th>1</th>
      <td>王女</td>
      <td>0.411794</td>
    </tr>
    <tr>
      <th>2</th>
      <td>王様</td>
      <td>0.423530</td>
    </tr>
    <tr>
      <th>3</th>
      <td>女王</td>
      <td>0.427799</td>
    </tr>
    <tr>
      <th>4</th>
      <td>王妃</td>
      <td>0.437568</td>
    </tr>
    <tr>
      <th>5</th>
      <td>王家</td>
      <td>0.443276</td>
    </tr>
    <tr>
      <th>6</th>
      <td>妃</td>
      <td>0.462706</td>
    </tr>
    <tr>
      <th>7</th>
      <td>王位</td>
      <td>0.479176</td>
    </tr>
    <tr>
      <th>8</th>
      <td>国王</td>
      <td>0.493776</td>
    </tr>
    <tr>
      <th>9</th>
      <td>王族</td>
      <td>0.498267</td>
    </tr>
  </tbody>
</table>
</div>



また, 上記とは異なる例も示します(ref: [fastText tutorial#word-analogies](https://fasttext.cc/docs/en/unsupervised-tutorial.html#word-analogies)).


```python
svec = vectors.query("psx") - vectors.query("sony") + vectors.query("nintendo")
scfg = payload_pb2.Search.Config(num=10, radius=-1.0, epsilon=0.01, timeout=3000000000)
sreq = payload_pb2.Search.Request(vector=svec, config=scfg)
response = sstub.Search(sreq)

pd.DataFrame(
    [(result.id, result.distance) for result in response.results],
    columns=["id", "distance"])
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>distance</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>nintendo</td>
      <td>0.267808</td>
    </tr>
    <tr>
      <th>1</th>
      <td>wiiu</td>
      <td>0.366355</td>
    </tr>
    <tr>
      <th>2</th>
      <td>プレステ</td>
      <td>0.369558</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Wii</td>
      <td>0.381307</td>
    </tr>
    <tr>
      <th>4</th>
      <td>ニンテンドウ</td>
      <td>0.394275</td>
    </tr>
    <tr>
      <th>5</th>
      <td>xbox</td>
      <td>0.424258</td>
    </tr>
    <tr>
      <th>6</th>
      <td>PS</td>
      <td>0.426417</td>
    </tr>
    <tr>
      <th>7</th>
      <td>イステーション</td>
      <td>0.427068</td>
    </tr>
    <tr>
      <th>8</th>
      <td>テンドー</td>
      <td>0.431633</td>
    </tr>
    <tr>
      <th>9</th>
      <td>ファミコン</td>
      <td>0.443734</td>
    </tr>
  </tbody>
</table>
</div>



---

以上で __"Vald Similarity Search using chiVe Dataset"__ notebookは終了です.  
Valdに興味を持っていただきありがとうございました.

更に詳しく知りたい方は, Githubやofficial web siteをご活用ください:
- https://github.com/vdaas/vald
- https://vald.vdaas.org/
