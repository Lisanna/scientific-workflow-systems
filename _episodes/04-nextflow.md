---
title: "Running the example in Nextflow"
teaching: 0
exercises: 0
questions:
- "Key question (FIXME)"
objectives:
- "First learning objective. (FIXME)"
keypoints:
- "First key point. Brief Answer to questions. (FIXME)"
---

# Nextflow

Core features:
- Fast prototyping
- Reproducibility
- Portability
- Simple parallelism
- Continuous checkpoints

Differences to snakemake:
- Don't need to specify the names of output files/folder
- Groovy instead of Python
- Cannot do a dry run (would have to use small datasets)

> ## Functions in workflows management system
>
> Our example doesn't cover this, so you will not see Python/Groovy flavour, but consider at a 
> certain level of advancement this will be needed.
{: .callout}

- Resource usage plots
- Nextflow tower 
- nf-core (predefined pipelines)

### Assembling and running the example Workflow in Nextflow

### Step-by-Step 

#### Final result

~~~
#!/usr/bin/env nextflow

nextflow.enable.dsl = 2

params.url = "http://ftp.ensembl.org/pub/release-105/fasta/ciona_intestinalis/cdna/Ciona_intestinalis.KH.cdna.all.fa.gz"
params.sampleid = "Ciona_intestinalis"

workflow {
  input_ch = Channel.from(params.url)

  fetch_data(input_ch)
  length_filter(fetch_data.out)
  gene_calling(length_filter.out.splitFasta(by:2000, file:true))
  join_fastas(gene_calling.out.collect())
  DEAD_motif_search(join_fastas.out, params.sampleid)
}


process fetch_data {

  input:
  val url

  output:
  path "*.fa.gz"

  script:
  """
  wget ${url}
  """
}

process length_filter {

  container "https://depot.galaxyproject.org/singularity/seqkit%3A2.1.0--h9ee0642_0"

  input:
  path fasta

  output:
  path "length_filtered.fa"

  script:
  """
  seqkit seq --min-len 500 ${fasta} > length_filtered.fa
  """
}

process gene_calling {

  container "https://depot.galaxyproject.org/singularity/prodigal%3A2.6.3--h779adbc_3"

  input:
  path fasta

  output:
  path "proteins.faa"

  script:
  """
  prodigal -i ${fasta} -a proteins.faa -o proteins.gbk
  """
}

process join_fastas {

  input:
  path "fasta?.faa"

  output:
  path "joined.faa"

  script:
  """
  cat *.faa > joined.faa
  """
}

process DEAD_motif_search {

  container "https://depot.galaxyproject.org/singularity/seqkit%3A2.1.0--h9ee0642_0"
  publishDir "results/${sampleid}/", mode: 'copy'

  input:
  path fasta
  val sampleid

  output:
  path "dead_motif_containing_genes.faa"
  path "ids.txt"

  script:
  """
  seqkit grep -s -p DEAD ${fasta} > dead_motif_containing_genes.faa
  grep ">" dead_motif_containing_genes.faa | awk ' { print substr(\$1, 2)}' > ids.txt
  """
}
~~~
{:.source}

{% include links.md %}

