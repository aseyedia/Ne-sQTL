﻿# neanderthal-sqtl

## Introduction

This is my (Arta Seyedian) first project with Dr. Rajiv McCoy's group. Today is 9/26/2018 and I am applying the general guidelines for documentation and organziation to this project repository. Most of the work I have done so far 
has been without this method of organization so please forgive me if it's a bit hard to go through these directories and files and fully understand what I have been going for here.

The crux of this project is to find sQTLs (splicing quantitative trait loci) among the DNA modern humans have inherited from neanderthals as a result of interbreeding that took place between 100,000 - 40,000 years ago [1]. 
These sQTLs basically determine mRNA transcript splicing patterns, which lead to different proportions of mRNA isoforms which leads to differential protein expression, which impacts the expression of quantitative traits,
such as height [2].


## Getting Started
Placeholder for ultimately instructing users on how to run all of these scripts to get identical output.

### Prerequisites
* Access to a high-performance cluster
* LeafCutter 0.2.8
* FastQTL v2.184
* SAMtools or HTSlib

### Installing
How to install what they will need to install.

## Running the Tests
Step-by-step instructions with intermediate ends and full explanations of important code.

## Built With
* R 3.4.4 (2018-03-15) -- "Someone to Lean On"
* Python 2.7-anaconda
* Obviously some other stuff

## Contributing
* Thank the important people

## Authors

## Acknowledgements


