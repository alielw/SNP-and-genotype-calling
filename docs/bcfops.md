## 4. Operations with BCF files
The next sections exemplify how to do operations with VCF/BCF files, including merging, subsetting and filtering, mostly using bcftools and awk.

### Samples and SNPs 
A list of the samples contained in the file can be obtained using simple linux commands or `bcftools query`, and can be counted with `wc`:
```bash
bcftools view -h bcftools/bcftools-Hmel201001.bcf | grep "^#CHROM" | sed 's/.*FORMAT\t//; s/\t/\n/g'
bcftools view -h bcftools/bcftools-Hmel201001.bcf | grep "^#CHROM" | sed 's/.*FORMAT\t//; s/\t/\n/g' | wc -l
bcftools query -l bcftools/bcftools-Hmel201001.bcf
bcftools query -l bcftools/bcftools-Hmel201001.bcf | wc -l
```
The number of variants can be counted using `bcftools view -H` and `wc`, but it includes all kinds of variants. To check particular ones, we need to use the flags `-v/-V` (or `--types/--exclude-types`) to include or exclude certain variant. 
```bash
# all variants
bcftools view -H bcftools/bcftools-Hmel201001.bcf | wc -l
# only SNPs
bcftools view -H -v snps bcftools/bcftools-Hmel201001.bcf | wc -l
# only indels
bcftools view -H -v indels bcftools/bcftools-Hmel201001.bcf | wc -l
```
Multiallelic SNPs (with >2 alternate alleles) can be extracted with awk or with the `-m` flag:
```bash 
bcftools view -H -v snps bcftools/bcftools-Hmel201001.bcf | awk '$5 ~ /,/' | less -S
bcftools view -H -v snps -m 3 bcftools/bcftools-Hmel201001.bcf | less -S
```

### Merging and subsetting files
Non-overlapping files with the same samples can be easily merged with `bcftools concat`. Let's merge the bcf files for our three scaffolds, index it and have a look:
```bash
bcftools concat bcftools/*.bcf -O b -o bcftools/bcftools-concat.bcf
bcftools index bcftools/bcftools-concat.bcf
bcftools view -H bcftools/bcftools-concat.bcf | less -S
```
Likewise, we can subset SNPs within particular regions, for example to extract the variants within the region 5000-5500bp in scaffold Hmel201001 and 15000-20000 in Hmel201003:
```bash
bcftools view -H bcftools/bcftools-concat.bcf Hmel201001:5000-5500,Hmel201003:15000-20000 | less -S 
```
Subsetting a number of samples is also possible, for example to get only the first 10 samples and the last 10 samples:
```bash
SAMPLES=$(bcftools query -l bcftools/bcftools-concat.bcf | head -n 10 | tr '\n/' ','| sed 's/,$//')
bcftools view -s $SAMPLES bcftools/bcftools-concat.bcf -O b -o bcftools/bcftools-concat-first16.bcf
bcftools index bcftools/bcftools-concat-first16.bcf

SAMPLES=$(bcftools query -lDepthPerSampleHC bcftools/bcftools-concat.bcf | tail -n 10 | tr '\n/' ','| sed 's/,$//')
bcftools view -s $SAMPLES bcftools/bcftools-concat.bcf -O b -o bcftools/bcftools-concat-last16.bcf
bcftools index bcftools/bcftools-concat-last16.bcf
```
And then merge them into a single file with `bcftools merge`:
```bash
bcftools merge bcftools/bcftools-concat-first16.bcf bcftools/bcftools-concat-last16.bcf -O b -o bcftools/bcftools-merge.bcf
bcftools index bcftools/bcftools-merge.bcf
```

### Comparing and combining outputs
We are now going to compare the SNPs called by bcftools and GATK. First, let's merge the bcf files for each scaffold:
```bash
mkdir comparison
# merge bcftools calls
bcftools concat bcftools/bcftools-Hmel*.bcf -O b -o comparison/bcftools.bcf
bcftools index comparison/bcftools.bcf
# merge GATK calls
bcftools concat gatk/gatk-Hmel*.bcf -O b -o comparison/gatk.bcf
bcftools index comparison/gatk.bcf
```
Let's see the total number of SNPs called by each programme:
```bash
bcftools view -H comparison/bcftools.bcf | wc -l
bcftools view -H comparison/gatk.bcf | wc -l
```
As you can see both programmes called a similar number of SNPs. Let's now find out the SNPs that are shared between both calls using ``bcfools isec``:
```bash
bcftools isec -O b comparison/bcftools.bcf comparison/gatk.bcf -p comparison/isec
```
This will result in the following files:
```ls -lh comparison/isec```

>``total 265K``<br>
>``drwxr-xr-x 2 myuser cs 4.0K Feb 17 02:06 .``<br>
>``drwxr-xr-x 3 myuser cs 4.0K Feb 17 02:06 ..``<br>
>``-rw-r--r-- 1 myuser cs  31K Feb 17 02:06 0000.bcf``<br>
>``-rw-r--r-- 1 myuser cs  268 Feb 17 02:06 0000.bcf.csi``<br>
>``-rw-r--r-- 1 myuser cs  41K Feb 17 02:06 0001.bcf``<br>
>``-rw-r--r-- 1 myuser cs  254 Feb 17 02:06 0001.bcf.csi``<br>
>``-rw-r--r-- 1 myuser cs  76K Feb 17 02:06 0002.bcf``<br>
>``-rw-r--r-- 1 myuser cs  279 Feb 17 02:06 0002.bcf.csi``<br>
>``-rw-r--r-- 1 myuser cs  85K Feb 17 02:06 0003.bcf``<br>
>``-rw-r--r-- 1 myuser cs  272 Feb 17 02:06 0003.bcf.csi``<br>
>``-rw-r--r-- 1 myuser cs  559 Feb 17 02:06 README.txt``<br>

