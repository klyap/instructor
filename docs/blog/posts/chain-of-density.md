---
draft: False
date: 2023-11-05
tags:
  - pydantic
  - validation
  - chain of density
authors:
  - jxnl
  - ivanleomk
---

# Implementing Chain Of Density

By the end of this article, you'll have a fine-tuned GPT 3.5 model which can take in source text and output a summary which mimics a chain-of-density summarization using `Instructor`'s new jobs cli feature. As usual, all of the code is avaliable for reference under our `examples/chain-of-density` folder. 

## What is a Chain Of Density?

It is often difficult to use large language models to summarize large corpuses of text. Most attempts try to use a brittle prompt to force constraints on language models or use multiple passes. However, without a consistent framework to guide the summarization, it really is no better than rolling some dice and hoping you get the result you want. 

In this article, we'll introduce a new approach to dealing with summarization called `chain of density`, first introduced by Salesforce's AI Research wing in their paper - [From Sparse to Dense: GPT-4 Summarization with Chain of Density Prompting](https://arxiv.org/abs/2309.04269). This method is useful and consistently beats similar summaries written by human annotators.

First, a large language model is provided with an article and asked to generate an initial summary using overly verbose language and fillers. Next, it then repeatedly rewrites this summary for a set number of iterations. During each iteration, the model identifies new entities from the article that are missing from the summary and then rewrites the existing summary to include them.

During the entire process, the model is also tasked with keeping the length of the summaries fairly constant. As a result, as new entities are added in with each iteration, we get more entities present within our summary. This results in a summary which is more entity dense, thus the name `Chain Of Density`. 

## Implementation

### Paper Prompt

Let's start with the original prompt that the paper uses.

```
Article: {{ARTICLE}}

You will generate increasingly concise, entity-dense summaries of the
above Article.

Repeat the following 2 steps 5 times.

Step 1. Identify 1-3 informative Entities (";" delimited) from the
Article which are missing from the previously generated summary.
Step 2. Write a new, denser summary of identical length which covers
every entity and detail from the previous summary plus the Missing
Entities.

A Missing Entity is:
- Relevant: to the main story.
- Specific: descriptive yet concise (5 words or fewer).
- Novel; not in the previous summary.
- Faithful: present in the Article.
- Anywhere: located anywhere in the Article.

Guidelines:
- The first summary should be long (4-5 sentences, -80 words) yet
highly non-specific, containing little information beyond the
entities marked as missing. Use overly verbose language and fillers
(e.g., "this article discusses") to reach -80 words.
- Make every word count: re-write the previous summary to improve
flow and make space for additional entities.
- Make space with fusion, compression, and removal of uninformative
phrases like "the article discusses"
- The summaries should become highly dense and concise yet
self-contained, e.g., easily understood without the Article.
- Missing entities can appear anywhere in the new summary.
- Never drop entities from the previous summary. If space cannot be
made, add fewer new entities.

Remember, use the exact same number of words for each summary.

Answer in JSON. The JSON should be a list (length 5) of dictionaries
whose keys are "Missing_Entities" and "Denser_Summary"
```

We can implement this using Instructor by breaking down the entire process into smaller api calls. This allows us to introduce validation at each step to ensure that we're getting the results that we want.


<figure markdown>
  ![RAG](img/chain-of-density.png)
  <figcaption>Improved process with Instructor</figcaption>
</figure>

### Data Modelling

#### Initial Summary

Let's start by walking through some of the data models that we'll be using as the `response_model` for our open ai function calls

Firstly, we'll need a data model for the initial summary that we will be generating. We'll take the description of this class straight from the original prompt.

```py
class InitialSummary(BaseModel):
    """
    This is an initial summary which should be long ( 4-5 sentences, ~80 words) yet highly non-specific, containing little information beyond the entities marked as missing. Use overly verbose languages and fillers (Eg. This article discusses) to reach ~80 words.
    """

    summary: str = Field(
        ...,
        description="This is a summary of the article provided which is overly verbose and uses fillers. It should be roughly 80 words in length",
    )
```

#### Rewritten Summary


We'll also need one additional class to help model the rewritten schema

```py
class RewrittenSummary(BaseModel):
    """
    This is a new, denser summary of identical length which covers every entity and detail from the previous summary plus the Missing Entities.

    Guidelines
    - Make every word count : Rewrite the previous summary to improve flow and make space for additional entities
    - Never drop entities from the previous summary. If space cannot be made, add fewer new entities.
    - The new summary should be highly dense and concise yet self-contained, eg., easily understood without the Article.
    - Make space with fusion, compression, and removal of uninformative phrases like "the article discusses"
    - Missing entities can appear anywhere in the new summary

    An Entity is a real-world object that's assigned a name - for example, a person, country a product or a book title.
    """

    summary: str = Field(
        ...,
        description="This is a new, denser summary of identical length which covers every entity and detail from the previous summary plus the Missing Entities. It should have the same length ( ~ 80 words ) as the previous summary and should be easily understood without the Article",
    )
    absent: List[str] = Field(
        ...,
        default_factory=list,
        description="this is a list of Entities found absent from the new summary that were present in the previous summary",
    )
    missing: List[str] = Field(
        default_factory=list,
        description="This is a list of 1-3 informative Entities from the Article that are missing from the new summary which should be included in the next generated summary.",
    )
```

!!! Using Validators in Instructor

    For a more in-depth walkthrough on how to use `Pydantic` validators with the `Instructor` 
    library, we recommend checking out our previous article on LLM 
    validation - [Good LLM Validation is just Good Validation](/instructor/blog/2023/10/23/good-llm-validation-is-just-good-validation/)

Ideally, we'd like for `Missing` to have a length between 1 and 3, `Absent` to be an empty list and for our summary to be around 80 tokens. With `Instructor`, we can implement this logic using native `Pydantic` validators that are simply declared as part of the class itself.

```py hl_lines="3"
@field_validator("summary")
def min_length(cls, v: str):
    tokens = nltk.word_tokenize(v) #(1)!
    num_tokens = len(tokens)
    if num_tokens < 75:
        raise ValueError(
            "The current summary is too short. Please make sure that you generate a new summary that is around 80 words long."
        )
    return v

@field_validator("missing")
def has_missing_entities(cls, missing_entities: List[str]):
    if len(missing_entities) == 0:
        raise ValueError(
            "You must identify 1-3 informative Entities from the Article which are missing from the previously generated summary to be used in a new summary"
        )
    return missing_entities

@field_validator("absent")
def has_no_absent_entities(cls, absent_entities: List[str]):
    absent_entity_string = ",".join(absent_entities)
    if len(absent_entities) > 0:
        print(f"Detected absent entities of {absent_entity_string}")
        raise ValueError(
            f"Do not omit the following Entities {absent_entity_string} from the new summary"
        )
    return absent_entities
```

1.  Similar to the original paper, we utilize the `NLTK` word tokenizer to count the number of tokens within our generated sentences. 
    We aim for at least 75 tokens in our generated summary so that we don't lose information.

### Putting it all Together

Now that we have our models and the rough flow figured out, let's implement a function to summarize a piece of text using `Chain Of Density` summarization. 

```py hl_lines="4-19 33-63"
def summarize_article(article: str, summary_steps: int = 3):
    summary_chain = []
    # We first generate an initial summary
    summary: InitialSummary = openai.ChatCompletion.create(  # (1)!
        model="gpt-4-0613",
        response_model=InitialSummary,
        messages=[
            {
                "role": "system",
                "content": "Write a summary about the article that is long (4-5 sentences) yet highly non-specific. Use overly, verbose language and fillers(eg.,'this article discusses') to reach ~80 words",
            },
            {"role": "user", "content": f"Here is the Article: {article}"},
            {
                "role": "user",
                "content": "The generated summary should be about 80 words.",
            },
        ],
        max_retries=2,
    )   
    prev_summary = None
    summary_chain.append(summary.summary)
    for i in range(summary_steps):
        missing_entity_message = (
            []
            if prev_summary is None
            else [
                {
                    "role": "user",
                    "content": f"Please include these Missing Entities: {','.join(prev_summary.missing)}",
                },
            ]
        )
        new_summary: RewrittenSummary = openai.ChatCompletion.create( # (2)!
            model="gpt-4-0613",
            messages=[
                {
                    "role": "system",
                    "content": """
                You are going to generate an increasingly concise,entity-dense summary of the following article.

                Perform the following two tasks
                - Identify 1-3 informative entities from the following article which is missing from the previous summary
                - Write a new denser summary of identical length which covers every entity and detail from the previous summary plus the Missing Entities 

                Guidelines
                - Make every word count: re-write the previous summary to improve flow and make space for additional entities
                - Make space with fusion, compression, and removal of uninformative phrases like "the article discusses".
                - The summaries should become highly dense and concise yet self-contained, e.g., easily understood without the Article.
                - Missing entities can appear anywhere in the new summary
                - Never drop entities from the previous summary. If space cannot be made, add fewer new entities.
                """,
                },
                {"role": "user", "content": f"Here is the Article: {article}"},
                {
                    "role": "user",
                    "content": f"Here is the previous summary: {summary_chain[-1]}",
                },
                *missing_entity_message,
            ],
            max_retries=3,
            max_tokens=1000,
            response_model=RewrittenSummary,
        )
        summary_chain.append(new_summary.summary)
        prev_summary = new_summary

    return summary_chain
```

1.  We first generate an initial summary. Note here that we explictly ask for a summary that has 
    80 words and is lengthy with overly verbose fillers in the system prompt

2.  We slightly modify the original system prompt used in the original paper to perform a rewrite of the summary. 
    Using `Instructor`, we also get validation of the generated output with our `field_validator`s that we defined above
    
We can run this function on a sample corpus of text - we provide a script to download the `griffin/chain-of-density` dataset which the paper utilized in the `examples/chain-of-density` folder. 

This yields a result as seen below which has almost 3 times the number of entities with roughly the same length. We can also see that stylistically, the summary is a lot more natural.

**First Summary** : This article discusses the highly-anticipated boxing match between Manny Pacquiao and Floyd Mayweather. The article revolves around Manny Pacquiao's statements about his upcoming fight and his preparations for the same. A portion of the article provides details about the financial stipulations of the match and its significance in the sporting arena. Quotes from Pacquiao illustrating his determination and his battle strategy are highlighted. The tone of the article is largely centered around creating a build-up to the upcoming mega event.

**Final Summary** : Manny Pacquiao, the Filipino boxer, anticipates the forthcoming May 2 showdown at the MGM Grand as the fight of his life, against the undefeated American Floyd Mayweather, in a $300m bout. Despite being seen as the underdog in this high-stakes Las Vegas match, Pacquiao is confident, promising a warrior's spirit and assuring the fans who have been awaiting this encounter for a decade, that it will indeed be the biggest sporting spectacle in history worthy of their anticipation

## Fine-tuning our Model

We can now finetune GPT3.5 to perform this task using these summaries generated using GPT4. Instructor provides an easy way to do so with the `Instructions` object. 

Simply create a new `Instruction` instance, use our provided annotator and you'll get a nicely formatted `.jsonl` file to be used for fine-tuning. In our example below, we read in 20 news articles from the `griffin/chain-of-density` dataset to generate our fine-tuning data.

```py hl_lines="2 9 13-20 25 28 33"
from typing import List
from chain_of_density import summarize_article #(1)!
import csv
import logging
import instructor
from itertools import islice
from pydantic import BaseModel

instructor.patch() #(2)!

logging.basicConfig(level=logging.INFO) 

instructions = instructor.Instructions( #(3)!
    name="Chain Of Density",
    finetune_format="messages",
    # log handler is used to save the data to a file
    # you can imagine saving it to a database or other storage
    # based on your needs!
    log_handlers=[logging.FileHandler("generated.jsonl")],
)

class GeneratedSummary(BaseModel):
    summary: str

@instructions.distil #(4)!
def distil_summarization(text: str) -> GeneratedSummary:
    summary_chain: List[str] = summarize_article(text)
    return GeneratedSummary(summary=summary_chain[-1]) #(5)!

with open("output.csv", "r") as file:
    reader = csv.reader(file)
    # Row 0 of our CSV file is the header row so we start from row 1
    for article, summary in islice(reader, 1, 20): #(6)!
        summaries.append(distil_summarization(article))
```

1.  In this example, we're using the summarize_article that we defined up above. We saved it in a local file called `chain_of_density.py`,
    hence the import

2.  Don't forget to use `instructor.patch()` so that you get the automatic parsing and validation of OpenAI responses.

3.  We instantiate a `Instruction` object which will help us handle the conversion of our function calls into a valid `.jsonl` file. We also define
    the name of the `.jsonl` file in the `log_handlers` parameter

4.  We add in an `instructions.distil` annotation so that we automatically capture the input and output of the function we'd like to 
    fine-tune our model to output

5.  We return a `Pydantic` object which matches the annotation that we use on our function. Note that we must specify a `Pydantic` object to
    be returned when using the `instructions.distil` annotation

6.  The `islice` operator is used to take a slice of an iterator - it allows us to take a subset of the original iterator from a specified
    start and end index.

!!! Rate Limiting

    We recommend running this script on a small subset of the dataset first to test you've got everything configured nicely.
    Don't forget to add in rate limiting error handling with `tenacity` and set the `OPENAI_API_KEY` shell environment variable
    before running any subsequent commands


Once we run this script, we'll have a new file called `generated.jsonl` in our local repository. Now all that's left is to run the command

```
instructor jobs create-from-file generated.jsonl
```

Once the job is complete, all we need to do is to then change the annotation in the function call to `distil_summarization` in our original file above to start using our new model.

```py
@instructions.distil(model='gpt-3.5-turbo:finetuned-123', mode="dispatch") #(1)!
def distil_summarization(text: str) -> GeneratedSummary:
    summary_chain: List[str] = summarize_article(text)
    return GeneratedSummary(summary=summary_chain[-1]) 
```

1. Don't forget to replace this with your new model id. OpenAI identifies fine tuned models with an id of 
    ft:gpt-3.5-turbo-0613:personal::<id> under their  Fine-tuning tab on their dashboard

With that, you've now got your own fine-tuned model ready to go and serve data in production. We've seen how Instructor can make your life easier, from fine-tuning to distillation.

If you enjoy the content or want to try out instructor please check out the github and give us a star!