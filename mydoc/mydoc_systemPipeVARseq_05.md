---
title: Variant calling
keywords: 
last_updated: Wed Jun  1 20:48:34 2016
---

The following performs variant calling with `GATK`, `BCFtools` and `VariantTools` 
in parallel mode on a compute cluster (McKenna et al., 2010; Li , 2011). If a cluster is not
available, the `runCommandline` function can be used to run the variant calling with `GATK` 
and `BCFtools` for each sample sequentially on a single machine, or `callVariants` in case 
of `VariantTools`. Typically, the user would choose here only one variant caller rather
than running several ones.

## Variant calling with `GATK`

The following creates in the inital step a new `targets` file
(`targets_bam.txt`). The first column of this file gives the paths to
the BAM files created in the alignment step. The new targets file and the
parameter file `gatk.param` are used to create a new `SYSargs`
instance for running GATK. Since GATK involves many processing steps, it is
executed by a bash script `gatk_run.sh` where the user can specify the
detailed run parameters. All three files are expected to be located in the
current working directory. Samples files for `gatk.param` and
`gatk_run.sh` are available in the `param` subdirectory
provided by `systemPipeRdata`.


{% highlight r %}
moduleload("picard/1.130"); moduleload("samtools/1.3")
system("picard CreateSequenceDictionary R=./data/tair10.fasta O=./data/tair10.dict")
system("samtools faidx data/tair10.fasta")
args <- systemArgs(sysma="param/gatk.param", mytargets="targets_bam.txt")
resources <- list(walltime="20:00:00", nodes=paste0("1:ppn=", 1), memory="10gb")
reg <- clusterRun(args, conffile=".BatchJobs.R", template="torque.tmpl", Njobs=18, runid="01",
                  resourceList=resources)
waitForJobs(reg)
# unlink(outfile1(args), recursive = TRUE, force = TRUE)
writeTargetsout(x=args, file="targets_gatk.txt", overwrite=TRUE)
{% endhighlight %}

## Variant calling with `BCFtools`

The following runs the variant calling with `BCFtools`. This step requires
in the current working directory the parameter file `sambcf.param` and the bash script 
`sambcf_run.sh`.


{% highlight r %}
args <- systemArgs(sysma="param/sambcf.param", mytargets="targets_bam.txt")
resources <- list(walltime="20:00:00", nodes=paste0("1:ppn=", 1), memory="10gb")
reg <- clusterRun(args, conffile=".BatchJobs.R", template="torque.tmpl", Njobs=18, runid="01",
                  resourceList=resources)
waitForJobs(reg)
# unlink(outfile1(args), recursive = TRUE, force = TRUE)
writeTargetsout(x=args, file="targets_sambcf.txt", overwrite=TRUE)
{% endhighlight %}

## Variant calling with `VariantTools`  


{% highlight r %}
library(gmapR); library(BiocParallel); library(BatchJobs)
args <- systemArgs(sysma="param/vartools.param", mytargets="targets_gsnap_bam.txt")
f <- function(x) {
    library(VariantTools); library(gmapR); library(systemPipeR)
    args <- systemArgs(sysma="param/vartools.param", mytargets="targets_gsnap_bam.txt")
    gmapGenome <- GmapGenome(systemPipeR::reference(args), directory="data", name="gmap_tair10chr", create=FALSE)
    tally.param <- TallyVariantsParam(gmapGenome, high_base_quality = 23L, indels = TRUE)
    bfl <- BamFileList(infile1(args)[x], index=character())
    var <- callVariants(bfl[[1]], tally.param)
    sampleNames(var) <- names(bfl)
    writeVcf(asVCF(var), outfile1(args)[x], index = TRUE)
}
funs <- makeClusterFunctionsTorque("torque.tmpl")
param <- BatchJobsParam(length(args), resources=list(walltime="20:00:00", nodes="1:ppn=1", memory="6gb"), cluster.functions=funs)
register(param)
d <- bplapply(seq(along=args), f)
writeTargetsout(x=args, file="targets_vartools.txt", overwrite=TRUE)
{% endhighlight %}

## Inspect VCF file 

VCF files can be imported into R with the `readVcf` function. Both `VCF` and `VRanges` objects provide
convenient data structure for working with variant data (_e.g._ SNP quality filtering). 


{% highlight r %}
library(VariantAnnotation)
args <- systemArgs(sysma="param/filter_gatk.param", mytargets="targets_gatk.txt")
vcf <- readVcf(infile1(args)[1], "A. thaliana")
vcf
vr <- as(vcf, "VRanges")
vr
{% endhighlight %}

