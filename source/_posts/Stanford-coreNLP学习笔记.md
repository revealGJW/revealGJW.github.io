---
title: Stanford coreNLP学习笔记
date: 2017-12-11 02:53:19
tags:
  - 中文分词
  - NLP
categories:
  - 机器学习
---
stanford coreNLP是目前Java常用的分词器。使用它需要JDK1.8。
### 项目配置
#### maven依赖
```
<dependency>
 <groupId>edu.stanford.nlp</groupId>
 <artifactId>stanford-corenlp</artifactId>
 <version>3.8.0</version>
</dependency>
<dependency>
 <groupId>edu.stanford.nlp</groupId>
 <artifactId>stanford-corenlp</artifactId>
 <version>3.8.0</version>
 <classifier>models</classifier>
</dependency>
<!--中文module包-->
<dependency>
 <groupId>edu.stanford.nlp</groupId>
 <artifactId>stanford-corenlp</artifactId>
 <version>3.8.0</version>
 <classifier>models-chinese</classifier>
</dependency>
```
#### properties文件
方式1：
将models-chinese包内的StanfordCoreNLP-chinese.properties拷贝到resources目录下
代码中添加：
```
StanfordCoreNLP nlp = new StanfordCoreNLP("StanfordCoreNLP-chinese.properties");
```
方式2：
用自定义Properties类生成
```
Properties props = new Properties();
props.setProperty("annotators", "tokenize, ssplit, pos, lemma, ner, parse, dcoref");
StanfordCoreNLP nlp = new StanfordCoreNLP(props);
```
*自定义词典可在properties文件内配置：
```
segment.serDictionary = edu/stanford/nlp/models/segmenter/chinese/dict-chris6.ser.gz,mydic.txt
//在resources目录下新建mydic.txt文件保存自定义词典，每行一词；自定义的NER文件同样可以这样配置
```
### 基本概念
corenlp中对文本的一次处理称为一个pipeline，annotators代表一个处理节点，类似于函数，如segment切词、ssplit句子切割（将一段话分为多个句子）、pos词性、ner实体命名、regexner是用自定义正则表达式来标注实体类型、parse是句子结构解析。
通过properties配置各annotator的属性。
每个处理生成一个Annotation，本质是一个Map，可通过不同的key获取不同annotator的结果。结果是一个CoreMap的List。
具体见Demo。
```
Demo
import edu.stanford.nlp.ling.CoreAnnotations;
import edu.stanford.nlp.ling.CoreAnnotations.NamedEntityTagAnnotation;
import edu.stanford.nlp.ling.CoreAnnotations.PartOfSpeechAnnotation;
import edu.stanford.nlp.ling.CoreAnnotations.TextAnnotation;
import edu.stanford.nlp.ling.CoreLabel;
import edu.stanford.nlp.ling.HasWord;
import edu.stanford.nlp.pipeline.Annotation;
import edu.stanford.nlp.pipeline.StanfordCoreNLP;
import edu.stanford.nlp.process.*;
import edu.stanford.nlp.tagger.maxent.MaxentTagger;
import edu.stanford.nlp.util.CoreMap;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.StringReader;
import java.util.List;

public class SParserTest {

    private static final String text = "碳碳键键能能否否定定律四";


    public void testPipeline() {
        StanfordCoreNLP pipeline = new StanfordCoreNLP("StanfordCoreNLP-chinese.properties");
        Annotation annotation = pipeline.process("腾讯公司的马化腾特别的厉害。");


        List<CoreMap> sentences = annotation.get(CoreAnnotations.SentencesAnnotation.class);
        CoreMap sentence = sentences.get(0);


        List<CoreLabel> tokens = sentence.get(CoreAnnotations.TokensAnnotation.class);
        System.out.println("分词" + "\t " + "标注" + "\t " + "实体识别");
        System.out.println("-----------------------------");
        for (CoreLabel token : tokens) {
            String word = token.getString(TextAnnotation.class);
            String pos = token.getString(PartOfSpeechAnnotation.class);
            String ner = token.getString(NamedEntityTagAnnotation.class);
            System.out.println(word + "\t " + pos + "\t " + ner);
        }
    }

    public void testTagger() {
        MaxentTagger tagger = new MaxentTagger("edu/stanford/nlp/models/pos-tagger/chinese-distsim/chinese-distsim.tagger");
        String text = "碳碳键 键能 能否 否定 定律四";
        // text is splitted with space internally
        List<List<HasWord>> sentences = tagger.tokenizeText(new BufferedReader(new StringReader(text)));
        for (List<? extends HasWord> sentence : sentences) {
            List<edu.stanford.nlp.ling.TaggedWord> tSentence = tagger.tagSentence(sentence);
            System.out.println(tSentence);
        }
    }


    public void testTokenizer() throws IOException {

        for(String s : ChineseDocumentToSentenceProcessor.fromPlainText(text)) {
            System.out.println(s);
        }

        //English tokenization
        PTBTokenizer<CoreLabel> ptbTokenizer = new PTBTokenizer<CoreLabel>(new StringReader("i love machine learning and i'm stupid"),
                new CoreLabelTokenFactory(), "");
        while(ptbTokenizer.hasNext()) {
            CoreLabel word = ptbTokenizer.next();
            System.out.println(word);
        }
    }

    public void testSegmenter() throws IOException {
        StanfordCoreNLP nlp = new StanfordCoreNLP("StanfordCoreNLP-chinese.properties");
        Annotation document = nlp.process(text);
        List<CoreMap> sentences = document.get(CoreAnnotations.SentencesAnnotation.class);
        for(CoreMap sentence : sentences) {
            for(CoreLabel token : sentence.get(CoreAnnotations.TokensAnnotation.class)) {
                String word = token.get(CoreAnnotations.TextAnnotation.class);
                System.out.println(word);
            }
        }
    }


    public static void main(String[] args) throws IOException {
        SParserTest test = new SParserTest();
        System.out.println("......test tokenizer......");
        test.testTokenizer();
        System.out.println("......test segmenter......");
        test.testSegmenter();
        System.out.println("......test tagger......");
        test.testTagger();
        System.out.println("......test pipeline......");
        test.testPipeline();
    }
}
```
程序输出：
```
......test tokenizer......
碳碳键键能能否否定定律四
i
love
machine
learning
and
i
'm
stupid
......test segmenter......
碳碳键
键能
能否
否定
定律
四
......test tagger......
[碳碳键/NR, 键能/NN, 能否/VV, 否定/VV, 定律四/NN]
......test pipeline......
分词 标注 实体识别
-----------------------------
腾讯 NR ORGANIZATION
公司 NN ORGANIZATION
的 DEC O
马化腾 NR PERSON
特别 JJ O
的 DEG O
厉害 NN O
。 PU O
```
### 常用annotators

|Name|	description|	Generated Annotation|	相关
|-|-|-|
|ssplit|	将文本切分成句子，可配置正则表达式	|SentencesAnnotation
|tokenizier|	分词|TokensAnnotation (list of tokens);<br> CharacterOffsetBeginAnnotation, <br>CharacterOffsetEndAnnotation, <br>TextAnnotation (for each token)
|pos	|词性标注	|PartOfSpeechAnnotation|	[宾州中文数库标记](http://www.voidcn.com/article/p-rqypzyqd-wp.html)
|ner|	命名实体标记	|NamedEntityTagAnnotation<br>NormalizedNamedEntityTagAnnotation|常见的命名实体，包括：名称（PERSON，LOCATION，ORGANIZATION，MISC），数字（MONEY，NUMBER，ORDINAL，PERCENT）和时间（DATE，TIME，DURATION，SET）
使用时首先在properties中配置属性，之后在处理生成的Annotation中传入不同的Annotation名称获取对应的处理结果。