Updates
----------------------------------------------------------------------------------------------------------------------------------------
### 12/11/2018
I found this script from [this stack overflow post](https://stackoverflow.com/questions/2074687/bash-command-to-remove-leading-zeros-from-all-file-names/2074704) that removes all leading zeros from any file in that directory that has a file name that starts with zeros. Not sure exactly how it works and I honestly don't really care. Just hope whoever is using this doesn't use input files that start with zeros.

`shopt -s extglob; for i in *.split; do mv "$i" "${i##*(0)}"; done`

All of the extra code I have to write just to call the LeafCutter scripts is getting out of hand. I really should consider writing a master script soon. Maybe after I call the intron clustering step, I could also call a `rm *.split` to clear the clutter in the working directory, but that's for another time. I should note that the only reason I'm going through all of this trouble is that from all indications, `leafcutter_clustering.py` **only** takes "text file[s] with all junction files to be processed," and not the junction files themselves, unless I'm mistaken which I might be.

Ok, I just checked and it doesn't. Going to `mkdir intronclustering` to reduce working directory clutter. 

~~python ../clustering/leafcutter_cluster.py -j ${SLURM_ARRAY_TASK_ID}.split -r intronclustering/ -m 50 -o testNE_sQTL -l 500000~~

From the [documentation](http://davidaknowles.github.io/leafcutter/articles/Usage.html#step-2--intron-clustering)
>This will cluster together the introns fond in the junc files listed in test_juncfiles.txt, requiring 50 split reads supporting each cluster and allowing introns of up to 500kb. The predix testYRIvsEU means the output will be called testYRIvsEU_perind_numers.counts.gz (perind meaning these are the per individual counts).

I also added the `-r` flag to indicate that all of the output files should go to our boy `intronclustering/`. The other flag parameters, I don't know about. For example, I don't know why I should make the minimum cluster reads 50 instead of 30 (which is the default), or why the max intron length should be 50,000bp as opposed to 100,000bp. I'll ask Rajiv if he wants to alter these values but for now I'm going to truck along.

~~Since this is starting to get complicated, I'm going to show what the master script will look like immediately preceeding the above `leafcutter_cluster.py` call:~~


~~split -d -a 6 -l 1 --additional-suffix '.split' test_juncfiles.txt ''~~
~~shopt -s extglob; for i in *.split; do mv "$i" "${i##*(0)}"; done~~
~~sbatch intron_cluster.sh~~


I just realized I'm going to have to do this with GNU-Parallel when I do the full dataset which is frustrating but I guess I gotta if we're going to be efficient. The guy never got back to me about how to best call it but I think I figured it out. Looking at MARCC's documentation, it seems that the most important things are (a) to not use job arrays, (b) `parallel="parallel --delay .2 -j 4 --joblog logs/runtask.log --resume"` and (c) `$parallel "$srun python example_lapack.py {1}000 6" ::: {1..10}`. I feel like meeting with the directors of MARCC made things more complicated for me, but it's okay. I'm actually going to talk to Rajiv about what he thinks about using GNU-Parallel. It might be useful for doing to `.sra` to `.bam` conversion and maybe some of the other more computationally intensive stuff but given that I don't quite understand how it works, I'm not sure how to implement it. **UPDATE:** yeah it's cool I don't need to use it. MARCC is so slow I can't launch a job(s) without first waiting like 4000 minutes.

Okay, so I just figured out that `leafcutter_cluster.py` doesn't work in a job array, probably because all of the files are merged in the end. Shouldn't be too taxing though, because each `.junc` file is only a few MB in size.

Ignore what I've striked above. Here is what the code must look like:

```
ml python/2.7-anaconda
mkdir intronclustering/
python ../clustering/leafcutter_cluster.py -j test_juncfiles.txt -r intronclustering/ -m 50 -o testNE_sQTL -l 500000
```

For PCA calculation,
```
python ../../scripts/prepare_phenotype_table.py testNE_sQTL_perind.counts.gz -p 10 # works fine; there's no option for parallelization.
ml htslib; sh testNE_sQTL_perind.counts.gz_prepare.sh
```

Here I am in FastQTL world again. I am going to just download the full genotype matrix in VCF format (~447 Gb): `./prefetch -X 500G --ascp-path '/software/apps/aspera/3.7.2.354/bin/ascp|/software/apps/aspera/3.7.2.354/etc/asperaweb_id_dsa.openssh' ../whole-genome-vcf-cart/cart_prj19186_201812111835.krt > ../../logs/vcf-dl-log.out 2> ../../logs/vcf-dl.err.out` I asked Rajiv which vcf I should download but he's away so I'm just going to do it.

### 12/10/2018
I was able to use `src/12-10-2018/filter_bam.sh` to get rid of all unplaced contigs from the bam files before converting them to `.junc` files and sending them through LeafCutter. `filter_bam.sh` does not remove mitochondrial DNA from the files and I am not sure if LeafCutter can process mitochondiral intron cluters so we will see how this plays out.  

`ls *.filt | tail -n +1 >> filtbamlist.txt`

Doing the same thing; going to round up all of the filtered bam files and put them in this here text file to do the `junc` conversion.

Don't forget to use `wc -l` on `filtbamlist.txt` or `bamlist.txt` or any other text file to be used with a job array to figure out what the job array size should be (if 0 indexed, -1 from size). I need to create some sort of master script that calls all of the job array scripts and basically does everything at once. I made `filter_bam.sh` and `bam2junccall.sh` functional as job arrays today. 

For the intron clustering step, `clustering/leafcutter_python.py -j` intakes a "text file with all junction files to be processed," so I'm trying to figure out how to make this into a job array. Of course, I could make it so each individual line in that text file gets its own text file. 

`split -d -a 6 -l 1 --additional-suffix '.split' <input> ''`

The above command splits a text file. The `-a` flag designates the suffixes (for 5,000 files, 6 should suffice), the `-l` flag indicates lines per output file (1) and the `-d` flag indicates numeric suffixes (`xaa, xab, xac...`) rather than alphabetic suffixes (`x01, x02, x03...`). The `--addtional-suffix` flag ensures that the following command, which would remove leading zeros, can target the resulting files. Lastly, the `''` ensure there is no prefix. The problem with this is that `$SLURM_ARRAY_TASK_ID` generates numbers like `1, 2, 3, etc` but it shouldn't be too difficult to remedy this.

### 12/08/2018
I wrote a script `scratch/filter_bam.txt`, picking up from yesterday. MARCC isn't working right now but I'll try it again tomorrow morning. 

Note: I feel like I didn't mention this before, but `scratch/convertgTEX.txt` is my obviously flawed draft to parallelize converting the gTEX files to `.bam`. I'll have to do the same thing for converting them to `.junc` and pretty much every ensuing step of the project. God what a pain. I'm going to talk to Rajiv about how he thinks we should approach this. 

### 12/07/2018
I met Rajiv about the problem that I wrote about in the README last night without committing the changes and pushing to GitHub (it's okay). Basically the code (`prepare_phenotype.py`) can't handle string inputs that are not simply 'X' or 'Y'. Rajiv slapped together `/data/12-07-2018/GRCh37.bed`, which has the the chromosome number/letter and sequence range (0-terminal) to be used with `samtools` to filter anything that is **not** that from the `bam`. He used the following command: `cat input.txt | sed 's/chr//g' | awk '{print $1"\t0\t"$2}' > GRCh37.bed` and his source was the [UCSC website](http://hgdownload.cse.ucsc.edu/goldenpath/hg19/encodeDCC/referenceSequences/male.hg19.chrom.sizes).

So now I have to figure out which `samtools` commands/flags I need to use to do this. I have to now go back to the previous step, after converting the `sra` files to `bam`s but before turning the `bam` files to `junc`.

From the [samtools documentation](http://www.htslib.org/doc/samtools.html):
>-L FILE
>Only output alignments overlapping the input BED FILE [null].

`samtools view -L GRCh37.bed <file>`

I created a text list of files to parallelize the above command: `ls | tail -n +1 >> bamlist.txt `; I'm going to have `$SLURM_ARRAY_TASK_ID` correspond to each line in the file.

### 12/06/2018
`interact` isn't working on MARCC right now so I'm still using our development node, which should be more than enough in terms of resources. I have converted our 10 `.sras` to `.bams`. Now, following the LeafCutter guide, I'm going to convert them to .junc files, and then I'm going to cluster the introns. 

```
for bamfile in `ls example_geuvadis/*.bam`
do
    echo Converting $bamfile to $bamfile.junc
    sh ../scripts/bam2junc.sh $bamfile $bamfile.junc
    echo $bamfile.junc >> test_juncfiles.txt
done
```

Above is the script provided by LeafCutter to do the .bam -> .junc conversion. Adapted for my needs, the script will look a little something like this:

```
#!/bin/bash
# used in bam2junc.sh
ml samtools
for bamfile in `ls *.bam`
do
    echo Converting $bamfile to $bamfile.junc
    sh /scratch/groups/rmccoy22/leafcutter/scripts/bam2junc.sh $bamfile $bamfile.junc
    echo $bamfile.junc >> test_juncfiles.txt
done
```

bam2junc.sh:

```
#!/bin/bash

leafCutterDir='/scratch/groups/rmccoy22/leafcutter/' ## use only if you don't have scripts folder in your path
bamfile=$1
bedfile=$1.bed
juncfile=$2


if [ ! -z "$leafCutterDir" ]
then
        samtools view $bamfile | python $leafCutterDir/scripts/filter_cs.py | $leafCutterDir/scripts/sam2bed.pl --use-RNA-strand - $bedfile
        $leafCutterDir/scripts/bed2junc.pl $bedfile $juncfile
else
        if ! which filter_cs.py>/dev/null
        then
                echo "ERROR:"
                echo "Add 'scripts' forlder to your path or set leafCutterDir variable in $0"
                exit 1
        fi
        samtools view $bamfile | filter_cs.py | sam2bed.pl --use-RNA-strand - $bedfile
        bed2junc.pl $bedfile $juncfile
fi
rm $bedfile
```

Again, these scripts are altered from what is included in LeafCutter. 

After running this, I got a lot of "problematic spliced reads." Of course, like I said earlier, I'm only using about 10 sra files, so I'm keeping the console output in MARCC. I hope that isn't too big of a deal. The files referenced above are named `callbam2junc.sh` and `bam2junc.sh`, respectively. The first script I pulled from the worked-out example that doesn't actually work that comes with LeafCutter.

I proceeded to the "Intron Clustering" portion of the LeafCutter:

`ml python/2.7-anaconda`

`python /scratch/groups/rmccoy22/leafcutter/clustering/leafcutter_cluster.py -j test_juncfiles.txt -m 50 -o testgTEX -l 500000`

From the [LeafCutter documentation](http://davidaknowles.github.io/leafcutter/articles/Usage.html#step-2--intron-clustering):

>This will cluster together the introns fond in the junc files listed in test_juncfiles.txt, requiring 50 split reads supporting each cluster and allowing introns of up to 500kb. The predix [testgTEX] means the output will be called [testgTEX]_perind_numers.counts.gz (perind meaning these are the per individual counts).

Here's what output `testgTEX_perind_numers.counts.gz` looks like:
```
SRR1068929.sra.bam SRR1068855.sra.bam SRR1069141.sra.bam SRR1068808.sra.bam SRR1068977.sra.bam SRR1069024.sra.bam SRR1069097.sra.bam SRR1069231.sra.bam SRR1069188.sra.bam SRR1068880.sra.bam
14:20813246:20813553:clu_1_NA 20 67 9 30 87 1 2 15 86 4
14:20813285:20813553:clu_1_NA 0 0 0 0 1 0 33 18 0 171
14:20822406:20822968:clu_2_NA 14 48 19 15 55 1 19 29 68 103
14:20822406:20822994:clu_2_NA 5 5 0 1 24 0 6 6 4 7
14:20839791:20840892:clu_3_NA 11 13 32 11 18 1 20 12 1 16
14:20839791:20841170:clu_3_NA 8 3 16 6 30 2 5 10 14 7
14:20841016:20841170:clu_3_NA 7 9 23 7 21 3 19 12 2 8
14:20916970:20917061:clu_4_NA 2 18 5 3 18 0 8 18 115 2
14:20916970:20917123:clu_4_NA 56 93 64 50 102 14 47 53 95 113
14:20917425:20919416:clu_5_NA 2 6 6 1 20 3 1 9 14 0
```

I think each line is an intron and the columns represent the number of split reads from the corresponding file that supports that intron, and the `clu_X` represents which "cluster" it's in. Cool. Now we can get into splicing QTL analysis, which may be the tricky part. I'm doing this all with just 10 SRA files and it's really highlighting the need to parallelize this process, because if we don't, it would take like 20 years just to get to this part using the full dataset.

[Splicing QTL Analysis](http://davidaknowles.github.io/leafcutter/articles/sQTL.html):

`python /scratch/groups/rmccoy22/leafcutter/scripts/prepare_phenotype_table.py /scratch/groups/rmccoy22/bamfiles/testgTEX_perind.counts.gz -p 10`

I get this error message:
```
[aseyedi2@jhu.edu@rmccoy22-dev bamfiles]$ python /scratch/groups/rmccoy22/leafcutter/scripts/prepare_phenotype_table.py testgTEX_perind.counts.gz -p 10
Starting...
Parsed 1000 introns...
Traceback (most recent call last):
  File "/scratch/groups/rmccoy22/leafcutter/scripts/prepare_phenotype_table.py", line 171, in <module>
    main(args[0], int(options.npcs) )
  File "/scratch/groups/rmccoy22/leafcutter/scripts/prepare_phenotype_table.py", line 99, in main
    chrom_int = int(chr_)
ValueError: invalid literal for int() with base 10: 'GL000219.1'
```
Going to tell Rajiv about it.

### 12/05/2018
Got up this morning and looks like firing off the command `for f in $PWD/*.sra; do ./sam-dump $f | samtools view -bS - > $f.bam; done` with a 4-hour long interactive session wasn't even enough for just 10 files. I'm converting the rest of the files on `rmccoy22-dev`. 

### 12/04/2018
We met with the directors of MARCC last week and we talked about using a tool called `GNU-Parallel` which, from what I remember, is a good way to multithread jobs in a way such that if one crashes, we can pick up from where it left off or at least figure out what the problem was. Also, Keven wanted us to measure for him the download speed of the SRA files so he has an idea of how fast a TB downloads. I'm gonna do a dry run with LeafCutter and only about 10 files to see how it works, first. Then I'm going to study `GNU-Parallel` a bit and then email Kevin saying I don't understand it. For the purposes of this practice run, I'm using `for f in $PWD/*.sra; do ./sam-dump $f | samtools view -bS - > $f.bam; done` to convert the boys into `.bam` files.

## Update
`for f in $PWD/*.sra; do ./sam-dump $f | samtools view -bS - | tee $f.bam; done`

I tried out the above command and it was a nightmare. I thought it would be readable for whatever reason. It would be cool if I figure out how to add like a progress bar or something just so I can monitor the progress of the conversion, but I guess I won't even have to with parallelization.

### 11/27/2018
Using `nohup` didn't work, or something else happened, but the cart files did not download. Disappointingly. I'm going to try again, but this time with `screen`. I'm going to use `./prefetch -X 50000000000 /scratch/groups/rmccoy22/sratools/cart_prj19186_201811221749.krt > /scratch/groups/rmccoy22/logs/SRA_DL.out`, with the output redirected to a `logs` folder so that the download progress can be monitored by `tail -f`. Just to clarify, I am using the dtn node. Also, Ubuntu's built-in text editor isn't working on my laptop, weirdly.

### 11/26/2018
The more things change, the more they stay the same. We are meeting with the director of MARCC on Wednesday to discuss our options when dealing with big ol' datasets. Looks like using `~/scratch` is a no-go, but `~/work` apparently has up to 50TB of space AND is a scratch partition, so that's what we're going to use. We're still waiting to meet with the dude on Wednesday but in the meantime, I'm going to play around with a small subset of the total data to see what's what. I already made some progress last week; I came up with a little ditty like this: `for f in $PWD/*.sra; do ./sam-dump $f | samtools view -bS > $f.bam; done`. However, this one-off command doesn't utilize parallelization, so Rajiv recommended something that makes use of SLURM's capabilities:
```
#!/bin/bash
#SBATCH --job-name=convert_gtex
#SBATCH --time=12:00:00
#SBATCH --partition=shared
#SBATCH --nodes=1
# number of tasks (processes) per node
#SBATCH --ntasks-per-node=1
#SBATCH --array=1-10%10
#SBATCH --output=/scratch/users/rmccoy22@jhu.edu/logs/slurm-%A_%a.out

#### load and unload modules you may need
module load samtools

# test.txt contains a list of dbGaP run IDs (e.g., SRR1068687) 
line=`sed "${SLURM_ARRAY_TASK_ID}q;d" /scratch/users/rmccoy22@jhu.edu/test.txt`

echo "Text read from file: $line"

cd /scratch/users/rmccoy22@jhu.edu/dbgap/sra

# Download the .sra file according to SRA ID
~/progs/sratoolkit.2.9.2-centos_linux64/bin/prefetch \
  --ascp-path '/software/apps/aspera/3.7.2.354/bin/ascp|/software/apps/aspera/3.7.2.354/etc/asperaweb_id_dsa.openssh' $line

# Convert from .sra to .bam
~/progs/sratoolkit.2.9.2-centos_linux64/bin/sam-dump $line.sra |\
  samtools view -bS - > $line.bam
#mv $line.bam /work-zfs/rmccoy22/rmccoy22/gtex/RootStudyConsentSet_phs000424.GTEx.v7.p2.c1.GRU/BamFiles/$line.bam
#rm /scratch/users/rmccoy22@jhu.edu/dbgap/sra/$line.*

echo "Finished with job $SLURM_ARRAY_TASK_ID"```
```
Since we won't need to delete and move things back and forth, I have commented-out the parts of the code that are now both practically and conceptually obsolete. I'm going to use the following command: `nohup ./prefetch -X 50000000000 /scratch/groups/rmccoy22/sratools/cart_prj19186_201811221749.krt &` to download the boys (the boys being the SRA files).

### 11/12/2018
Our journey continues. Rajiv accidentally deleted the dbGaP folder so now we have to download the sra files again. The cart file I was using before was incomplete so I got a new one. What I need to do now is to download the sra files in chunks to /scratch, convert the files to .bam, then move the converted files to /data. A good thing to keep in mind is to set the max size for prefetch to a high number so that the download won't be capped. I used the command `./prefetch -s ~/scratch/ncbi/cart_prj19186_201811121649.krt > ../../sizeOfCart.txt` to get a text file with the sizes of all the sra files so that I can find the best possible chunking. Actually, now that I think about it, that's not necessary; chunking by 3TB is probably fine and that's what I'm going to do, but it's cool that I have the sizes of the sra files. So now I have to write a bash script that will download to scratch in a 3TB chunk, convert to *.bam, move *.bam to /data, delete pre-converted data as well as bam data on /scratch, repeat for remainder of sra files in 3TB chunks.

### 11/09/2018
Disregard that last entry, the download was unsuccessful and after speaking with Rajiv, he found a repository of GTEx data he had downloaded previously minus some larger files had yet to download. He offered to convert the files to `.bam` format for me after repeated difficulties on my end working with sra-toolkit. The total size of the RNA-Seq data that will be used is ~16TB. Additonally, he will supply me the required `.vcf` file for QTL mapping via FastQTL. I did some investigation into potentially creating an R shiny app to visualize the data and it seems not only easy, but clear cut for LeafCutter users.

### 11/08/2018
After weeks of learning how to use leafcutter, installing a dependent QTL mapper (FastQTL), and fixing certain problems with using the GTEx data, I was finally approved by Rajiv as a downloader on dbGaP, and am about to download RNA-Seq GTEx data from dbGaP using sra-toolkit's `prefetch` command. The command will look something like this: `nohup ./prefetch -v -O /home-1/aseyedi2@jhu.edu/data/aseyedi2/GTEx_SRA cart_DAR72305_201811081445.krt &` and I will be logged into the dtn2 server, which is intended for large data transfers. It's about 16 TB. 

### 11/02/2018
Got rid of the contents of the scratch folder, updated the README for `TissueMerge.R`. Added analysis folder.

### As of 10/17/2018
Fixed a problem with the column headers in the merged Neanderthal x Altrans data.

### As of 10/15/2018
I realized that I didn't include the Ensembl ID codes in my Neanderthal x Altrans data merge, so changed the TissueMerge.R code to make it do that.

### As of 10/11/2018
The repo was cloned directly onto MARCC and reorganized the data files and purged some useless stuff, including some pre-concatenated output files. Best thing I did was find a bash command that finds and appends to the gitignore file all files greater than 100MB, so I can now host this project directly on MARCC and update it through MARCC without overcomplicating getting stuck accidentally committing a massive file.

### As of 10/09/2018
I am writing this on 10/11 because I forgot to update this lab journal with what I did. I finished writing the R script that would merge the altrans tissue sQTL data with the Neanderthal rsID data and also the Bash script that would call that R script. Generated merged files for each of the Neanderthal regions and each of the tissue types. Stored them in files; now they're in /data. 

### As of 10/04/2018
I wrote some code in /scratch that merges one altrans tissue sQTL file with one neanderthal rsID file, while stripping some of the unnecesary columns. The next step is to figure out how to automate this for each one of the 3 neanderthal files, such that one neanderthal file (let's say, EASmatch.txt) merges with each one of the dozen or so tissue data files and either appends this merge to the resulting data table from the first merge, and includes maybe some header information about which tissue file that it merged with OR (and this is probably what I'll end up doing because it just occurred to me and makes more sense and seems obvious in hindsight) just make a merged file for each one of the tissue files that the neanderthal rsIDs merged with.

### As of 9/26/2018
I'm still at the beginning stages of the project. No serious analysis has taken place just yet; however, I have matched the neandertal SNP positions with the human rsIDs from the 1k data. The next step would be to match
these neanderthal rsID data with known sQTLs sequenced by altrans as a proof of concept, before we move onto using LeafCutter on GTEx data to create a full map of human sQTL data to match against the neanderthal rsIDs.

	
## Bibliography

1. Rogers Ackermann, Rebecca & Mackay, Alex & L. Arnold, Michael. (2015). The Hybrid Origin of “Modern” Humans. Evolutionary Biology. 43. 10.1007/s11692-015-9348-1. 
2. Silver, L. (2001). QTL (Quantitative Trait Locus). Encyclopedia of Genetics, 1593-1595. doi:10.1006/rwgn.2001.1054
