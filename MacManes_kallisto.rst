================================================ 
Kallisto and Sleuth
================================================

Kallisto: https://liorpachter.wordpress.com/2015/05/10/near-optimal-rna-seq-quantification-with-kallisto/ and http://pachterlab.github.io/kallisto/starting.html

Sleuth: https://liorpachter.wordpress.com/2015/08/17/a-sleuth-for-rna-seq/

--------------

Step 1: Launch and AMI. For this exercise, we will use a **c4.2xlarge** We need to mount ``snap-9b14b2f3`` This has to be minimum of 100Gb and mounted at ``/dev/xvdf``


::

    ssh -i ~/Downloads/?????.pem ubuntu@ec2-???-???-???-???.compute-1.amazonaws.com

--------------

**Update Software**

::

    sudo apt-get update

--------------

**Install updates**

::

    sudo apt-get -y upgrade

--------------

**Install other software** Note that you can install a large amount of software from the Ubuntu "App Store" using a single command. Some of this software we will not use for this tutorial, but...

::

  sudo apt-get -y install build-essential tmux git gcc make cmake g++ python-dev libhdf5-dev \
  unzip default-jre libcurl4-openssl-dev libxml2-dev libssl-dev zlib1g-dev python-pip samtools bowtie ncbi-blast+

--------------


**Mount hard drive** The EBS volume we asked for is not automatically mounted - we need to do that. 

::

    sudo mount /dev/xvdf /mnt  
    sudo chown -R ubuntu:ubuntu /mnt  
    df -h

--------------

**INSTALL KALLISTO**

::

  cd $HOME
  git clone https://github.com/pachterlab/kallisto.git
  cd kallisto/ && mkdir build
  cd build
  cmake ..
  make
  sudo make install


----------------


**RUN KALLISTO**: Kallisto can use a genome, but transcriptome is better for this work. See https://twitter.com/PeroMHC/status/633621603759165440 and discussion. 

::

  mkdir -p /mnt/kallisto/results/ && cd /mnt/kallisto
  tmux new -s kallisto

  curl -LO ftp://ftp.ensembl.org/pub/release-75/fasta/drosophila_melanogaster/cdna/Drosophila_melanogaster.BDGP5.75.cdna.abinitio.fa.gz

  kallisto index -i dros Drosophila_melanogaster.BDGP5.75.cdna.abinitio.fa.gz
  

  samples[1]=ORE_wt_rep1
  samples[2]=ORE_wt_rep2
  samples[3]=ORE_sdE3_rep1
  samples[4]=ORE_sdE3_rep2
  samples[5]=SAM_wt_rep1
  samples[6]=SAM_wt_rep2
  samples[7]=SAM_sdE3_rep1
  samples[8]=SAM_sdE3_rep2
  samples[9]=HYB_wt_rep1
  samples[10]=HYB_wt_rep2
  samples[11]=HYB_sdE3_rep1
  samples[12]=HYB_sdE3_rep2
 

  for i in 1 2 3 4 5 6 7 8 9 10 11 12
  do
      sample=${samples[${i}]}
      mkdir -p results/${sample}/kallisto
      kallisto quant -i dros --threads=4 --bootstrap-samples=100 \
      --output-dir=results/${sample}/kallisto \
      /mnt/drosophila/cleaned_reads/${sample}_1.fastq \
      /mnt/drosophila/cleaned_reads/${sample}_2.fastq
  done

**Download data from EC2 to laptop**: Make directory on your laptop.. ``mkdir ~/Downloads/sleuth/`` for the MAC people. 

::

  scp -r -i your.pem ubuntu@ec2-xx-x-xxx-xxx.compute-1.amazonaws.com:/mnt/kallisto/results ~/Downloads/sleuth/


**LAUNCH SLEUTH**

::
  
  #to Launch into RStudio
  source("http://bioconductor.org/biocLite.R")
  biocLite("rhdf5")
  install.packages('devtools')
  devtools::install_github('pachterlab/sleuth')
  library("sleuth")

  #Change project dir in R

  base_dir <- "~/Downloads/sleuth"
  sample_id <- dir(file.path(base_dir,"results"))
  kal_dirs <- sapply(sample_id, function(id) file.path(base_dir, "results", id, "kallisto"))
  kal_dirs
  s2c <- read.table(file.path(base_dir,"experiment.info"), header = TRUE, stringsAsFactors=FALSE)
  s2c <- dplyr::select(s2c, sample = name, reads, condition, wt)
  s2c
  so <- sleuth_prep(kal_dirs, s2c, ~ condition)
  so <- sleuth_fit(so)
  so <- sleuth_test(so, which_beta = 'conditionSAM')
  sleuth_live(so)