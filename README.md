# Nate Wilking BalkanID Hiring Task

This document serves as both documentation of my decision-making process and instructions for using my work to replicate my results for Task 2, Part 3.

## Feature Selection

The unarXiv data format encodes a lot of information, not all of which is relevant to citation network impact analysis:
- Hashes are effectively random for the purposes of this analysis and are therefore discarded.
- The full text and figures of each paper are likely useful to deeper impact analysis with more computing power at its disposal, but are not considered here given time and resource constraints.
- Outgoing citations are not considered in this analysis as doing so would encourage artificially inflating future papers with irrelevant bibliography entries.
- Bibliography entries referencing research papers not included in the unarXiv snapshot or referencing sources of other types (social media, news articles, etc.) are discarded because no information on what works those references cite in turn is available.

An example entry from the reduced dataset is provided below:
```
"2001.01128": {
    "title": "Locality-Sensitive Hashing for Efficient Web Application Security\n  Testing",
    "authors": [
      "Ilan Ben-Bassat",
      "Erez Rokah"
    ],
    "categories": [
      "cs.CR",
      "cs.LG",
      "stat.ML"
    ],
    "abstract": "  Web application security has become a major concern in recent years, as more\nand more content and services are available online. A useful method for\nidentifying security vulnerabilities is black-box testing, which relies on an\nautomated crawling of web applications. However, crawling Rich Internet\nApplications (RIAs) is a very challenging task. One of the key obstacles\ncrawlers face is the state similarity problem: how to determine if two\nclient-side states are equivalent. As current methods do not completely solve\nthis problem, a successful scan of many real-world RIAs is still not possible.\nWe present a novel approach to detect redundant content for security testing\npurposes. The algorithm applies locality-sensitive hashing using MinHash\nsketches in order to analyze the Document Object Model (DOM) structure of web\npages, and to efficiently estimate similarity between them. Our experimental\nresults show that this approach allows a successful scan of RIAs that cannot be\ncrawled otherwise.\n",
    "discipline": "Statistics",
    "arxiv_bib_ids": [
      "2001.01128"
    ]
  },
  ```

## External Data

After ingesting the raw JSONL, I performed rudimentary density estimation, which concluded that this dataset contains 795 papers making 10918 citations of 8947 unique works, 245 of which reference other papers in the dataset ([generate_graph.ipynb](/src/generate_graph.ipynb), cell 2). With only 2.2% of citations resolvable within the set, it was clear external data would be useful. Without it, the knowledge graph would be too sparse for detailed impact analysis.

