# AMR plasmid de-novo assembly 

### Setup

Fast5 files saved to: 

`/mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_fast5`

## Basecalling using albacore

To basecall fastq files from fast5 files. `nohup` and `&` are optional: 

```
nohup read_fast5_basecaller.py -i /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_fast5/ -t 96 -s /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_fastq/ -f FLO-MIN106 -k SQK-LSK108 -o fastq -r &
```

` -i = [INTPUT FILE LOCATION]` : location of fast5 pass folder  
` -t = [NUMBER]` : number of threads to run on this job. Check your comp (probably 6-8)  
` -s = [OUTPUT FILE LOCATION]` : where to save fastq files to  
` -f = [FLOW CELL TYPE]` : type of flow cell used, FLO-MIN106  
` -k = [KIT CODE]` : code of seqencing kit used, SQK_LSK_108  
` -o = [OUTPUT TYPE]` : output as fastq or fast5  
` -r = [RECURSIVE]` : looks for files in directories within directories  

Fastq files will be written to:  

`/mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_fastq/`  

(Optional) If you want to you can watch how many sequences are being writen by changing to the directory 
where your fastq files are being writen (workspace/pass) and typing this:
```
watch -n 10 'find . -name "*.fastq" -exec grep 'read' -c {} \; | paste -sd+ | bc'
```

`watch` = envoke 'watch' command : tells the computer to watch for something  
`- n 10` = every 10 seconds  
`find .` = look in every directory below where you are   
`-name "*.fastq"` = : target of find which will find every file with 'fastq' in its name  
`-exec` = execute the following command on find .  
`grep` = command to look for stuff within each file found by 'find . -name'  
`'read'` = the thing grep is looking for in the header of each sequence (1 per sequence)  
`-c` = count command : count the number of the occourance 'read' in all files  
`{} \; | paste -sd+ | bc'` = paste the output from grep to basic calculator to display a count on the screen  

### Concatinate fastq files into one:

Once albacore has finished concatinate all the fastq files into one and count them.
```
cat Sev_plasmid_fastq/workspace/pass/*fastq > Sev_plasmid_cat.fastq
grep 'read' Sev_plasmid_cat.fastq -c
```

## Adapter trimming using the program porechop

Once bases are called, adapter sequences and optional barcode sequences must be removed from the reads before assembly.
Porechop will do this adequately with default settings in this case. For more information and settings call `porechop`.

```
porechop -i Sev_plasmid_cat.fastq -o Sev_plasmid_cat_chop.fastq --discard_middle
```

`i` Input file path  
`-o` output file path
`--discard_middle` remove chimeric sequences

### Count reads

Count the number of basecalled reads that passed quality filters from porechop 

```
grep 'read' /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_cat_chop.fastq -c
```

## Resample reads

To resample 150,000 reads with the same length distrobution but no less than 1000bp:

```
fastqSample -U -p 150000 -m 1000 -I /mnt/gpfs/rob/Sev_plasmid/plasmid_cat_chop.fastq -O /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_cat_chop_150k.fastq
```
`-U` = unpaired reads  
`-p` = total number of random reads (add `-max` to get longest reads)  
`-m` = minimum read lenght  
`-I` [INPUT FILE]  
`-O` [OUTPUT FILE]  

## De-novo plasmid assembly using CANU

To assemble the plasmid without a referance sequence. Use the default settings of canu.

```
canu -p plasmid_assembly -d Sev_plasmid_assembly genomeSize=150k -nanopore-raw /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_cat_chop_150k.fastq

```

`canu` = Envoke canu command  
`-p` = output directory name  
`-d` = temporary output directory name  
`genomeSize` = very aproximate genome size, better to overestimate  
`-nanopore-raw` = type of read (uncorrected raw)  

Check assembly in bandage. Then use the .gfa file to find the overlap at the two ends and trim the fasta file at that point.
Alternitivly search the start or end of the assembly to find the overlap trim point. This is importent for nanopolish to have
better end alignments. 

Save trimmed plasmid sequence to `/mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_trimmed_150k.fastq` with the header in the fastq file set
to `>Sev_plasmid`.

## Polish the assembly with Nanopolish.

Nanopolish is used to increase the consensus sequence accuracy. You can reduce the time it takes to index the reads by only indexing the fast5 files randomly subsampled using `fastqSample`. 

To match the subsampled fastq file reads back to the fast5 files, construct a new directory to copy the fast5 files to and call `fast5seek`

```
mkdir /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_sample_fast5/
fast5seek -i /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_fast5/ -r /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_cat_chop_150k.fastq | xargs cp -t /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_sample_fast5/

```
`-i` : input fast5 dir
`r` : subsampled fastq reads file 
`| xargs cp -t` : pipe output to xargs and copy matching fast5 files

Nanopolish needs access to signal data in fast5 files that corrispond to fastq reads in the assembly.
Index `Sev_plasmid_cat_chop_150k.fastq` reads with their corrisponding fast5 files that contain raw squiggle data. 

Call `nanopolish index`.

```
nanopolish index -d /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_sample_fast5/ /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_cat_chop_150k.fastq
```

Align `Sev_plasmid_cat_chop_150k.fastq` with the draft assembly from canu using `minimap2` and index them using `samtools`.

```
minimap2 -ax map-ont -t 96 /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_trimmed_150k.fastq /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_cat_chop_150k.fastq | samtools sort -o /mnt/gpfs/rob/Sev_plasmid/Sev_reads_sorted.bam -T reads.tmp
samtools index reads.sorted.bam
```

Check the `Sev_reads_sorted.bam` file has reads in it.

```
samtools view /mnt/gpfs/rob/Sev_plasmid/Sev_reads_sorted.bam | head
```

Polish the assembly from canu call `nanopolish_makerange.py` for assemblys > 1mb and `variants` <100 kb

```
nanopolish variants --consensus Sev_plasmid_polished.fasta  -w "Sev_plasmid" -r /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_trimmed_150k.fastq -b /mnt/gpfs/rob/Sev_plasmid/Sev_reads_sorted.bam -g /mnt/gpfs/rob/Sev_plasmid/Sev_plasmid_trimmed_150k.fastq
```




