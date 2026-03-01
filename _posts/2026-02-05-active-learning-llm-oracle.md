---
title: 'Replacing human data labeling with LLMs in active learning'
date: 05/02/2026
permalink: /posts/active-learning-llm-oracle/
tags:
  - Active Learning
  - LLM
  - everyrow
---

*Evaluating a model trained with automated data annotation.*

**tl;dr:** Human data labeling is slow and expensive. Using everyrow's agent_map, we replaced human annotators with an LLM oracle in an active learning loop and achieved identical classifier performance — 200 labels in under 5 minutes for $0.26. Over 10 controlled repeats on DBpedia-14, LLM labels matched human accuracy within 0.1%.

[Active learning](https://en.wikipedia.org/wiki/Active_learning_(machine_learning)) reduces labeling costs by letting the model choose which examples to label next, focusing on the ones it is most uncertain about. But you still need an oracle to provide those labels, traditionally a human annotator.

We replaced the human annotator with an LLM oracle, then ran a controlled experiment on a 14 class text classification task to measure whether automated data labeling produces labels good enough to train on.

We used a TF-IDF + LightGBM classifier with entropy based uncertainty sampling. Each iteration selects the 20 most uncertain examples, sends them to the LLM for annotation, and retrains. 10 iterations, 200 labels total.

We ran 10 independent repeats with different seeds, each time running both a ground truth oracle (human labels) and the LLM oracle with the same seed, a direct, controlled comparison.

```python
class DBpediaClassification(BaseModel):
    category: Literal[
        "Company", "Educational Institution", "Artist",
        "Athlete", "Office Holder", "Mean Of Transportation",
        "Building", "Natural Place", "Village",
        "Animal", "Plant", "Album", "Film", "Written Work",
    ] = Field(description="The DBpedia ontology category")

async def query_llm_oracle(texts_df: pd.DataFrame) -> list[int]:
    async with create_session(name="Active Learning Oracle") as session:
        result = await agent_map(
            session=session,
            task="Classify this text into exactly one DBpedia ontology category.",
            input=texts_df[["text"]],
            response_model=DBpediaClassification,
            effort_level=EffortLevel.LOW,
        )
        return [CATEGORY_TO_ID.get(result.data["category"].iloc[i], -1)
                for i in range(len(texts_df))]
```

## Results

Final test accuracies averaged over 10 repeats:

| Data Labeling Method | Final Accuracy (mean ± std) |
|---|---|
| Human annotation (ground truth) | 80.6% ± 1.0% |
| LLM annotation (everyrow) | 80.7% ± 0.8% |

The LLM oracle is within noise of the ground truth baseline, automated data labeling produces classifiers just as good as human labeled data.

The LLM agreed with ground truth labels 96.1% ± 1.6% of the time. Roughly 1 in 25 labels disagrees with the human annotation, but that does not hurt the downstream classifier.

| Metric | Value |
|---|---|
| Cost per run (200 labels) | $0.26 |
| Cost per labeled item | $0.0013 |
| Total (10 repeats) | $2.58 |

200 labels in under 5 minutes for $0.26, fully automated. Compare that to Mechanical Turk, where even simple labeling tasks cost $0.05 to $0.10 per item.

## Limitations

We tested on one dataset with well separated categories. More ambiguous labeling tasks may see a gap between human and LLM annotation quality. We used a simple classifier (TF-IDF + LightGBM); neural models that overfit individual examples may be less noise tolerant.

The low cost in this experiment comes from using `EffortLevel.LOW`, which selects a small, fast model on the pareto frontier and doesn't use web research to improve the label quality. For simple classification tasks with well separated categories, this is sufficient.

For more ambiguous labeling tasks, you can use `EffortLevel.MEDIUM` or `EffortLevel.HIGH` to get higher quality labels from smarter models using the web. The cost scales accordingly, but even at higher effort levels, LLM labeling remains cheaper and faster than human annotation.

## Reproduce This Experiment

The full pipeline is available as a [companion notebook on Kaggle](https://www.kaggle.com/code/rafaelpoyiadzi/active-learning-with-an-llm-oracle). The experiment uses the [DBpedia-14 dataset](https://huggingface.co/datasets/fancyzhx/dbpedia_14), a 14 class text classification benchmark. Install the SDK with `pip install everyrow` and get an API key at [everyrow.io/api-key](https://everyrow.io/api-key).

## Resources

- [everyrow SDK](https://github.com/futuresearch/everyrow-sdk) for running LLM operations over dataframes
- [everyrow.io](https://everyrow.io) for docs and API keys
