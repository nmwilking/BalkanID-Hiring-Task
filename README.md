# Nate Wilking BalkanID Hiring Task

## Feature Selection

- Hashes are essentially random and therefore uncorrelated to importance
- Full body text + figures are likely useful to deeper analysis but outside the scope of a week-long investigation
- References not included in the dataset can't be meaningfully weighted (ignore vs count vs apply mean will be covered in the heuristic discussion)


## Citation weighting

Concepts to consider for citaiton weighting of B's citation of A:
- How many citations B received
- How dissimilar B's abstract was to A (value of replication studies?)
- Different disciplines are indicative of greater impact
- Common author penalty

## UnarXiv

External data necessary after density estimation
https://zenodo.org/records/7752615

reqs: nx, arxiv