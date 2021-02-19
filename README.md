There is a shell script detailing an example imbalance analysis of islet ChIP-seq data at `/home/data/chip-imbalance-example/example_HI_87.sh`. It relies on a few tools that other groups have made:

```sh
pyQuASAR-genotype raw-reads/*_HI_87.fastq.gz \
  --write-bam \
  --bam-dir alignment \
  --vcf-chr genotype/HI_87 \
  --sample HI_87 \
  --quiet \
  --processes 8 \
  --tmp-dir /nfs/lab/tmp/
for vcf in genotype/HI_87.chr*.vcf; do bgzip $vcf; done
wasp_map-map remap \
  --sample MAFB_HI_87 alignment/ERS353636_MAFB_HI_87.filt.bam \
  --sample NKX2_2_HI_87 alignment/ERS353636_NKX2_2_HI_87.filt.bam \
  --vcf-format genotype/HI_87.chr{}.vcf.gz \
  --vcf-sample HI_87 \
  --r2 0 \
  --processes 16 \
  --tmp-dir /nfs/lab/tmp
for factor in MAFB NKX2_2
do pileups-count --json remap/pileup/${factor}_HI_87.pileup \
  | call-imbalance --processes 16 --tmp-dir /nfs/lab/tmp \
  > ${factor}_HI_87_imbalance.txt
done
```

- [QuASAR](https://github.com/piquelab/QuASAR) for inferring genotypes of samples from e.g. ChIP/ATAC-seq data
- [WASP](https://github.com/bmvdgeijn/WASP) for correcting reference mapping bias
- [NPBin](https://pubmed.ncbi.nlm.nih.gov/29126153/) for estimating beta-binomial parameters to perform beta-binomial tests

I wrote a few Python tools to make installing and using those components a bit easier. I called them `pyQuASAR-genotype`, `wasp_map-map`, and `call-imbalance`. In the shell script above you can also see the command `pileup-to-counts`, which takes care of a small intermediate step.

Each of these tools depends on a bit of infrastructure that's set up on gatsby, so I'll go through them one at a time.

## Environment variables

Running imbalance analyses requires some data on SNPs and a reference genome. The above tools will need to know where these things are, and the easiest way to provide them is with environment variables. You can set up your environment on on gatsby or sherlock by running this:

```sh
echo '
# pydbsnp
export PYDBSNP_VCF_GRCH37=/nfs/lab/pydbsnp/GCF_000001405.25.bgz
export PYDBSNP_VCF_GRCH38=/nfs/lab/pydbsnp/GCF_000001405.38.bgz
export PYDBSNP_RSID_GRCH37=/nfs/lab/pydbsnp/GCF_000001405.25.rsid.bgz
export PYDBSNP_RSID_GRCH38=/nfs/lab/pydbsnp/GCF_000001405.38.rsid.bgz
# pyhg19
export PYHG19_PATH=/nfs/lab/aaylward/ref/male.hg19.fa
export PYHG19_MASKED=/nfs/lab/KG/ref/male.masked.hg19.fa
export PYHG19_BOWTIE2_INDEX=/nfs/lab/hg19/male.hg19
# py1kgp
export PY1KGP_DIR=/nfs/lab/1KGP' \
>> ~/.bash_profile
source ~/.bash_profile
```

After this, the environment variables will automatically be set when you log in, so you won't have to worry about them anymore.


## wasp_map-map

`wasp_map-map` is a command that automates the intermediate steps of remapping reads with WASP to correct reference mapping bias. It's part of a Python package called `wasp_map` seen [here on github](https://github.com/anthony-aylward/wasp_map), which also automates most of installation of WASP for you.

A good way to install WASP for yourself on gatsby is:

```sh
echo '
# wasp_map
export WASP_MAP_ANACONDA_DIR=~/anaconda3
export WASP_MAP_DIR=~/WASP
export LD_LIBRARY_PATH=${WASP_MAP_ANACONDA_DIR}/lib:$LD_LIBRARY_PATH' \
>> ~/.bash_profile
source ~/.bash_profile
pip3 install --user wasp_map
wasp_map-download
```

The command `wasp_map-download` will download a copy of Anaconda3 and install WASP. The installation process will include a couple of prompts to answer. First this one:

```
installing Anaconda3. When prompted, specify the following install location:
~~ path ending in /anaconda3 ~~
press ENTER to continue >>>
```

You should copy down this anaconda path because you'll need to enter it at another prompt. Then press ENTER for the next prompt:

```
In order to continue the installation process, please review the license
agreement.
Please, press ENTER to continue
>>> 
```

Here you can press ENTER and scroll through the license agreement, until the next prompt:

```
Do you accept the license terms? [yes|no]
[no] >>> 
```

Enter `yes` to accept it. Next, you'll see this prompt:

```
Anaconda3 will now be installed into this location:
/home/aaylward/anaconda3

  - Press ENTER to confirm the location
  - Press CTRL-C to abort the installation
  - Or specify a different location below

[/home/aaylward/anaconda3] >>> 
```

This is where you should enter the path you copied earlier (or you can just scroll back up to the previous prompt that displayed it.) This will ensure this copy of anaconda is stored in a convenient location that won't interfere with anything else. Once Anaconda is installed, you'll see this prompt:

```
Do you wish the installer to initialize Anaconda3
by running conda init? [yes|no]
[no] >>> 
```

Here you should use the default setting `no`.

Finally, WASP will be installed. There will be one last prompt to proceed with the installation:

```
Proceed ([y]/n)?
```

You can just enter the default `y`.

You can check that `wasp_map` was installed successfully by running `wasp_map-map -h` to get the help text for `wasp_map-map`.


## pyQuASAR-genotype

Installing and setting up QuASAR is comparatively easy, just follow:

```
pip3 install --user pyQuASAR
pyQuASAR-download
pip3 install --user pyQuASAR_genotype
pyQuASAR-genotype -h
```


## pileups-count

The `pileups-count` command is part of a package called `pileups` I made to help with handling [pileup files](http://www.htslib.org/doc/samtools-mpileup.html). You can get it by simply running:

```
pip3 install --user pileups
pileups-count -h
```
