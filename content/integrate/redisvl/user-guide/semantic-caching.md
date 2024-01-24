---
linkTitle: Semantic caching
title: Semantic caching
type: integration
description: Semantic caching with RedisVL
weight: 5
---

{{< note >}}
This document is a converted form of [this Jupyter notebook](https://github.com/RedisVentures/redisvl/blob/main/docs/user_guide/llmcache_03.ipynb).
{{< /note >}}

Before beginning, be sure of the following:

1. You have installed `redisvl` and have that environment activated.
1. You have a running Redis instance with the search and query capability.

## Semantic Caching for LLMs

RedisVL provides an ``SemanticCache`` interface to turn Redis into a semantic cache to store responses to previously asked questions. This reduces the number of requests and tokens sent to the Large Language Models (LLM) service, decreasing costs and enhancing application throughput (by reducing the time taken to generate responses).

This notebook will go over how to use Redis as a Semantic Cache for your applications

First, we will import [OpenAI](https://platform.openai.com) to use their API for responding to user prompts. We will also create a simple `ask_openai` helper method to assist.


```python
import os
import openai
import getpass

os.environ["TOKENIZERS_PARALLELISM"] = "False"

api_key = os.getenv("OPENAI_API_KEY") or getpass.getpass("Enter your OpenAI API key: ")

openai.api_key = api_key

def ask_openai(question: str) -> str:
    response = openai.Completion.create(
      engine="gpt-3.5-turbo-instruct",
      prompt=question,
      max_tokens=200
    )
    return response.choices[0].text.strip()
```


```python
# Test
print(ask_openai("What is the capital of France?"))
```

    The capital of France is Paris.


## Initializing ``SemanticCache``

``SemanticCache`` will automatically create an index within Redis upon initialization for the semantic cache content.


```python
from redisvl.llmcache import SemanticCache

llmcache = SemanticCache(
    name="llmcache",                     # underlying search index name
    prefix="llmcache",                   # redis key prefix for hash entries
    redis_url="redis://localhost:6379",  # redis connection url string
    distance_threshold=0.1               # semantic cache distance threshold
)
```


```python
# look at the index specification created for the semantic cache lookup
$ rvl index info -i llmcache
```

    
    
    Index Information:
    ╭──────────────┬────────────────┬──────────────┬─────────────────┬────────────╮
    │ Index Name   │ Storage Type   │ Prefixes     │ Index Options   │   Indexing │
    ├──────────────┼────────────────┼──────────────┼─────────────────┼────────────┤
    │ llmcache     │ HASH           │ ['llmcache'] │ []              │          0 │
    ╰──────────────┴────────────────┴──────────────┴─────────────────┴────────────╯
    Index Fields:
    ╭───────────────┬───────────────┬────────┬────────────────┬────────────────╮
    │ Name          │ Attribute     │ Type   │ Field Option   │   Option Value │
    ├───────────────┼───────────────┼────────┼────────────────┼────────────────┤
    │ prompt        │ prompt        │ TEXT   │ WEIGHT         │              1 │
    │ response      │ response      │ TEXT   │ WEIGHT         │              1 │
    │ prompt_vector │ prompt_vector │ VECTOR │                │                │
    ╰───────────────┴───────────────┴────────┴────────────────┴────────────────╯


## Basic Cache Usage


```python
question = "What is the capital of France?"
```


```python
# Check the semantic cache -- should be empty
if response := llmcache.check(prompt=question):
    print(response)
else:
    print("Empty cache")
```

    Empty cache


Our initial cache check should be empty since we have not yet stored anything in the cache. Below, store the `question`,
proper `response`, and any arbitrary `metadata` (as a python dictionary object) in the cache.


```python
# Cache the question, answer, and arbitrary metadata
llmcache.store(
    prompt=question,
    response="Paris",
    metadata={"city": "Paris", "country": "france"}
)
```


```python
# Check the cache again
if response := llmcache.check(prompt=question, return_fields=["prompt", "response", "metadata"]):
    print(response)
else:
    print("Empty cache")
```

    [{'id': 'llmcache:115049a298532be2f181edb03f766770c0db84c22aff39003fec340deaec7545', 'vector_distance': '8.34465026855e-07', 'prompt': 'What is the capital of France?', 'response': 'Paris', 'metadata': {'city': 'Paris', 'country': 'france'}}]



```python
# Check for a semantically similar result
question = "What actually is the capital of France?"
llmcache.check(prompt=question)[0]['response']
```




    'Paris'




```python
# Widen the semantic distance threshold
llmcache.set_threshold(0.3)
```


```python
# Really try to trick it by asking around the point
# But is able to slip just under our new threshold
question = "What is the capital city of the country in Europe that also has a city named Nice?"
llmcache.check(prompt=question)[0]['response']
```




    'Paris'




```python
# Invalidate the cache completely by clearing it out
llmcache.clear()

# should be empty now
llmcache.check(prompt=question)
```




    []



## Simple performance testing

Next, we will measure the speedup obtained by using ``SemanticCache``. We will use the ``time`` module to measure the time taken to generate responses with and without ``SemanticCache``.


```python
import time


def answer_question(question: str) -> str:
    """Helper function to answer a simple question using OpenAI with a wrapper
    check for the answer in the semantic cache first.

    Args:
        question (str): User input question.

    Returns:
        str: Response.
    """
    results = llmcache.check(prompt=question)
    if results:
        return results[0]["response"]
    else:
        answer = ask_openai(question)
        return answer
```


```python
start = time.time()
# asking a question -- openai response time
answer = answer_question("What is the capital of France?")
end = time.time()

print(f"Without caching, a call to openAI to answer this simple question took {end-start} seconds.")
```

    Without caching, a call to openAI to answer this simple question took 0.5017588138580322 seconds.



```python
llmcache.store(prompt="What is the capital of France?", response="Paris")
```


```python
cached_start = time.time()
cached_answer = answer_question("What is the capital of France?")
cached_end = time.time()
print(f"Time Taken with cache enabled: {cached_end - cached_start}")
print(f"Percentage of time saved: {round(((end - start) - (cached_end - cached_start)) / (end - start) * 100, 2)}%")
```

    Time Taken with cache enabled: 0.327639102935791
    Percentage of time saved: 34.7%



```python
# check the stats of the index
$ rvl stats -i llmcache
```

    
    Statistics:
    ╭─────────────────────────────┬─────────────╮
    │ Stat Key                    │ Value       │
    ├─────────────────────────────┼─────────────┤
    │ num_docs                    │ 1           │
    │ num_terms                   │ 7           │
    │ max_doc_id                  │ 2           │
    │ num_records                 │ 16          │
    │ percent_indexed             │ 1           │
    │ hash_indexing_failures      │ 0           │
    │ number_of_uses              │ 9           │
    │ bytes_per_record_avg        │ 5.25        │
    │ doc_table_size_mb           │ 0.000134468 │
    │ inverted_sz_mb              │ 8.01086e-05 │
    │ key_table_size_mb           │ 2.76566e-05 │
    │ offset_bits_per_record_avg  │ 8           │
    │ offset_vectors_sz_mb        │ 1.33514e-05 │
    │ offsets_per_term_avg        │ 0.875       │
    │ records_per_doc_avg         │ 16          │
    │ sortable_values_size_mb     │ 0           │
    │ total_indexing_time         │ 0.548       │
    │ total_inverted_index_blocks │ 7           │
    │ vector_index_sz_mb          │ 3.0161      │
    ╰─────────────────────────────┴─────────────╯



```python
# Clear the cache AND delete the underlying index
llmcache.delete()
```