Eventually, I recognized the provided JSONL as a subset of the unarXive project and retreived its most recent public snapshot from [its Zenodo page](https://zenodo.org/records/7752615). After parsing it into the same format as the original data, the hit rate for citations made by the papers of interest rose to 6.1% ([generate_graph.ipynb](/src/generate_graph.ipynb), cell 3). Stil far from perfect, but a significant improvement. The cascading citation information available in the unarXive snapshot is also useful to this analysis - more information on broad impact across disciplines is available in the expanded dataset.

## Graph Construction

An directed graph is a natural choice for this analysis. Weighting edges is a reasonable approach as well, but since I'm only using each edge in one impact computation, that of its source, I left the graph unweighted. I constructed the graph by iterating over each paper in the unarXive dataset, then interating over its bibliography entries. Any citations that were not self-referential or outside the dataset were added to the graph ([generate_graph.ipynb](/src/generate_graph.ipynb), cell 4).

One decision I made in constructing the graph was the direction of the edges. An edge from A to B can imply either that A cites B or vice versa. There is no difference between the possible computations under the two, but the latter case aligns chronological order with a topological sort of the graph and marks impactful papers as sources rather than sinks. Therefore, intuitively and stylistically, I decided that an edge from A to B should represent citation of A by B.

In the graph below, papers were published in alphabetical order, with A cited by B, C citing both A and B, etc. Notice that while triangles exist, no cycles are present.

![Example Graph](/media/graph.png)

Unfortunately, the resulting graph was cyclic, as visible in cell 5. This is a result of the peer review and revision process: sometimes, paper A would be published, cited and critiqued by paper B, then revised to incorporate B's feedback, citing it in the process. Removing cycles from a graph is usually an expensive and difficult process, but here I have access to a premade topological sort of the papers: publication time. By removing edges that represent citations made to papers published before the citing work, I can ensure that there are no cycles in the graph ([generate_graph.ipynb](/src/generate_graph.ipynb), cells 6-8). While this process does remove potentially useful information in a selective way that may introduce bias, less than 400 of the >33,000 edges in the graph were removed, so the ability to leverage DAG algorithms likely outweighs any such effects.

## Heuristic Development

The basic idea of this analysis is taken from the task instructions for Part 2:

> Importance of a research paper is measured not just by the number of citations that it receives but also by the quality of the citations. The quality of a citation is in turn determined by the number and quality of its own citations.

My implementation of this concept is a recursive descent algorithm similar to depth-first search. Beginning with each paper in the original dataset (unarXive papers are only used to poulate the networks of these papers - they are not themselves scored), I recursively evaluate every work that cites it until reaching a work that has no citations (outgoing edges). The score of a given node is calculated as the baseline impact of any paper (1.0) plus the impact of each work that cites it.

To differentiate citations from each other, I weighted each reference based on how different from each other the two papers were. Citations become less important when the two papers have authors in common, up to a full nullification of the impact score if the entire author list is identical. Citations across disciplines are also more heavily weighted than those within the same discipline. The full code of the method that handles this logic, `pairwise_impact()`, is available in cell 2 of [score_nodes.ipynb](/src/score_nodes.ipynb).

## Future Work

Even with the pairwise coefficient modification, this model makes a number of strong assumptions, including the following:
- All papers are impactful
- All papers not cited by another work are equally impactful
- Citations contribute to impact linearly (the tenth citation is weighted the same as the first)
- Receiving a citation cannot make a paper less impactful
- Citations from different categories within the same discipline are no more impactful that those in the same categories

Future analyses could benefit from accounting for deviations from these assumptions.

## Replication Instructions

Before beginning, ensure Python is updated to at least 3.11 and the following PyPi packages are installed:

- `arxiv`
- `networkx`
- `pandas`

### 1. Preparing the data

- Download the unarXive snapshot from [Zenodo](https://zenodo.org/records/7752615)
- Ensure `dataset.jsonl` and the unarXive `.tar.xz` are in the project folder
- Set the path variables in `/src/config.py` to the locations of the data and the desired output paths
- Run `/src/ingest_data.ipynb` (unarXive is large, so expect this to take up to 10 minutes)

### 2. Building the graph

Note: This section is time-intensive. While data and GitHub ordinarily don't mix well, since my graph.bin is quite small, I've committed it to this repository to allow this section to be skipped entirely.

- Verify that `reduced.json` and `unarxive.json` are present at the configured paths
- Run `/src/generate_graph` (this script makes a large volume of batched API queries and may take up to 45 minutes to execute)
- Confirm that `graph.bin` has been generated

### 3. Analyzing the graph

- Run `/src/scores_nodes.ipynb`
- The DataFrame `most_impactful` holds the five top-scoring papers
- The `impact_pct` column is impact score as a percentage of the highest score in the set

Every part of my implementation is deterministic, so provided you are using the March 27, 2023 unarXive release, your output should match mine. These are the five most impactful papers in the provided dataset according to my analysis:

|      id |   impact |   impact_pct | title                                                                    |
|--------:|---------:|-------------:|:-------------------------------------------------------------------------|
| 2004.04 | 254.595  |     100      | Compiling Spiking Neural Networks to Neuromorphic Hardware               |
| 2005.05 | 161.004  |      63.2392 | Exploiting Inter- and Intra-Memory Asymmetries for Data Mapping in Hybrid Tiered-Memories |
| 2005.05 | 129.269  |      50.7745 | Improving Phase Change Memory Performance with Data Content Aware Access |
| 2007.02 | 129.174  |      50.7371 | A Case for Lifetime Reliability-Aware Neuromorphic Computing             |
| 2009.13 |  89.6637 |      35.2182 | Reliability-Performance Trade-offs in Neuromorphic Computing             |