---
title: "SetFitABSA: Few-Shot Aspect Based Sentiment Analysis using SetFit"
thumbnail: /blog/assets/setfit-absa/intel_hf_logo.png
authors:
- user: ronenlap
- user: tomaarsen
- user: lewtun
- user: dkorat
- user: orenpereg
- user: moshew
---

# SetFitABSA: Few-Shot Aspect Based Sentiment Analysis using SetFit




<p align="center">
    <img src="assets/setfit-absa/method.png" width=500>
</p>
<p align="center">
    <em>SetFitABSA is an efficient technique to detect the sentiment towards specific aspects within the text.</em>
</p>

Aspect-Based Sentiment Analysis (ABSA) is the task of detecting the sentiment towards specific aspects within the text. For example, in the sentence, "This phone has a great screen, but its battery is too small", the _aspect_ terms are "screen" and "battery" and the sentiment polarities towards them are Positive and Negative, respectively.

ABSA is widely used by organizations for extracting valuable insights by analyzing customer feedback towards aspects of products or services in various domains. However, labeling training data for ABSA is a tedious task because of the fine-grained nature (token level) of manually identifying aspects within the training samples.

Intel Labs and Hugging Face are excited to introduce SetFitABSA, a framework for few-shot training of domain-specific ABSA models;  SetFitABSA is competitive and even outperforms generative models such as Llama2 and T5 in few-shot scenarios.

Compared to LLM based methods, SetFitABSA has two unique advantages:

<p>🗣 <strong>No prompts needed:</strong> few-shot in-context learning with LLMs requires handcrafted prompts which make the results brittle, sensitive to phrasing and dependent on user expertise. SetFitABSA dispenses with prompts altogether by generating rich embeddings directly from a small number of labeled text examples.</p>

<p>🏎 <strong>Fast to train:</strong> SetFitABSA requires only a handful of labeled training samples; in addition, it uses a simple training data format, eliminating the need for specialized tagging tools. This makes the data labeling process fast and easy.</p>

In this blog post, we'll explain how SetFitABSA works and how to train your very own models using the [SetFit library](https://github.com/huggingface/setfit). Let's dive in!

## How does it work?

<p align="center">
    <img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/method.png" width=700>
</p>
<p align="center">
    <em>SetFitABSA's three-stage training process</em>
</p>

SetFitABSA is comprised of three steps. The first step extracts aspect candidates from the text, the second one yields the aspects by classifying the aspect candidates as aspects or non-aspects, and the final step associates a sentiment polarity to each extracted aspect. Steps two and three are based on SetFit models.

### Training

**1. Aspect candidate extraction**

