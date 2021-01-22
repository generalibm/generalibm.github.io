---
title: What I Have Learnt From the Dissertation of Danqi
date: 2019-03-03 16:53:28
tags: dissertation
keyword: [Reading comprehension, NLP, Dissertation for doctor degree]
categories: Thinking
description: It is some ideas about what I have recently been reading and thinking.
toc: true
---

## What I have learnt from the dissertation of Danqi
Not many doctors' dissertations could be big bang, but [Danqi Chen](https://github.com/danqi)'s did. Recently, the doctor from Stanford NLP Group has attracted a lot of attention, including my friends and me.

The library of Stanford said her doctor dissertation named [Neural Reading Comprehension and Beyond](https://stacks.stanford.edu/file/druid:gd576xb1833/thesis-augmented.pdf) that reaches 156 pages had been read over 1,000 times in four days, since it was uploaded, which has becoming the most popular dissertation for doctor degree in recent decade.

And here I am writing what I have learnt from it.

<!--more-->

### Why this paper

I decided to write something about my daily learning and thinking down when I realized that did not keep doing it in the notebook or on my blog.

Actual, I have completed the course [**how to write a good research paper**](http://www.xuetangx.com/courses/course-v1:Tsinghua+20150001+sp/about) and got a **excellent** grade, which was encouraged from [Li ZHU](https://github.com/zhuli19901106)，Siraj , etc. And then I am looking for some papers to learn something fresh and hope to try a new era. I downloaded some about cyber security but no one about artificial intelligence until I met Danqi's, which could be a perfect example on how to write a good research paper in my mind.

### What it is about

The paper is trying to build computer systems to read a passage of text and answer comprehension questions by focusing on neural reading comprehension. It consists of two parts: the first one is called foundations to tell readers an overview of reading comprehension and her group's effort of their models which understands what they have actually learned, more of that, they summarized recent advances and discussed future directions and open questions in that field; the second one is called applications to tell readers two new directions they pioneered: 1)how they can combine information retrieval techs with neural reading comprehension to tackle large-scale open-domain question answering; and 2)how they can build conversational question answering systems form current single-turn, span-based reading comprehension models, what's more, they implemented those ideas in the DrQA and CoQA projects and they demonstrate the effectiveness of those approaches.

### Who is Danqi Chen

[She](https://cs.stanford.edu/~danqi/) graduated from Yao class of Tsinghua university in 2012. During the time, she won the silver medal in the world final of ACM/ICPC. 

Before going to Tsinghua, she studied in [Yali](https://en.wikipedia.org/wiki/Yali_High_School) Changsha, in which she presented [plug-like dynamic programming](https://cs.stanford.edu/~danqi/misc/dynamic-programming.pdf) and [CDQ's divide-and-conquer](https://cs.stanford.edu/~danqi/misc/divide-and-conquer.pdf) .

After studying in Tsinghua, she went to Stanford. Now she currently visiting Facebook AI Research (Seattle) and University of Washington. And she will be joining the Computer Science Department at Princeton University as an assistant professor in Fall 2019.

### What I have learnt

Specifically, what impressed me most, firstly, is understanding Reading Comprehension from a fool computer view, 

>
> Alyssa got to the beach after a long trip, She's from Charlotte. She traveled from Atlanta. She's now in Miami. She went to Miami to visit some friends. But she wanted some time to herself at the beach, so she went there first. After going swimming and laying out, she went to her friend Ellen's house. Ellen greeted Alyssa and they both had some lemonade to drink. Alyssa called her friends Kristen and Rachel to meet at Ellen's house. The girls traded stories and caught up on their lives. It was a happy time for everyone. The girls went to a restaurant for dinner. The restaurant had a special on catfish. Alyssa enjoyed the restaurant's special. Ellen ordered a salad. Kristen had soup.Rachel had a steak. After eating, the ladies went back to Ellen's house to have fun. The had lots of fun. They stayed the night because they were tired. Alyssa was happy to spend time with her friend again.
>
>
> - Question: What city is Alyssa in?
> - Answer: Miami
> - Quesion: what did Alyssa eat at the restaurant?
> - Answer: catfish
> - Question: How may friends does Alyssa have in this story?
> - Answer: 3
>

so how many steps should a machine process before answer the above questions?

> - part-of-speech tagging. It requires our machines to understand that, for example, in the first sentence *Alyssa got to the beach after a long trip.*, *Alyssa* is a proper noun, *beach* and *trip* are common nouns. *got* is a verb in its past tense, *long* is an adjective, *after* is a preposition.
> - named entity recognition. Our machines also should understand that *Alyssa*, *Ellen*, *Kristen* are the names of people in the story while *Charlotte*, *Atlanta* and *Miami* are the names of locations.
> - syntactic parsing. To understand the meaning of each single sentence, our machines also need to understand the relationship between words, or the syntactical (grammatical) structure. Taking the first sentence *Alyssa got to the beach after a long trip.* as an example again, the machines should understand that *Alyssa* is the subject, and *beach* is object of the verb *got*, while *after a long trip* as a whole is a prepositional phrase which describes a temporal relationship with the verb.
> - coreference resolution. Furthermore, our machines even need to understand the interplay between sentences. For example, the mention *She* in the sentence *She's now in Miami* refers to *Alyssa* mentioned in the first sentence, while the mention *The girls* refers to *Alyssa, Ellen, Kristen and Rachel* in the previous sentences.

I did a lot of reading comprehensions during the school time, both English and Chinese class, but I have never thought them from the view like this even when I prepared the examinations in high school. Actually, it works effectively when learning a new language, and I would like to try it when I learn my third language. Still now, I have to read a lot of papers and blogs from the Internet, I got suck sometime when they were not tiny or simple. 

So secondly, what I learnt from it is that to try to write things down in simple way, which is what I am doing, this paper is just a kind of timely rain in my almost dry land.

And thirdly, I knew that a doctor dissertation is not easy and also not difficult for me, and I know what it should cover from the outline and something else even though I am trying to be a doctor. If I have a chance, I will not afraid of it.
