---
title: 4.ES分词器
date: 2020-03-02 21:03:00
tags:
    - Elaticsearch
categories:
    - 中间件
    - Elaticsearch
---

## 4.1 Analysis

**文本分析,把全文本转换一系列单词(term/token)的过程,也叫作分词**

## 4.2 Analyzer

**分词通过分词器来实现,可以使用ES内置的分词器,也可以按需定制化分词器**

### 4.2.1 Analyzer组成

![](http://dist415.oss-cn-beijing.aliyuncs.com/esalsis.png)

- Character Filters （针对原始文本处理，例如去除html）
- Tokenizer （按照规则切分为单词）
- Token Filter（将切分的的单词进行加工，小写，删除stopwords,增加同义词）

### 4.2.2 Character Filters

- 在Tokenizer之前对文本进行处理，例如增加删除及替换字符。可以配置多个 Character Filters。会影响 Tokenizer的 position 和 offset 信息
- —些自带的 Character Filters
  - HTML strip —出除 html 标签
  - Mapping 一字符串替换
  - Pattern replace —正则匹配替换

### 4.2.3 Tokenizer

- 将原始的文本按照一定的规则，切分为词(term or token)
- Elasticsearch 内置的 Tokenizers

  - whitespace / standard / uax_url_email / pattern / keyword / path hierarchy
- 可以用Java开发插件，实现自己的Tokenizer

### 4.2.4 Token Filters 

- 将Tokenizer输出的单词（term ）,进行增加，修改，删除
- 自带的 Token Filters
  - Lowercase / stop / synonym （添加近义词）

### 4.2.2 ES内置分词器

- **Standard Analyzer** 一默认分词器，按词切分，小写处理
- **Simple Analyzer** 一按照非字母切分（符号被过滤），小写处理
- **Stop Analyzer** —小写处理，停用词过滤（the, a, is）
- **Whitespace Analyzer** —按照空格切分，不转小写
- **Keyword Analyzer** 一不分词，直接将输入当作输出
- **Patter Analyzer** -正则表达式，默认\W+ （非字符分隔）
- **Language** 一提供了30多种常见语言的分词器
- **Customer Analyzer**自定义分词器

## 4.3 中文分词

常用的中文分词器如下:

- **[IK](https://github.com/medcl/elasticsearch-analysis-ik)**:支持自定义词库，支持热更新分词字典
- **[THULAC](https://github.com/microbun/elasticsearch-thulac-plugin)**:清华大学自然语言处理和社会人文计算实验室的一套中文分词器 