In this work we assume that aspects, which are usually features of products and services, are mostly nouns or noun compounds (strings of consecutive nouns). We use spaCy (https://spacy.io/) to tokenize and extract nouns/noun compounds from the sentences in the (few-shot) training set. Since not all extracted nouns/noun compounds are aspects, we refer to them as aspect candidates.

**2. Aspect/Non aspect classification**

Now that we have aspect candidates, we need to train a model to be able to distinguish between nouns that are aspects and nouns that are non-aspects. For this purpose, we need training samples with aspect/no-aspect labels. This is done simply by labeling candidates that are labeled as aspects in the training set as TRUE aspects, while the rest are non-aspects and therefore labeled as FALSE as illustrated in the following example:

<p><strong>Training sentence:</strong> "Waiters aren't friendly but the cream pasta is out of this world."</p>
<p><strong>Tokenized:</strong> [Waiters, are, n't, friendly, but, the, cream, pasta, is, out, of, this, world, .]</p>
<p><strong>Extracted aspect candidates:</strong> [Waiters, are, n't, friendly, but, the, cream, pasta, is, out, of, this, world, .]</p>
<p><strong>Gold labels from training set, in BIO format:</strong> [B-ASP, O, O, O, O, O, B-ASP, I-ASP, O, O, O, O, O, .]</p>
<p><strong>Generated aspect/non-aspect Labels:</strong> [Waiters, are, n't, friendly, but, the, cream, pasta, is, out, of, this, world, .]</p>

Now that we have all the aspect candidates labeled, how do we use it to train the candidate aspect classification model? In other words, how do we use SetFit, a sentence classification framework, to classify individual tokens? Well, this is the trick: each aspect candidate is concatenated with the entire training sentence to create a training instance using the following template:

```
aspect_candidate:training_sentence
```

Applying the example above to the template, will generate 3 training instances – two with TRUE labels representing aspect training instances and one with FALSE label representing non-aspect training instance:

| Text                                                                          | Label |
|:------------------------------------------------------------------------------|:------|
| Waiters:Waiters aren't friendly but the cream pasta is out of this world.     | 1     |
| cream pasta:Waiters aren't friendly but the cream pasta is out of this world. | 1     |
| world:Waiters aren't friendly but the cream pasta is out of this world.       | 0     |
| ...                                                                           | ...   |

After generating the training instances, we are ready to use the power of SetFit to train a few-shot domain-specific binary classifier to extract aspects from an input text review.

**3. Sentiment polarity classification**

Once the system extracts the aspects from the text, it needs to associate a sentiment polarity (e.g., positive, negative or neutral) to each aspect. For this purpose, we use a 2nd SetFit model and train it in a similar fashion to the aspect extraction model training as illustrated in the following example:

<p><strong>Training sentence:</strong> "Waiters aren't friendly but the cream pasta is out of this world."</p>
<p><strong>Tokenized:</strong> [Waiters, are, n't, friendly, but, the, cream, pasta, is, out, of, this, world, .]</p>
<p><strong>Gold labels from training set:</strong> [NEG, O, O, O, O, O, POS, POS, O, O, O, O, O, .]</p>

| Text                                                                          | Label |
|:------------------------------------------------------------------------------|:------|
| Waiters:Waiters aren’t friendly but the cream pasta is out of this world.     | NEG   |
| cream pasta:Waiters aren't friendly but the cream pasta is out of this world. | POS   |
| ...                                                                           | ...   |

Note that as opposed to the aspect extraction model, we don’t include non-aspects in this training set because the goal is to classify the sentiment polarity towards real aspects.

## Running inference

At inference time, the test sentence passes through the aspect candidate extraction, resulting in test instances using the template `aspect_candidate:test_sentence`. Next, non-aspects are filtered by the aspect/non-aspect classifier. Finally, the extracted aspects are fed to the sentiment polarity classifier that predicts the sentiment polarity per aspect.

```
Input test sentence: "their dinner specials are fantastic."
```

**Model Output**

```
['their', 'dinner', 'specials', 'are', 'fantastic', '.'],
[‘O’, B-ASP, I-ASP, ‘O’, ‘O’, ‘O’],
[‘O’, ‘positive’, ‘positive’, ‘O’, ‘O’, ‘O’]
```

## Benchmarking

SetFitABSA was benchmarked against the recent state-of-the-art work by [AWS AI Labs](https://arxiv.org/pdf/2210.06629.pdf) and [Salesforce AI Research](https://docs.google.com/document/d/1hhwzSr5CNunAU5wmtN8dcrL4AUskesRw/edit#bookmark=id.3znysh7) that finetune T5 and GPT2 using prompts. To get a more complete picture, we also compare our model to the Llama-2-chat model using in-context learning.
We use the popular Laptop14 and Restaurant14 ABSA [datasets](https://huggingface.co/datasets/alexcadillon/SemEval2014Task4) from the Semantic Evaluation Challenge 2014 ([SemEval14](https://docs.google.com/document/d/1hhwzSr5CNunAU5wmtN8dcrL4AUskesRw/edit#bookmark=id.30j0zll)). 
SetFitABSA is evaluated both on the intermediate task of aspect term extraction (SB1) and on the full ABSA task of aspect extraction along with their sentiment polarity predictions (SB1+SB2).

### Model size comparison

|       Model        | Size (params) |
|:------------------:|:-------------:|
|    Llama-2-chat    |      7B       |
|      T5-base       |     220M      |
|     GPT2-base      |     124M      |
|    GPT2-medium     |     355M      |
| **SetFit (MPNet)** |     110M      |

### Performance comparison

We see a clear advantage of SetFitABSA when the number of training instances is low, despite being 2x smaller than T5 and x3 smaller than GPT2-medium.  Even when compared to Llama 2, which is x64 larger, the performance is on par or better.

**SetFitABSA vs GPT2**

<p align="center">
    <img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/SetFitABSA_vs_GPT2.png" width=700>
</p>

**SetFitABSA vs T5**

<p align="center">
    <img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/SetFitABSA_vs_T5.png" width=700>
</p>

Note that the baseline works use different numbers of training sentences and different data splits. For fair comparison, we conducted separate comparisons with our model against each of them, ensuring consistency by using the splits employed by each respective work for accurate assessments.

**SetFitABSA vs Llama2**

<p align="center">
    <img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/SetFitABSA_vs_Llama2.png" width=700>
</p>

We notice that increasing the number of in-context training samples for Llama2 did not result in improved performance. This phenomenon has been shown for ChatGPT in [this blog post](https://www.analyticsvidhya.com/blog/2023/09/power-of-llms-zero-shot-and-few-shot-prompting/), however it should be further investigated.

## Training your own model

WIP

## References

* Maria Pontiki, Dimitris Galanis, John Pavlopoulos, Harris Papageorgiou, Ion Androutsopoulos, and Suresh Manandhar. 2014. SemEval-2014 task 4: Aspect based sentiment analysis. In Proceedings of the 8th International Workshop on Semantic Evaluation (SemEval 2014), pages 27–35.
* Siddharth Varia, Shuai Wang, Kishaloy Halder, Robert Vacareanu, Miguel Ballesteros, Yassine Benajiba, Neha Anna John, Rishita Anubhai, Smaranda Muresan, Dan Roth, 2023 "Instruction Tuning for Few-Shot Aspect-Based Sentiment Analysis". https://arxiv.org/abs/2210.06629
* Ehsan Hosseini-Asl, Wenhao Liu, Caiming Xiong, 2022. "A Generative Language Model for Few-shot Aspect-Based Sentiment Analysis". https://arxiv.org/abs/2204.05356
* Lewis Tunstall, Nils Reimers, Unso Eun Seo Jo, Luke Bates, Daniel Korat, Moshe Wasserblat, Oren Pereg, 2022. "Efficient Few-Shot Learning Without Prompts". https://arxiv.org/abs/2209.11055