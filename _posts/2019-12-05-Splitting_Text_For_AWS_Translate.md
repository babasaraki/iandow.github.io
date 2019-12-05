---
layout: post
title: Splitting Text for AWS Translate
tags: [aws, translate, natural language, python]
bigimg: /img/abstract-1278060_1920.jpg
---
<!-- image source: https://pixabay.com/illustrations/abstract-geometric-world-map-1278060/ -->
<!-- Free for commercial use. No attribution required -->

This post describes how to split large text documents into chunks small enough they can be processed by AWS Translate. When splitting text for AWS Translate you should avoid splitting words or sentences so you don't break grammatical correctness of the source text. I'll show how to do this with an AWS Lambda function that uses the Python NLTK library for detecting sentence boundaries.

# What is AWS Translate

<img src="http://iandow.github.io/img/flags.png" width="30%" style="margin-left: 15px" align="right">

[AWS Translate](https://docs.aws.amazon.com/translate/latest/dg/what-is.html) is a service for translating text on the Amazon Web Services (AWS) platform. It's designed to be used programmatically and supports interfaces in Python, Java, AWS Mobile SDKs, and the AWS CLI. It currently supports 54 different languages. I have been using AWS Translate to translate video transcripts in an application I've built with a few colleagues, called the [Media Insights Engine](https://github.com/awslabs/aws-media-insights-engine). We use these translations as content for which users can query in a search engine designed to index massively large video archives.

<img src="http://iandow.github.io/img/MIE_bienvenido.png" width="70%">

# What are AWS service limits?

Service limits are built into every AWS Service. They control things like how much data you can include in service requests, how often you can send requests, how long your requests can run, etc. For example, AWS Lambda, a popular serverless computing platform, limits functions to running no longer than 15 minutes and will shutdown functions that exceed that limit without warning. Other AWS services return errors that explicitly identify the service limit that was exceeded. For example, if you send AWS Translate input text that is too long then you'll see an error like this:

```
An error occurred (TextSizeLimitExceededException) Input text size exceeds limit. Max length of request text allowed is 5000 bytes while in this request the text size is 5074 bytes'
```

When you need a service to process more data than you're allowed to include in a single job then you should try to split the workload into multiple smaller parts. We can see in the error shown above that AWS Translate can accept no more than 5000 bytes (which is close to but not exactly 5000 characters; more on that later). If you're working with a text document that is longer than 5000 bytes then you could try to split the source text into chunks smaller than 5000 bytes, call Translate for each chunk, and combine the translated results once complete.

However, in order to maintain grammatical integrity of the source text it must not be split in the middle of words or sentences. The simplest way to locate sentence boundaries involves looking for a period followed by a capitalized word. A more accurate strategy will also consider language specific abbreviations that include a period but don't necessarily end a sentence, such as the English title "Missus" abbreviated as Mrs. or German title "Frau" abbreviated as Fr. 

# Splitting text with the Natural Language Toolkit for Python 

The Natural Language Toolkit (NLTK) for Python provides a convenient way to split text into sentences with this strategy. The following Python code shows how to download NLTK tokenizers and find sentence boundaries for a block of text. In this example, I create a tokenizer using an English language data file. You can find data files for other languages under the `tokenizers/punkt/` directory. In the following code we download NLTK data files to `/tmp/` so that it can run in AWS Lambda where `/tmp` is the only writable filesystem.

{% highlight python %}
# Tell the NLTK data loader to look for files in /tmp/
nltk.data.path.append("/tmp/")
# Download NLTK tokenizers to /tmp/
# We use /tmp because that's where AWS Lambda provides write access to the local file system.
nltk.download('punkt', download_dir='/tmp/')
# Load the English language tokenizer
tokenizer = nltk.data.load('tokenizers/punkt/english.pickle')
# Split input text into a list of sentences
sentences = tokenizer.tokenize(transcript)
print("Input text length: " + str(len(transcript)))
print("Number of sentences: " + str(len(sentences)))
translated_text = ''
source_text_chunk = ''
for sentence in sentences:
    # Translate can handle 5000 unicode characters but we'll process no more than 4000
    # just to be on the safe side.
    if (len(sentence) + len(source_text_chunk) < 4000):
        source_text_chunk = source_text_chunk + ' ' + sentence
    else:
        print("Translation input text length: " + str(len(source_text_chunk)))
        translation_chunk = translate_client.translate_text(Text=source_text_chunk,SourceLanguageCode=source_lang,TargetLanguageCode=target_lang)
        print("Translation output text length: " + str(len(translation_chunk)))
        translated_text = translated_text + ' ' + translation_chunk["TranslatedText"]
        source_text_chunk = sentence
# Translate the final chunk of input text
print("Translation input text length: " + str(len(source_text_chunk)))
translation_chunk = translate_client.translate_text(Text=source_text_chunk,SourceLanguageCode=source_lang,TargetLanguageCode=target_lang)
print("Translation output text length: " + str(len(translation_chunk)))
translated_text = translated_text + ' ' + translation_chunk["TranslatedText"]
print("Final translation text length: " + str(len(translated_text)))
{% endhighlight %}

# Accommodating international characters

In the above code example I limited the translation input size to 4000 characters even though Translate's service limit allows 5000 unicode characters because in practice, I found that if you're close to 5000 characters and your text contains non-English characters, like 'ü' or 'ß', then Translate will say your text size is too large. I'm not sure exactly what's going on there, but just to be on the safe side, I limit the input size to 4000 characters.

# Summary

In this blog post I showed how to respect sentence boundaries when splitting large text documents into chunks  that can be processed by AWS Translate. With this strategy the limits to what you can translate are limitless! That is, as long as you stay withing the bounds of service guidelines and limits ;-). For more information about AWS Translate see the developer guide here: https://docs.aws.amazon.com/translate/latest/dg/what-is.html.

<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/16">https://github.com/iandow/iandow.github.io/issues/16</a>.</p>

<div class="main-explain-area padding-override jumbotron">
  <img src="http://iandow.github.io/img/mustache-udnie.cropped.jpg" width="120" style="margin-left: 15px" align="right">
  <p class="margin-override font-override" style="margin-left: 15px">
  	I’m fundraising for <a href="https://us.movember.com">Movember</a>! Please consider donating $5 to help me support Men's health: <a href="https://mobro.co/iandownard">https://mobro.co/iandownard</a>
  </p>
  <img src="http://iandow.github.io/img/movember.jpg" width="60" style="margin-left: 30px" align="right">
  <br>
  <div id="paypalbtn">
    <a class="btn btn-primary btn" href="https://mobro.co/iandownard">Donate to Movember</a>
  </div>
</div>