The content of the BCF files is explained in the README.txt file:
* ``0000.bcf``: private to bcftools
* ``0001.bcf``: private to gatk
* ``0002.bcf``: bcftools records shared with gatk
* ``0003.bcf``: gatk records shared with gatk

### Filtering
The steps above got us a quite raw sets of SNPs, but usually we want to apply some filters depending on the sort of downstream analyses that we intend to do. Filtering usually consist on establishin some sort of threshold for the SNP quality in the QUAL field and some of the variables encoded in the INFO field (e.g. depth, mapping quality, etc.), but more sophisticated filterin is possible based on genotype-specific variables (e.g. discarding genotypes below a certain quality or with fewer than a number of reads). Although in principle it is possible to do soft-filtering of SNPs and simply tag them as `PASSED` or `FAILED` in the VCF/BCF file, in practice it is more useful to do hard-filtering and simply remove all those SNPs (and genotypes) that did not meet our thresholds. We are going to focus on only the SNPs, so let's extract only the snps from the variants shared by bcftools and gatk:

```bash
mkdir filtering
bcftools view -v snps comparison/isec/0002.bcf -O b > filtering/snps.bcf
bcftools index filtering/snps.bcf
```

#### SNP-based filtering
bcftools allows applying filter either with ``bcftools view`` or with ``bcftools filter`` and using information encoded in the QUAL or INFO fields, also allowing expression with multiple conditions and basic arithmetics (more details [here](https://samtools.github.io/bcftools/bcftools.html#expressions)). These are some examples:

##### Filter by SNP quality
An obvious filter is to exclude (-e) SNPs below a quality threshold:
```bash
bcftools view -e 'QUAL<20' -O b filtering/snps.bcf > filtering/snps.QS20.bcf
```
Filters can also be specified as includes (-i), the equivalent of the one above is:
```bash
bcftools view -i 'QUAL>=20' -O b filtering/snps.bcf > filtering/snps.QS20i.bcf
```
Comparing the contents show they are identical:
```bash
diff -s <(bcftools view -H filtering/snps.QS20.bcf) <(bcftools view -H filtering/snps.QS20i.bcf)
```
##### Filter by SNP depth 
An example to exclude SNPs with depth <30:
```bash
bcftools view -e 'INFO/DP<100' -O b filtering/snps.bcf > filtering/snps.DP100.bcf
```
An important note here is to be aware that if the file is further processed so that only part of the individuals are used, the fields in INFO may not be updated and it would be then unreliable to filter out using any information from that field.

##### Filter by mapping quality
Another typical filter is to remove SNPs with low mapping quality (RMS MQ: root mean square of the mapping quality of reads across all samples). For examples, all SNPs with RMS MQ < 20 can be discarded with the following command:
```bash
bcftools view -e 'MQ<20' -O b filtering/snps.bcf > filtering/snps.MQ20.bcf
```
##### Filter by allele frequency
For population genetic analyses it is frequently advised to remove variants below a certain allele frequency, as these ones are difficult to tell apart from sequencing errors. For example, to exclude all SNPs with a minor allele frequency (MAF) below 5% we would run:
```bash
bcftools view -e 'MAF<0.05' -O b filtering/snps.bcf > filtering/snps.MAF005.bcf
```
##### Filter by sample coverage
Another typical situation is to want to exclude all SNPs for which only a small fracion of all the individuals have sequence data. That requires using the total number of alleles in the called genotypes (AN). In the case of a diploid species, this can be used to filter SNPs by number of individuals genotyped. For example, we can remove all SNPs genotyped at less than 13 individuals:
```bash
bcftools view -e 'AN/2<13' -O b filtering/snps.bcf > filtering/snps.SAMP13.bcf
```

#### Genotype-based filtering
Filtering using the genotype fields can allow for some more precise filtering. For example, in cases of high depth heterogeneity across samples, it may be better to filter out by median genotype depth than bytotal depth across all samples. An example removing all SNPs where the mean genotype depth is below 5:
```bash
bcftools view -e 'AVG(FMT/DP)<5' -O b filtering/snps.bcf > filtering/snps.MEANGTDP5.bcf
```
Sometimes it is reasonable to ignore genotype calls based on few reads. The following command remove all genotype calls (i.e. genotypes are substited by dots) based on less than 3 reads:
```bash
bcftools filter -S . -e 'FMT/DP<3' -O b filtering/snps.bcf > filtering/snps.NOGT3.bcf
```
#### Combining multiple filters 
Multiple filters can be combined in a single command using or piping several ones. For example, we can combine a few of the filters we have used above:
```bash
bcftools filter -S . -e 'FMT/DP<3' filtering/snps.bcf | \
bcftools view -e 'AVG(FMT/DP)<5 || MAF<0.05 || MQ<20 || AN/2<13' -O b > filtering/snps.NOGT3.MEANGTDP5.MAF005.MQ20.SAMP13.bcf
```

It is important to be careful with the order of the filters, as different combinations can result in different end results.

[Back to TOC](index.md)
