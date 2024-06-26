processing\_raw\_fluidigm\_data
================
Allie Byrne

The purpose of this document is to show you how to process raw
sequencing data produced by running Bd samples on the Fludigim Access
array system. You will go from raw sequencing data (typically in the
form of XXX\_R1.fastq.gz and XXX\_R2.fastq.gz) to sequence (consensus,
ambiguities, occurrence) and variant data files. These files can then be
used to create trees, PCAs, etc.

Each chunk in this file indicates whether it is run in R or in bash
(i.e., terminal in mac).

In addition to some analyses in this code, to fully process the data you
will also need to download and install the following programs:

[flash](https://ccb.jhu.edu/software/FLASH/#:~:text=FLASH%20is%20designed%20to%20merge,to%20merge%20RNA%2Dseq%20data.)

[bwa](https://github.com/lh3/bwa)

[picard](https://github.com/broadinstitute/picard/releases/tag/3.1.1)

[samtools](https://www.htslib.org/download/)

[freebayes](https://github.com/freebayes/freebayes)

First, install and load the R packages we need.

\#Producing consensus, ambiguities, and occurrence sequences

First, let’s begin the pipeline to produce consensus, ambiguities, and
occurrence sequences from our raw data. For this we will use the program
flash2 to combine read1 and read2 into paired reads. In my case I
downloaded flash and the executable file is within my working directory
in the “FLASH-1.2.11” folder.

``` r
#fill in the prefix of your sequence files (whatever comes before _R1.fastq.gz)
basename <- "Bd_Fluidigm_V3_7_20_20"

R1 <- paste(basename,"_R1.fastq.gz",sep="")
R2 <- paste(basename,"_R2.fastq.gz",sep="")
f_output <- paste("flash_output/",basename,sep="")

#create directory for output
dir.create("flash_output")

#create the flash command
call <- paste("./FLASH-1.2.11/flash --max-overlap=600 --allow-outies -x 0.25 -z -o",f_output,R1,R2,sep=" ")
  
#run the command
system(call)
```

This will output the joined reads in the “flash\_output” folder with the
prefix as the basename.

Now we can run reduce\_amplicons which is a script originally written by
[Matt
Settles](https://github.com/msettles/dbcAmplicons/blob/master/scripts/R/reduce_amplicons.R).
This script takes your joined reads and produces three different files:
consensus (most common sequence for each sample/amplicon), ambiguities
(one sequence per sample/amplicon with multiple alleles coded with IUPAC
ambiguity codes), and occurrence (all individual sequences for each
sample/allele). These will be output into a folder named as the basename
variable.

``` r
#establish some parameters for running the programs
arguments <- list(options = list(program="consensus,ambiguities,occurrence",min_seq=5,min_freq=0.05,trimOne=0,trimTwo=0,reuse=TRUE,output=basename,procs=8),args=basename)

opt <- arguments$options
output = opt$output

dir.create(output)

avail_functions <- c("consensus","ambiguities","occurrence")
program_list = unlist(strsplit(opt$program,split=","))

min_seq = opt$min_seq
min_freq = opt$min_freq
trimOne = opt$trimOne
trimTwo = opt$trimTwo

### read in input files made with flash    
fq <- readFastq(paste(f_output,".extendedFrags.fastq.gz",sep=""))

fq_r1 <- readFastq(paste(f_output,".notCombined_1.fastq.gz",sep=""))

fq_r2 <- readFastq(paste(f_output,".notCombined_2.fastq.gz",sep=""))


## prepareCore
##  Set up the numer of processors to use
## 
## Parameters
##  opt_procs: processors given on the option line
##  samples: number of samples
##  targets: number of targets
"prepareCore" <- function(opt_procs){
    # if opt_procs set to 0 then expand to samples by targets
    if( opt_procs == 0 ) opt_procs <- detectCores()
    if (opt_procs > detectCores()){
        warning(paste("number of processors specified exceeds the number of available processors '",detectCores(),"'",sep=" "))
        opt_procs <- detectCores()
    }
    write(paste("Using",opt_procs,"processors",sep=" "),stdout())
    return(opt_procs)
}

procs <- prepareCore(opt$procs)

### process merged files
nms <- as.character(ShortRead::id(fq))
id <- sapply(strsplit(sapply(strsplit(nms,split=" "),"[[",2L),split=":"),"[[",4L)
primer <- sapply(strsplit(sapply(strsplit(nms,split=" "),"[[",2L),split=":"),"[[",5L)
occurrence = table(id,primer)

### process unmerged files
nms_p <- as.character(ShortRead::id(fq_r1))
id_p <- sapply(strsplit(sapply(strsplit(nms_p,split=" "),"[[",2L),split=":"),"[[",4L)
primer_p <- sapply(strsplit(sapply(strsplit(nms_p,split=" "),"[[",2L),split=":"),"[[",5L)
occurrence_p = table(id_p,primer_p)

uprimer <- sort(unique(c(primer,primer_p)))
uid <- sort(unique(c(id,id_p)))
```

This chuck contains the analysis functions.

``` r
### Analysis Functions BEGIN

## consensus
##  compute strict consensus sequence for each sample/amplicon
## 
## Parameters
##  name: amplicon name
##  min_freq: minimum frequence to accept variant
##  min_seq: minimum number of sequences to accept variant
"consensus" <- function(name,min_freq,min_seq) {
#    cat(name,", ")
    seqs <- DNAStringSet()
    maxsize = 0    
    if (name %in% names(splitfq)){
        tmp <- splitfq[[name]]
        wtmp = split(tmp,width(tmp))
        wtmp = wtmp[[which.max(sapply(wtmp,length))]]
        if(length(wtmp) > maxsize & length(wtmp) > min_seq & (length(wtmp)/counts[[name]]) > min_freq){
            abc<-alphabetByCycle(wtmp)
            consensus <- paste(rownames(abc)[apply(abc,2,which.max)],collapse="")
            error_rate=sum(1-apply(sweep(abc,MARGIN=2,STATS=colSums(abc),FUN="/"),2,max))/nchar(consensus)
            ssingle_str <- DNAStringSet(consensus)
            names(ssingle_str) <- paste(name,"merged", seq.int(1,length(ssingle_str)), length(wtmp), counts[[name]], round(length(wtmp)/counts[[name]],3),round(error_rate,3),sep="|")
            maxsize = length(wtmp)
            seqs <- ssingle_str
        }
    }
    if (name %in% names(splitfq_p)){
        tmp <- splitfq_p[[name]]
        wtmp = split(tmp,width(tmp))
        wtmp = wtmp[[which.max(sapply(wtmp,length))]]
        if(length(wtmp) > maxsize & length(wtmp) > min_seq & (length(wtmp)/counts[[name]]) > min_freq){
            abc<-alphabetByCycle(wtmp)
            consensus <- paste(rownames(abc)[apply(abc,2,which.max)],collapse="")
            error_rate=sum(1-apply(sweep(abc,MARGIN=2,STATS=colSums(abc),FUN="/"),2,max))/(nchar(consensus)-5) ## - 5 for Ns
            spaired_str <- lapply(strsplit(consensus,split=".....",fixed = TRUE),DNAStringSet)
            spaired_str <- do.call("c",spaired_str)
            names(spaired_str) <- paste(name,c("read1","read2"),rep(seq.int(1,length(spaired_str)/2),each=2),length(wtmp), counts[[name]], round(length(wtmp)/counts[[name]],3),round(error_rate,3),sep="|")            
            maxsize = length(wtmp)
            seqs <- spaired_str
        }
    }
    seqs
}

######################################################################
## ambiguities
##  compute ambiguity sequence for each sample/amplicon
## 
## Parameters
##  name: amplicon name
##  min_freq: minimum frequence to accept variant
##  min_seq: minimum number of sequences to accept variant

"ambiguities" <- function(name,min_freq,min_seq) {
#    cat(name,", ")
    seqs <- DNAStringSet()
    maxsize = 0    
    if (name %in% names(splitfq)){
        tmp <- splitfq[[name]]
        wtmp = split(tmp,width(tmp))
        wtmp = wtmp[[which.max(sapply(wtmp,length))]]
        if(length(wtmp) > maxsize & length(wtmp) > min_seq & (length(wtmp)/counts[[name]]) > min_freq){
            abc<-alphabetByCycle(wtmp)
            abcf<-sweep(abc,MARGIN=2,STATS=colSums(abc),FUN="/")
            abcA = abc
            abcA[abc < min_seq | abcf < min_freq] = 0
            consensus <- names(IUPAC_CODE_MAP)[match(apply(abcA,2,function(x) paste(rownames(abcA)[x>0],collapse="")),IUPAC_CODE_MAP)]
            consensus[is.na(consensus)] = "N"
            amb_bases <- sum(consensus %in% names(IUPAC_CODE_MAP[-c(1:4)]))
            consensus = paste(consensus,collapse="")
            error_rate=sum(1-apply(sweep(abc,MARGIN=2,STATS=colSums(abc),FUN="/"),2,max))/nchar(consensus)
            ssingle_str <- DNAStringSet(consensus)
            names(ssingle_str) <- paste(name,"merged", seq.int(1,length(ssingle_str)), length(wtmp), counts[[name]], round(length(wtmp)/counts[[name]],3), round(error_rate,3), amb_bases,sep="|")
            maxsize = length(wtmp)
            seqs <- ssingle_str
        }
    }
    if (name %in% names(splitfq_p)){
        tmp <- splitfq_p[[name]]
        wtmp = split(tmp,width(tmp))
        wtmp = wtmp[[which.max(sapply(wtmp,length))]]
        if(length(wtmp) > maxsize & length(wtmp) > min_seq & (length(wtmp)/counts[[name]]) > min_freq){
            abc<-alphabetByCycle(wtmp)
            abcf<-sweep(abc,MARGIN=2,STATS=colSums(abc),FUN="/")
            abcA = abc
            abcA[abc < min_seq | abcf < min_freq] = 0
            consensus <- names(IUPAC_CODE_MAP)[match(apply(abcA,2,function(x) paste(rownames(abcA)[x>0],collapse="")),IUPAC_CODE_MAP)]
            consensus[is.na(consensus)] = "N"
            consensus[which(abc['.',] > (length(wtmp)*0.9))] =  "."  ## at least 90% of reads must have '.' at the join location
            amb_bases <- sum(consensus %in% names(IUPAC_CODE_MAP[-c(1:4)])) ## - 5 for Ns
            consensus = paste(consensus,collapse="")
            error_rate=sum(1-apply(sweep(abc,MARGIN=2,STATS=colSums(abc),FUN="/"),2,max))/(nchar(consensus)-5) ## - 5 for Ns
            spaired_str <- lapply(strsplit(consensus,split="\\.+",fixed = FALSE),DNAStringSet)
            spaired_str <- do.call("c",spaired_str)
            names(spaired_str) <- paste(name,c("read1","read2"),rep(seq.int(1,length(spaired_str)/2),each=2),length(wtmp), counts[[name]], round(length(wtmp)/counts[[name]],3),round(error_rate,3),amb_bases,sep="|")            
            maxsize = length(wtmp)
            seqs <- spaired_str
        }
    }
    seqs
}

######################################################################
## occurrence
##  output all sequences, for each sample/amplicon, which meet the min_freq and min_seq criteria
## 
## Parameters
##  name: amplicon name
##  min_freq: minimum frequence to accept variant
##  min_seq: minimum number of sequences to accept variant
"occurrence" <- function(name,min_freq,min_seq) {
#    cat(name,", ")
    seqs <- DNAStringSet()
    
    ssingle = c(); spaired=c();
    if (name %in% names(splitfq)){
        ssingle <- table(splitfq[[name]])[table(splitfq[[name]])/counts[[name]] >=min_freq & table(splitfq[[name]])>= min_seq]
        if (length(ssingle) > 0){
            ssingle_str <- DNAStringSet(names(ssingle))
            names(ssingle_str) <- paste(name,"merged", seq.int(1,length(ssingle_str)), ssingle, counts[[name]], signif(ssingle/counts[[name]],3),sep="|")
            seqs <- c(seqs,ssingle_str)
        }
    }
    if (name %in% names(splitfq_p)){
        spaired <- table(splitfq_p[[name]])[table(splitfq_p[[name]])/counts[[name]] >=min_freq & table(splitfq_p[[name]])>= min_seq]
        if (length(spaired) > 0){
            spaired_str <- lapply(strsplit(names(spaired),split=".....",fixed = TRUE),DNAStringSet)
            spaired_str <- do.call("c",spaired_str)
            names(spaired_str) <- paste(name,c("read1","read2"),rep(seq.int(1,length(spaired_str)/2),each=2),rep(spaired,each=2), counts[[name]], rep(signif(spaired/counts[[name]],3),each=2),sep="|")
            seqs <- c(seqs,spaired_str)
        }
    }
    seqs
}


### Analysis Functions END
```

This chunk merges the reads to prepare for analysis

``` r
## merge
merged <- DNAStringSet(paste(as.character(sread(fq_r1)),".....",as.character(reverseComplement(sread(fq_r2))),sep=""))
names(merged) <- nms_p

single <- sread(fq)
names(single) <- nms

joined = paste(id,primer,sep=":")
joined_p = paste(id_p,primer_p,sep=":")
counts <- table(c(joined,joined_p))

anames <- unique(c(joined,joined_p))
splitfq <- split(single,paste(id,primer,sep=":"))
splitfq_p <- split(merged,paste(id_p,primer_p,sep=":"))
```

This chunk runs the programs. WARNING it takes awhile to run. Could run
overnight.

``` r
#run the programs

sapply(program_list,function(program){
    write(paste("Running program:",program),stdout())
    freq_seqs <- mclapply(anames, function(nms) do.call(program,list(name=nms,min_freq=min_freq,min_seq=min_seq)), mc.cores = procs)
    
    redo <- sapply(freq_seqs,function(x) class(x) != "DNAStringSet")
    if (sum(redo) > 0){
        write(paste("There were",sum(redo),"failed amplicon analysis, trying to rerun"),stdout())
#        stop(paste("There were",sum(redo),"failed amplicon analysis, trying to rerun"))
    } 
    retry = 0
    while (any(redo) & retry < 5){
        freq_seqs[redo] <- mclapply(anames[redo], do.call(program,list(name=nms,freq_min=min_freq,min_seq=min_seq)), mc.cores = procs)
        redo <- sapply(freq_seqs,function(x) class(x) != "DNAStringSet")
        retry = retry+1
        write(paste("Retry number", retry,sep=" "),stdout())
    }
    if (any(redo)) stop("Failed to complete all amplicons, try reducing the number of processors.")
    write(paste("Finished analyzing amplicons"),stdout())
        

#    names(freq_seqs) <- anames
#    result_seqs <- sapply(freq_seqs,length)
#    tt <- DNAStringSet(unlist(unname(sapply(freq_seqs,as.character))))
    tt <- do.call('c',unname(freq_seqs))  
    onms <- names(tt)
    first <- sapply(strsplit(onms,split="|",fixed=TRUE),"[[",1L)
    oprimer <- sapply(strsplit(first,split=":"),function(x) tail(x,1))
    oid <- apply(cbind(paste(":",oprimer,sep=""),first),1,function(x) sub(x[1],"",x[2]))
    
    second <- sapply(strsplit(onms,split="|",fixed=TRUE),"[[",2L)

    write(paste("Writing Reads"),stdout())
    writeXStringSet(tt,file.path(output,paste(program,"reduced.fasta",sep=".")))

    dir.create(file.path(output,paste(program,"split_samples",sep=".")))
    split_tt <- split(tt,oid)
    mclapply(names(split_tt), function(x){
        writeXStringSet(split_tt[[x]],file.path(output,paste(program,"split_samples",sep="."),paste("Sample",x,"fasta",sep=".")))
    }, mc.cores = procs)
    dir.create(file.path(output,paste(program,"split_amplicon",sep=".")))
    split_tt <- split(tt,paste(oprimer,second,sep="."))
    mclapply(names(split_tt), function(x){
        writeXStringSet(split_tt[[x]],file.path(output,paste(program,"split_amplicon",sep="."),paste("Amplicon",x,"fasta",sep=".")))
    }, mc.cores = procs)

    write(paste("Producing final images"),stdout())
    #### PLOTTING RESULTS    
    png(file.path(output,paste(program,"freq_read_counts.png",sep=".")),height=8,width=10.5,units="in",res=300)
    hist(as.numeric(sapply(strsplit(onms,split="|",fixed=TRUE),"[[",5L)),breaks=200,main=paste("histogram of amplicon read counts\n",program," trimR1:",trimOne," trimR2:",trimTwo,"\n",sep=""),xlab="frequency")
    invisible(dev.off())
    png(file.path(output,paste(program,"freq_most_occuring.png",sep=".")),height=8,width=10.5,units="in",res=300)
    hist(as.numeric(sapply(strsplit(onms,split="|",fixed=TRUE),"[[",6L)),breaks=200,main=paste("histogram of chosen amplicon frequency\n",program," trimR1:",trimOne," trimR2:",trimTwo,"\n",sep=""),xlab="read counts")
    invisible(dev.off())
    if (program %in% c("consensus","ambiguities")){
        png(file.path(output,paste(program,"error_rate.png",sep=".")),height=8,width=10.5,units="in",res=300)
        hist(as.numeric(sapply(strsplit(onms,split="|",fixed=TRUE),"[[",7L)),breaks=200,main=paste("histogram of error rate in amplicons\n",program," trimR1:",trimOne," trimR2:",trimTwo,"\n",sep=""),xlab="error rate")
        invisible(dev.off())
        if (program %in% c("ambiguities")){
            png(file.path(output,paste(program,"ambiguities.png",sep=".")),height=8,width=10.5,units="in",res=300)
            hist(as.numeric(sapply(strsplit(onms,split="|",fixed=TRUE),"[[",8L)),breaks=200,main=paste("histogram of number of ambiguites in amplicons\n",program," trimR1:",trimOne," trimR2:",trimTwo,"\n",sep=""),xlab="number of ambiguity bases")
            invisible(dev.off())
        }
    }
    
    occurrence_a <- table(oid,oprimer)
    uoccurrence_a <- matrix(0,nrow=length(uid),ncol=length(uprimer))
    rownames(uoccurrence_a) <- uid
    colnames(uoccurrence_a) <- uprimer
    
    uoccurrence_a[rownames(occurrence_a),colnames(occurrence_a)] <- occurrence_a
    
    mratio <- melt(uoccurrence_a)
    colnames(mratio) <- c("SampleID","PrimerID","value")
    mratio$value = as.factor(mratio$value)
    
    jRdGyPalette <- brewer.pal(n = 4, "Set1")
    paletteSize <- 4
    
    p <- ggplot(mratio, aes(x = PrimerID, y = SampleID, fill = value)) +
        theme(axis.text.x = element_text(angle = 90, hjust = 1, vjust = 0.5,size=8), 
              axis.text.y = if(length(unique(mratio$SampleID))> 96) { element_blank() }else{element_text(size=8)}) +
        geom_tile() +
        scale_fill_brewer(palette="Set1") +
        #     scale_fill_gradient2(low = jRdGyPalette[1],
        #                          mid = jRdGyPalette[paletteSize/2],
        #                          high = jRdGyPalette[paletteSize],
        #                          midpoint = 0,
        #                          name = "Amplicon Count") +
        labs(title=paste("Resulting number of amplicons\n",program," trimR1:",trimOne," trimR2:",trimTwo,"\n",sep=""))
    
    png(file.path(output,paste(program,"Amplicons","Per","Sample","png",sep=".")),width=8,height=10.5,units="in",res=300)
    print(p)
    invisible(dev.off())
})

write("Finished",stdout())
```

Now we have our consensus, ambiguities, and occurrence files for
downstream analysis. \#Calling Variants

Let’s continue to process the data to call variants.

First we will split the FASTQ file produced by flash into separate
sample FASTQ files.

``` r
split_tt <- split(fq,id)
dir.create("split_samples")
#procs = 2 not supported on windows
mclapply(names(split_tt), function(x){
 writeFastq(split_tt[[x]],file.path(paste("split_samples/",sep="."),paste("Sample",x,"fastq.gz",sep=".")))
})
```

With seperate fastqs index our reference and align/create bams for each
sample against reference.

*run this on your own time when you have programs ready to go or on a
cluster*

``` r
#list split files
fqFiles <- list.files("./split_samples")

#to make the bwa commands to put in a slurm or run on command line
sample_names <-  sapply(fqFiles, function(x) unlist(strsplit(x, split="Sample."))[2])
sample_names <- sapply(sample_names , function(x) unlist(strsplit(x, split=".fastq.gz"))[1])

for (i in 1:length(fqFiles)){
  if (i == 1){
list1 <-paste("bwa mem -M -t 9 -R '@RG\tID:group",i,"\tSM:",sample_names[i],"\tPL:illumina\tLB:",sample_names[i],"' ../Bd_Fl_ref_amplicon_seqs_noprimer.fasta Sample.", sample_names[i],".fastq.gz > ", sample_names[i], "_aligned_reads.sam", sep="")
} else {
  newobj <-paste("bwa mem -M -t 9 -R '@RG\tID:group",i,"\tSM:",sample_names[i],"\tPL:illumina\tLB:",sample_names[i],"' ../Bd_Fl_ref_amplicon_seqs_noprimer.fasta Sample.", sample_names[i],".fastq.gz > ", sample_names[i], "_aligned_reads.sam", sep="")
  list1 <- append(list1, newobj)
}
}

write.csv(list1, file="db_bwa_commands.csv")

#to make the picard sort commands to put in a slurm

for (i in 1:length(fqFiles)){
  if (i == 1){
list1 <-paste("java -Xmx2g -jar /clusterfs/vector/home/groups/software/sl-7.x86_64/modules/picard/2.9.0/lib/picard.jar SortSam INPUT=",sample_names[i],"_aligned_reads.sam OUTPUT=", sample_names[i], "_sorted.bam SORT_ORDER=coordinate", sep="")
} else {
  newobj <- paste("java -Xmx2g -jar /clusterfs/vector/home/groups/software/sl-7.x86_64/modules/picard/2.9.0/lib/picard.jar SortSam INPUT=",sample_names[i],"_aligned_reads.sam OUTPUT=", sample_names[i], "_sorted.bam SORT_ORDER=coordinate", sep="")
  list1 <- append(list1, newobj)
}
}

write.csv(list1, file="db_picard_sort_commands.csv")
```

clean up anything in bams folder that doesn’t end with sort.bam - BE
CAREFUL \!\!\!\!\!\!

``` bash
cd bams
ls | grep -v sort.bam
ls | grep -v sort.bam | xargs rm
```

index bams with samtools

``` bash
#do this on the cluster

##!/usr/bin/env bash
#for i in *.bam;
#do
#   samtools index $i
#done;

cd bams > samtool_index.sh
```

Get active region list

``` r
xx = readDNAStringSet("Bd_Fl_ref_amplicon_seqs_noprimer.fasta")
write.table(paste(names(xx),paste(1,width(xx),sep="-"),sep=":"),"ref_amp_activeRegions.txt",quote=F,row.names=F,col.names=F)
write.table(paste(names(xx),paste(1,width(xx),sep=" "),sep=" "),
            "ref_amp_activeRegions.bed",quote=F,row.names=F,col.names=F)
```

run Freebayes - beforehand make a list of all your bams as a text file
to feed in each bam

``` bash
#makes a list of all the bam files
for (i in 1:length(fqFiles)){
  if (i == 1){
list1 <-paste("/clusterfs/vector/scratch/allie128/Bd_panama_fl/raw_fastas/",sample_names[i],"_sorted.bam",sep="")
} else {
  newobj <- paste("/clusterfs/vector/scratch/allie128/Bd_panama_fl/raw_fastas/",sample_names[i],"_sorted.bam",sep="")
  list1 <- append(list1, newobj)
}
}

write.csv(list1, file="db_bamlist.csv")


#run this in the command line
freebayes -f freebayes/ref_amp.fa -L bamlist.txt -t freebayes/ref_amp_activeRegions.bed -0 -X --haplotype-length 0 -kwVa --no-mnps --min-coverage 10 --no-complex --use-best-n-alleles 4 --min-alternate-count 5 --min-alternate-fraction 0.3 > freebayes/ltreb_freebae_newref.vcf
```

Now you have a .vcf file you can use to make PCAs and do other analyses
on variants.

Note that you may need to do additional variant and/or individual
filtration depending on your samples.

Now you can use the sequences and vcf file you processed and follow the
following tutorials from previous workshops:

[RIBBiTR workshop 2022](https://github.com/allie128/RIBBiTR_workshop)

[RIBBiTR workshop 2023](https://github.com/allie128/workshop_Costa_Rica)
