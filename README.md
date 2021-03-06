# AI law

판례 내에 있는 핵심 정보를 추출하는 것이 목적입니다.<br/>
본 프로젝트에서 정의한 핵심 정보란 아래와 같다
* "누가 범죄를 저질렀는지"
* "언제 범죄를 저질렀는지"
* "어디서 범죄를 저질렀는지"
* "어떤 범죄를 저질렀는지"

정보를 추출하기 위해 기계독해 방식으로 문제에 접근하였고, 데이터를 그에 맞게 SQuAD형식으로 만들었습니다.<br/>
판례 내에 있는 범죄 사실이라는 사건의 경위를 담고 있는 단락을 context로 하고,
핵심 정보를 question과 answer을 통해 태깅했습니다.

Etri의 KorBERT와 구글의 Multilingual BERT를 사용하여 성능을 비교해봤고,<br/>
형태소 분석한 뒤 KorBERT를 통해 학습시키는 방법이 대부분의 상황에서 좋은 성능을 냈습니다.

## 1. preprocess data

```
python3 preprocess.py 
--input_file ../data/law.xlsx
--output_dir ../data/
```

data-processing 폴더 안에 있는 preprocess.py 실행시 excel 파일을 squad 데이터 형식의 데이터셋으로 변환

* train.json : 70% data
* test.json : 30% data
* law.json  : 100% data


## 2. multilingual-BERT

``` 
python3 run_squad.py 
--model_type bert 
--model_name_or_path bert-base-multilingual-uncased 
--do_train 
--do_lower 
--do_eval 
--train_file ../../data/train.json 
--predict_file ../../data/test.json 
--per_gpu_train_batch_size 12 
--learning_rate 3e-5 
--num_train_epochs 2.0 
--max_seq_length 384 
--doc_stride 128 
--overwrite_output 
--overwrite_cache 
--output_dir ../../outputDir/multilingual/ 
--save_steps 5000 
--make_cache_file False
```

multilingualBERT/runningcode 폴더 안에 있는 run_squad.py 실행시 입력 인자로 받은 train_file을 이용하여 학습하고, predict_file로 모델 평가를 한다. <br/>
predict_file에 대한 f1 score와 exact match score를 결과로 출력창에 보여준다.

output_dir인 결과 폴더 안에는 예측 결과가 들어 있는 predictions.json,<br/>
fine tuning 을 거친 모델인 pytorch_model.bin 등이 생성된다.


## 3. ETRI-BERT

Etri의 KorBERT를 사용하기 위해서는 [이 사이트](http://aiopen.etri.re.kr/service_dataset.php)에서 사용허가협약서를 작성한 뒤 다운로드 받아야 합니다.<br/>
pretrained model은 개인이 따로 공개할 수 없어 업로드하지 않겠습니다.<br/>
모델을 다운받으신 후 EtriBERT/ 폴더에 넣어주세요.


### 3-1. 형태소 분석용 파일을 생성

```
python3 tokenizing.py
--openapi_key your key
--input_file ../../data/law.json 
--output_file ../../data/tokenizing.json
```

EtriBERT를 사용하기 위해서는 Etri에서 제공하는 형태소 분석 API를 사용하여 형태소 분석된 문장을 인풋으로 넣어줘야한다.<br/>
하지만 형태소 분석 API 호출은 하루 당 5000건으로 제한돼있습니다.<br/>
그래서 API를 미리 호출시켜 학습 시 사용될 입력 문장을 형태소 분석된 문장으로 바꾼 뒤 파일인 tokenizing.json에 저장했습니다.

* 모든 데이터가 들어있는 law.json 파일을 불러와서 API호출 후 tokenizing.json 파일을 생성


### 3-2. EtriBERT를 실행

```
python3 run_squad_ETRI.py 
--openapi_key `your key` 
--bert_model .. 
--train_file ../../data/train.json 
--predict_file ../../data/test.json 
--output_dir ../../outputDir/EtriBERT 
--tokenized_file ../../data/tokenizing.json 
--do_train 
--do_predict
```

EtriBERT 폴더 안에 있는 run_squad_ETRI.py 실행시 train_file을 학습시켜, test_file을 f1 score와 exact match score를 통해 평가합니다.<br/>
결과 폴더 안에 예측 결과가 들어 있는 predictions.json과 점수가 들어있는 result.txt, fine tuning된 모델인 pytorch_model.bin가 생성됩니다.

> openapi_key는 etri API DATA 서비스 포털에서 발급 가능합니다.<br/>
> 반드시 발급받은 후 key값을 넣어주세요.

## 4. Result

한국어만을 사용해 학습한 뒤 형태소 분석기를 사용한 Etri의 KorBERT가 Multilingual BERT에 비해 전체적으로 높은 성능을 보였다.

![](./image/experiment_result.png)