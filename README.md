# ChineseWebText: Large-Scale High-quality Chinese Web Text Extracted with Effective Evaluation Model

This directory contains the ChineseWebText dataset, and the EvalWeb tool-chain to process CommonCrawl Data for high-quality chinese data. Our ChineseWebText dataset is publicly available on huggingface.[(here)]https://huggingface.co/datasets/CASIA-LM/ChineseWebText).

## ChineseWebText

- ### Dataset Overview

We release the latest and largest Chinese dataset **ChineseWebText**, which consists of **1.42 TB** (See Table 1) data and each text is assigned a quality score, facilitating LLM researchers to select data according to a new quality threshold. We also release a much cleaner subset of **600 GB** Chinese texts with quality exceeding **90%** .

<div align="center">
  <img src=".\assets\Overview_of_output_datasets.png" width="50%" />
</div>

- ### Data Example

  ```json
  {
    "title": "潍坊银行2021年上半年净利润同比增长29.57% 不良率降至1.10%_财经_中国网",
    "score": 0.95,
    "text": "潍坊银行2021年上半年净利润同比增长29.57% 不良率降至1.10%\n中国网财经8月24日讯 潍坊银行昨日披露2021年二季度信息报告显示，截至2021年6月末，潍坊银行资产总额1920.44亿元，较上年末增长9.34%；负债总额1789.16亿元，较上年末增长10.54%。2021年上半年，潍坊银行实现净利润6.09亿元，同比增长29.57%。\n资产质量方面，截至2021年6月末，潍坊银行不良贷款率1.10%，较上年末下降0.13个百分点。\n资本金方面，截至2021年6月末，潍坊银行资本充足率、核心一级资本充足率、一级资本充足率分别为11.66%、7.89%、10.13%，分别较上年末下降1.89、0.89、1.15个百分点。",
    "url": "http://finance.china.com.cn/news/special/2021bnb/20210824/5638343.shtml",
    "source_domain": "finance.china.com.cn"
  }
  ```

  - "title":  【string】The title of the data text.
  - "score":  【float】Quality score generated by the quality evaluation model.
  - "text":  【string】Text content of data sample.
  - "url":  【string】External URL, points to the original web address of the text.
  - "source_domain": 【string】The domain name of the source website.

## EvalWeb

### Introduction

We introduce a new complete tool-chain **EvalWeb** (See Figure 1), which could extract high-quality Chinese texts from raw web data.  For the crawled data from web, we first use a preparation module to process them, and then extract the monolingual Chinese data. After that, a preprocessing module will be used to further filter them with mannual crafted rules, including data length, sensitive words, proportion of Chinese characters and so on. Finally, a BERT-based evaluation model will be employed to assess the qualities of filtered data. By this way, we can generate a quality score for each of the text, and then use an appropriate threshold to extract the high-quality data as we required. Furthermore, considering computational cost and efficiency, we also propose to leverage knowledge distillation techniques to train a FastText classifier, which can achieve similar performance with faster efficiency and lower computational costs.

<div align="center">
  <img src=".\assets\BERTEval.png"/>
</div>

### Environment Dependencies

```shell
codescikit-learn==1.3.0
transformers==4.31.0
scipy==1.11.1
numpy==1.24.3
pytorch==2.0.1
jieba==0.42.1
zhconv==1.4.3
fasttext==0.9.2
```

### Stage 1: Data Preparation

#### 1. Deduplication and Language Identification (LID) using CCNet Tools

- Following the work of CCNet, in this module a Hash-based inter-string deduplication method is employed to remove duplicate text from different CommonCrawl snapshots.  Additionally, a well-trained language identification model, which could support 157 languages, is applied to select Chinese data. By this way, we can obtain all the monolingual Chinese text data we required.

