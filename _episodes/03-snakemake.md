---
title: "Running the example in Snakemake"
teaching: 0
exercises: 0
questions:
- "Key question (FIXME)"
objectives:
- "First learning objective. (FIXME)"
keypoints:
- "First key point. Brief Answer to questions. (FIXME)"
---

# Snakemake

GNU make + Python = Snakemake

GNU make: software to manage projects with several intermediate steps.
"You only build what needs to be built"
Instructions are written in a makefile, which looks like this:

~~~
all proteins.faa
proteins.faa: transcripts.fna
  getorf -sequence transcripts.fna -outseq proteins.faa
~~~
{: .source}

Why Snakemake instead of make? 
- Python-dialect
- Native support for reproducibility principles 
  - different environment
  - conda/mamba singularity
  - docker support
- HPC support
- Simple control-flow mechanisms
- Python API (bind Snakemake workflows into a Python application)


## Assembling and running the example Workflow in Snakemake

### Step-by-Step 

#### Download transcripts

~~~
TRANSCRIPTS_URL = "http://ftp.ensembl.org/pub/release-105/fasta/naja_naja/cdna/Naja_naja.Nana_v5.cdna.all.fa.gz"

rule all:
	input:
		"output/transcripts.fa.gz",


rule fetch_transcriptome:
	output:
		"output/transcripts.fa.gz"
  params:
    url = TRANSCRIPTS_URL
	shell:
		"""
		mkdir -p output/
		wget {TRANSCRIPTS_URL} -O output/transcripts.fa.gz
		"""
~~~
{:.language-python}

#### Filter by length 

~~~
ENVIRONMENTS = "../envs"
~~~
{: .source}

Point at environments.

~~~
rule all:
	input:
		"output/transcripts.fa.gz",
		"output/transcripts.length_filtered.fa.gz",
~~~
{:.language-python}

Add filtered to rule all.

~~~
rule filter_by_length:
	input:
		transcripts = rules.fetch_transcriptome.output.transcripts
	output:
		filtered = rules.fetch_transcriptome.output.transcripts.replace(".fa.gz", ".length_filtered.fa.gz")
	params:
		minlen = 300
	conda:
		f"{ENVIRONMENTS}/bbmap.yml"
	shell:
		"""
		bbduk.sh in={input.transcripts} out={output.filtered} minlen={params.minlen}
		"""
~~~
{:.language-python}

#### Splitting transcriptome ("scatter")

Add the rule to split the transcriptome. 

{% raw %}
~~~
rule split_transcriptome:
	input:
		rules.filter_by_length.output.filtered
	output:
		txome_chunks = expand(os.path.join("output", "txome_chunks", "chunk-{chunk}.fa"), chunk=range(10))
	params:
		chunksize = 2935
	shell:
		"""
		mkdir -p output/txome_chunks/
		gzip -dc {input[0]} | awk 'BEGIN {{n=0;m=0;}} /^>/ {{ if (n%{params.chunksize}==0) {{f=sprintf(\"output/txome_chunks/chunk-%d.fa\",m); m++;}}; n++; }} {{ print >> f }}'
		"""
~~~
{:.language-python}
{% endraw %}

> ## Check the steps
>
> This is currently the structure of your Snakemake file.
>
> ~~~
> TRANSCRIPTS_URL = ...
> ENVIRONMENTS = ...
> 
> rule all:
>   input:
>     ...
> 
> rule fetch_transcriptome:
>   ...
> 
> rule filter_by_length:
>   ...
> 
> rule split_transcriptome:
>   ...
> ~~~
> {: .source}
{: .callout}

#### Find and translate ORFs

~~~
rule find_and_translate_orfs:
	input:
		chunk = os.path.join("output", "txome_chunks", "chunk-{chunk}.fa"),
	output:
		orfs = os.path.join("output", "orf_chunks", "chunk-{chunk}.orfs.fa"),
	params:
		minlen = 300
	conda:
		f"{ENVIRONMENTS}/emboss.yml"
	shell:
		"""
		mkdir -p output/orf_chunks/
		getorf -sequence {input.chunk} -outseq {output.orfs} -minsize {params.minlen}
		"""
~~~
{:.language-python}

#### Extract toxins

~~~
rule extract_toxins:
	input:
		toxin_proteins_blast = rules.combine_chunks_and_filter.output.toxin_proteins_blast,
		proteins = rules.combine_proteins.output.all_proteins
	output:
		toxin_proteins = os.path.join("output", "toxins.faa")
	conda:
		f"{ENVIRONMENTS}/seqtk.yml"
	shell:
		"""
		seqtk subseq {input.proteins} <(cut -f 1 {input.toxin_proteins_blast} | sort -u) > {output.toxin_proteins}
		"""
~~~
{:.language-python}

#### Final result

{% raw %}
~~~
# TRANSCRIPTS_URL = "http://ftp.ensembl.org/pub/release-105/fasta/naja_naja/cdna/Naja_naja.Nana_v5.cdna.all.fa.gz"

# ENVIRONMENTS = "../envs"

# report: "workflow_report.rst"

