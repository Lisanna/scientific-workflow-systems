---
title: "Introduction to scientific workflow management systems"
teaching: 0
exercises: 0
questions:
- "Key question (FIXME)"
objectives:
- "First learning objective. (FIXME)"
keypoints:
- "First key point. Brief Answer to questions. (FIXME)"
---

### Workflow description

We will assemble a simple workflow easy to understand for anyone who attended basic biology classes. The goal of this workflow is to *predict Open Reading Frames (ORFs) from RNA sequences and filter these predicted ORFs to keep only those containing the specific amino acids (sub)sequence "VERA”*.

Although very simple, our implemetation will demonstrate:

- how to load remote data into Galaxy
- how to assemble multiple steps into a workflow
- how to use the *split-apply-combine* strategy

Here is a step-by-step overview:

- Step 1: download a fasta file containing RNA sequence from ensembl
- Step 2: split fasta file in eg 10 files (for parallel processing)
- Step 3: translate the sequence with an ORF finder (here I used a getorf wrapper on emboss)
- Step 4: filter for desired pattern
- Step 5: merge back translated fastq files
- Step 6: extract the original transcript IDs

{% include links.md %}