- [CCNet Tools](https://github.com/facebookresearch/cc_net)

- Run the script:

  ```shell
   python -m cc_net --config config/my_config_2023-23.json。
  ```

- Outputs:

  ```shell
  /data/mined_split/2023-23/{0-4999}/zh_[head|middle|tail].json.gz
  ```

- config/my_config_2023-23.json：

```json
{
  "hash_in_mem": 10,
  "dump": "2023-23",
  "task_parallelism": 20,
  "num_shards": 5000,
  "mine_num_processes": 20,
  "num_segments_per_shard":-1,
  "lang_whitelist": ["zh","en"],
  "lang_blacklist": [],
  "lang_threshold": 0.5,
  "keep_bucket": [],
  "pipeline": ["dedup", "lid", "keep_lang", "sp", "lm", "pp_bucket", "drop", "split_by_lang"],
  "metadata": "None",
  "execution": "local",
  "output_dir": "data",
  "mined_dir": "mined",
  "target_size": "4G",
  "min_len": 300,
  "cache_dir": "/mnt/data/ccnet_data/commoncrawl"
}
```

#### 2. Filter using blacklist and regular expression matching.

- run python clear_ccnet.py

```sh
python clear_ccnet.py --source /mnt/data/ccnet_clean/cc_net/data/mined_split/2023-23 --target /mnt/data/cc_cleaned
# --source directory of cleaned data after the first step 
# --target directory of data filtered by blacklist and regular expression matching
```

- outputs:

```sh
cleared*.jsonl
cleared_dirty*.jsonl
```

- compress files

```sh
tar -czvf ccnet-2023-23.tar.gz 2023-23
```

### Stage 2:  Preprocessing

This section focuses on extracting high-quality texts from Chinese monolingual web data by using manually crafted rules to filter out violent, pornographic, advertising content, and erroneous characters. The details of the filtering rules are presented in the following:

- #### Text Extraction

Extract text content from `jsonl` file after the data preparation stage.

- #### Data Length

To improve language model training, documents will be filtered out if they have an average line length of fewer than **10** characters or a total text length of less than 200 characters, as such short texts often lack meaningful context and semantic relevance.

- #### Proportion of Characters

We aim to create a high-quality simplified Chinese dataset from web data by eliminating traditional Chinese characters and removing texts with less than **30%** Chinese characters to ensure the dataset is suitable for training large language models.

- #### Sensitive Words

To prevent large language models from generating toxic content, a method is proposed where texts are analyzed for the occurrence of harmful words from a predefined list, and any text with more than **0.5** occurrences of such words per line is classified as toxic and removed from the training dataset.

- #### Internal duplication

To enhance training efficiency and model performance, a subsequent analysis using a 13-gram granularity is conducted to identify and filter out data samples where over **50%** of the character sequences are repetitive in each data entry.

Here is an example command to run the preprocessing stage:

```shell
python preprocess.py --dates 2023-06 2023-14
```

> The **"dates"** parameter passed in corresponds to the folder names of the snapshots generated during the preparation stage.
>
> Then, you will get six subfolders under the corresponding date's folder. These six folders are respectively named **"text_extraction"**, **"length"**,  **"Character"**, **"sensitive"**, **"duplication"** and **"remain"**. The **"text_extraction"** folder contains the results after extracting text from each piece of data, while **"length"**,  **"Character"**, **"sensitive"**, and **"duplication"** correspond to four filtering operations, storing the filtered noise data. The **"remain"** folder stores the remaining data after the preprocessing stage, and these data will subsequently be scored through our evaluation model.

### Stage 3:  Quality Evaluation

In preprocessing procedure, we have used some handcrafted rules to remove the explicit noisy texts from our dataset. However, within the remaining data, there is still a considerable amount of low-quality text data, which cannot be filtered out with handcrafted rules. In order to extract the data of higher quality from them, in this section we further propose to design an evaluation models.

#### Stage 3.1:  BERTEval

#### 1. BERTEval Training Data Composition

<div align="center">
  <img src=".\assets\BERTEval_data_composition.png" width="50%" />
</div>

#### 2. BERTEval Training and Inference

- Step 1: 2-stage Training

  ```shell
  python train.py # stage1  you can modify configs/base_config.json to set hyper-parameters
  python train_ust.py # stage2 you can modify configs/ust_config.json to set hyper-parameters
  ```

- Step 2: Split the previously processed CommonCrawl into multiple shards, where each shard is a JSON file. All shards for a single snapshot are stored in the same path. Refer to the example `util/text_separate.py`.

- Step 3: Run the Python inference script `pred.py` to split each text using delimiters such as newline `\n` or periods into complete paragraphs of a maximum length of 512. Predict the text quality score for each paragraph. The configuration can be modified using `config/pred_config.json`, with key parameters as follows:

  ```shell
  "data_path": ccnet data path
  "output_path": Path to store the scored data
  "num_workers": Number of CPU processes for data preprocessing
  "batch_size": BERT batch size
  "checkpoint": Model checkpoint path
  "tokenizer_path": Path to store BERT tokenizer
  "pretrained_model_path": Pre-trained BERT weights path
  ```

  Other parameters do not require modification. The processed text is stored in multiple JSONL files. Then, run

  ```shell
  python pred.py
  ```

  Step 4: Set the threshold value $T$ and retain text data with a quality threshold greater than $T$. Since the maximum input token limit for bert-base is 512, for longer texts, they are split into multiple text segments. For consecutive text segments in the same document with thresholds greater than $T$, the program automatically concatenates them. This functionality is implemented in the function `text_select_with_pred(file, score_threshold)` in `utils/util.py`.

  Usage:

  ```python
  file = "test\data\cleared0_0000.jsonl"
  score_threshold = 0.99
  selected_data = text_select_with_pred(file, score_threshold)
  ```

### Stage 3.2:  FastText

#### 1. FastText Training Data Composition

<div align="center">
  <img src=".\assets\FastText_data_composition.png" width="50%" />
</div>

### 2. FastText Training and Inference

We provide our FastText training data examples and training script in folder **"fasttext"**. You can download our trained FastText model [here](https://huggingface.co/CASIA-LM/ChineseWebText-fasttext) and replace the existing file located at **"./fasttext/output/model.bin"**.

```shell
cd fasttext
python main.py --mode train --train_file ./data/train.txt --test_file ./data/test.txt
```

> To understand the process of constructing the **"train.txt"** and **"test.txt"** files, please refer to the **"./data/build_data.py"**.
>
> The trained model **"model.bin"** will be stored in the **"output"** folder.

After getting the remaining data after the preprocessing stage(should be stored in path like **"./2023-06/remain"**), you can using our FastText model to score all the data:

```shell
python main.py --mode test --dates 2023-06 2023-14
```

> This step will assign a FastText score to each data entry, with the results being stored in a directory such as **"./2023-06/remain/fasttext"**. Subsequently, you can utilize these scores to filter and extract high-quality data by using a threshold(default set to 0.5).

## Citation

Please cite the paper if you use the data or code in this repo.
```shell
   @misc{chen2023chinesewebtext,
      title={Chinesewebtext: Large-scale high-quality Chinese web text extracted with effective evaluation model}, 
      author={Jianghao Chen and Pu Jian and Tengxiao Xi and Yidong Yi and Chenglin Ding and Qianlong Du and Guibo Zhu and Chengqing Zong and Jinqiao Wang and Jiajun Zhang},
      year={2023},
      eprint={2311.01149},
      archivePrefix={arXiv},
      primaryClass={cs.CL}
}
```