rule all:
	input:
		transcripts = "output/transcripts.fa.gz",
		txome_chunks = expand(os.path.join("output", "txome_chunks", "chunk-{chunk}.fa"), chunk=range(10)),
		orf_chunks = expand(os.path.join("output", "orf_chunks", "chunk-{chunk}.orfs.fa"), chunk=range(10)),
		toxin_chunks = expand(os.path.join("output", "toxin_chunks", "chunk-{chunk}.toxins.tsv"), chunk=range(10)),
		all_proteins = os.path.join("output", "all_proteins.faa"),
		blast_output = os.path.join("output", "toxin_blast.filtered.tsv"),
		toxins = os.path.join("output", "toxins.faa")

	shell:
		"""
		rm -rvf {input.transcripts} $(dirname {input.txome_chunks}) $(dirname {input.orf_chunks}) $(dirname {input.toxin_chunks}) {input.all_proteins} {input.blast_output}
		"""


rule fetch_transcriptome:
	output:
		transcripts = "output/transcripts.fa.gz"

	params:
		url = config["TRANSCRIPTS_URL"]

	shell:
		"""
		mkdir -p output/
		wget {params.url} -O {output.transcripts}
		"""


rule filter_by_length:
	input:
		transcripts = rules.fetch_transcriptome.output.transcripts
	output:
		filtered = rules.fetch_transcriptome.output.transcripts.replace(".fa.gz", ".length_filtered.fa.gz")
	params:
		minlen = config["transcript_criteria"]["minimum_transcript_length"]
	conda:
		f"{config['ENVIRONMENTS']}/bbmap.yml"
	shell:
		"""
		bbduk.sh in={input.transcripts} out={output.filtered} minlen={params.minlen}
		"""


rule split_transcriptome:
	input:
		rules.fetch_transcriptome.output.transcripts
	output:
		txome_chunks = expand(os.path.join("output", "txome_chunks", "chunk-{chunk}.fa"), chunk=range(10))
	params:
		chunksize = config["transcript_criteria"]["chunksize"]
	shell:
		"""
		mkdir -p output/txome_chunks/
		gzip -dc {input[0]} | awk 'BEGIN {{n=0;m=0;}} /^>/ {{ if (n%{params.chunksize}==0) {{f=sprintf(\"output/txome_chunks/chunk-%d.fa\",m); m++;}}; n++; }} {{ print >> f }}'
		"""


rule find_and_translate_orfs:
	input:
		chunk = os.path.join("output", "txome_chunks", "chunk-{chunk}.fa"),
	output:
		orfs = os.path.join("output", "orf_chunks", "chunk-{chunk}.orfs.fa"),
	params:
		minlen = 1000
	conda:
		f"{config['ENVIRONMENTS']}/emboss.yml"
	shell:
		"""
		mkdir -p output/orf_chunks/
		getorf -sequence {input.chunk} -outseq {output.orfs} -minsize {params.minlen}
		"""


rule find_toxins:
	input:
		proteins = os.path.join("output", "orf_chunks", "chunk-{chunk}.orfs.fa"),
		db = config["VENOM_DATABASE"]
	output:
		toxin_hits = os.path.join("output", "toxin_chunks", "chunk-{chunk}.toxins.tsv")
	conda:
		f"{config['ENVIRONMENTS']}/blast.yml"
	threads:
		2
	shell:
		"""
		mkdir -p output/toxin_chunks/
		blastp -db {input.db} -query {input.proteins} -num_threads {threads} -out {output.toxin_hits} -outfmt '6 qseqid sseqid pident qstart qend sstart send qlen slen length nident mismatch positive gapopen gaps evalue bitscore'
		"""


rule combine_chunks_and_filter:
	input:
		expand(os.path.join("output", "toxin_chunks", "chunk-{chunk}.toxins.tsv"), chunk=range(10))
	output:
		toxin_proteins_blast = os.path.join("output", "toxin_blast.filtered.tsv")
	params:
		eval_threshold = config['transcript_criteria']['eval_threshold']
	shell:
		"""
		awk '$16 < {params.eval_threshold}' {input} > {output.toxin_proteins_blast}
		"""

rule combine_proteins:
	input:
		expand(os.path.join("output", "orf_chunks", "chunk-{chunk}.orfs.fa"), chunk=range(10))
	output:
		all_proteins = os.path.join("output", "all_proteins.faa")
	shell:
		"""
		cat {input} > {output.all_proteins}
		"""


rule extract_toxins:
	input:
		toxin_proteins_blast = rules.combine_chunks_and_filter.output.toxin_proteins_blast,
		proteins = rules.combine_proteins.output.all_proteins
	output:
		toxin_proteins = os.path.join("output", "toxins.faa")
	conda:
		f"{config['ENVIRONMENTS']}/seqtk.yml"
	shell:
		"""
		seqtk subseq {input.proteins} <(cut -f 1 {input.toxin_proteins_blast} | sort -u) > {output.toxin_proteins}
		"""
	

~~~
{:.language-python}
{% endraw %}







{% include links.md %}

