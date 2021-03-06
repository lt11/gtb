Ground Truth Builder
================
Lorenzo Tattini
20/04/2022

## Preamble and Definitions

A ground truth is what we need to evaluate the performance of
graph-based pipelines.

## Dependencies

-   bcftools
-   bgzip (1.4)
-   tabix
-   mummer
-   data.table

## Quick Run

This is a list of the things that must be edited to have a quick run
working properly:

-   the name of the “run” folder (e.g. gt-1)
-   check all chunks with “eval = FALSE”
-   anything else?

## Brief Description of the Contents

Given a set of assemblies, this markdown performs the following tasks:

-   running nucmer using any assembly against the reference genome
-   transforming the results in a vcf file
-   checking for loci which not genotyped because they bear the REF
    allele (rather than being actually missing because no alignment in
    that locus was found by nucmer)

## TODOs

Update bgzip!

-   yes, update bgzip
-   check if the “reverse alignments” behaviour of nucmer may cause
    problems
-   add a description of the folders

### Folders Tree

All folders are generated automatically, except “mrk” and “aux”. The
tree is:

-   aux
-   genomes
    -   collinear
    -   rearranged
    -   reference
-   mrk
-   ms-vcf
-   nuc-aln
-   os-vcf

## Analysis

First, we need to merge the nuclear genome with mitochondrial genome.
This chunk creates the input folder (genomes).

``` bash
## settings -------------------------------------------------------------------

### strains selection
ref_assembly="SGDref-hc-genome.fa.gz"
collinear_assembly=('SK1-hc-genome.fa.gz' 'YPS128-hc-genome.fa.gz' \
'DBVPG6044-hc-genome.fa.gz' 'S288C-hc-genome.fa.gz' \
'DBVPG6765-hc-genome.fa.gz' 'Y12-hc-genome.fa.gz')
rearranged_assembly=('UWOPS034614-hc-genome.fa.gz')

### fasta repository
fasta_rep="${HOME}/data/nano-assemblies-latest"

### base dir
dir_base="${HOME}/prog/ground-truth/gt-1"

## clmnt ----------------------------------------------------------------------

### clean and make folders
assemblies_dir="${dir_base}/genomes"
if [[ -d "${assemblies_dir}" ]]; then 
  rm -rf "${assemblies_dir}"
  mkdir -p "${assemblies_dir}"
fi
collinear_dir="${dir_base}/genomes/collinear"
ref_dir="${dir_base}/genomes/reference"
rearranged_dir="${dir_base}/genomes/rearranged"
mkdir -p "${collinear_dir}"
mkdir -p "${ref_dir}"
mkdir -p "${rearranged_dir}"

### process the collinear genomes
cd "${collinear_dir}"
for fasta_file in "${collinear_assembly[@]}"; do
  mito_id=$(basename "${fasta_file}" | cut -f 1 -d "-") # e.g. ADE
  mito_name="${mito_id}-mt-genome.fa.gz"
  mito_path=$(find "${fasta_rep}" -name "${mito_name}")
  fasta_path=$(find "${fasta_rep}" -name "${fasta_file}")
  asse_type=$(basename "${fasta_file}" | cut -f 2 -d "-") # e.g. hc or h1
  if [[ -f "${mito_path}" ]] && [[ "${asse_type}" != "h2" ]]; then
    gunzip -c "${fasta_path}" "${mito_path}" > ./nuc-temp.fa
  else
    gunzip -c "${fasta_path}" > ./nuc-temp.fa
  fi
  strain_id=$(basename "${fasta_file}" | cut -f 1,2 -d "-") # e.g. ADE-hc
  mv -f ./nuc-temp.fa "${strain_id}-genome.fa"
done

### process the rearranged genomes
cd "${rearranged_dir}"
for fasta_file in "${rearranged_assembly[@]}"; do
  mito_id=$(basename "${fasta_file}" | cut -f 1 -d "-") # e.g. ADE
  mito_name="${mito_id}-mt-genome.fa.gz"
  mito_path=$(find "${fasta_rep}" -name "${mito_name}")
  fasta_path=$(find "${fasta_rep}" -name "${fasta_file}")
  asse_type=$(basename "${fasta_file}" | cut -f 2 -d "-") # e.g. hc or h1
  if [[ -f "${mito_path}" ]] && [[ "${asse_type}" != "h2" ]]; then
    gunzip -c "${fasta_path}" "${mito_path}" > ./nuc-temp.fa
  else
    gunzip -c "${fasta_path}" > ./nuc-temp.fa
  fi
  strain_id=$(basename "${fasta_file}" | cut -f 1,2 -d "-") # e.g. ADE-hc
  mv -f ./nuc-temp.fa "${strain_id}-genome.fa"
done

### process the reference
cd "${ref_dir}"
for fasta_file in "${ref_assembly[@]}"; do
  mito_id=$(basename "${fasta_file}" | cut -f 1 -d "-") # e.g. ADE
  mito_name="${mito_id}-mt-genome.fa.gz"
  mito_path=$(find "${fasta_rep}" -name "${mito_name}")
  fasta_path=$(find "${fasta_rep}" -name "${fasta_file}")
  asse_type=$(basename "${fasta_file}" | cut -f 2 -d "-") # e.g. hc or h1
  if [[ -f "${mito_path}" ]] && [[ "${asse_type}" != "h2" ]]; then
    gunzip -c "${fasta_path}" "${mito_path}" > ./nuc-temp.fa
  else
    gunzip -c "${fasta_path}" > ./nuc-temp.fa
  fi
  strain_id=$(basename "${fasta_file}" | cut -f 1,2 -d "-") # e.g. ADE-hc
  mv -f ./nuc-temp.fa "${strain_id}-genome.fa"
done

### expand and index all the fasta files
cd "${assemblies_dir}"
for ind_a in $(find . -name "*fa"); do
  samtools faidx "${ind_a}"
done
```

Here, we align the SGD genome against any other assembly. We only
perform the direct alignment (compared to mulo-ydh’s “double alignment
and intersection” strategy). We do not want to reduce too much the
number of variants called since this may lead to a high number of FP
calls in the graph dataset.

``` bash
## settings -------------------------------------------------------------------

### base dir
dir_base="${HOME}/prog/ground-truth/gt-1"

## clmnt ----------------------------------------------------------------------

### clean and make output folder
out_dir="${dir_base}/nuc-aln"
if [[ -d "${out_dir}" ]]; then 
  rm -rf "${out_dir}"
fi
mkdir -p "${out_dir}"

### reference file
ref_dir="${dir_base}/genomes/reference"
### just in case we'll want to use many references...
any_ref=$(find "${ref_dir}" -name "*fa")
### ...we can place a loop here
ref_seq="${any_ref}"
ref_name=$(basename "${ref_seq}" | sed 's|-genome\.fa$||')

## collinear assemblies -------------------------------------------------------

in_dir="${dir_base}/genomes/collinear"
cd "${in_dir}"

### direct search, e.g. 1 vs 2
seq_arr=( $(ls *fa) )
seq_dim=$(echo "${#seq_arr[@]}")
all_chroms=$(grep ">chr" "${ref_seq}" | sed 's|>||g')

for (( ind_i=0; ind_i<seq_dim; ind_i++ )); do
  name_i=$(echo ${seq_arr[ind_i]} | sed 's|-genome\.fa$||')
  out_prefix="${ref_name}-vs-${name_i}"
  
  ### clean output files to append the results
  if [[ -f "${out_dir}/${out_prefix}-var.txt" ]]; then
    rm -f "${out_dir}/${out_prefix}-var.txt"
  fi
  
  if [[ -f "${out_dir}/${out_prefix}-coords.txt" ]]; then
    rm -f "${out_dir}/${out_prefix}-coords.txt"
  fi
  
  ### nucmer chromosome-wise
  for ind_c in ${all_chroms}; do

    ### extract chromosome sequence
    samtools faidx "${ref_seq}" "${ind_c}" > "${ref_name}-${ind_c}.fa"
    samtools faidx ${seq_arr[ind_i]} "${ind_c}" > "${name_i}-${ind_c}.fa"
    
    ### go nucmer, go!
    nucmer --prefix="${out_dir}/${out_prefix}-${ind_c}" \
    "${ref_name}-${ind_c}.fa" "${name_i}-${ind_c}.fa"
    
    # delta-filter -u 100 "${out_dir}/${out_prefix}-${ind_c}.delta" \
    # > "${out_dir}/${out_prefix}-${ind_c}-flt.delta"
    
    show-snps -THC "${out_dir}/${out_prefix}-${ind_c}.delta" \
    >> "${out_dir}/${out_prefix}-var.txt"
    
    show-coords -TH "${out_dir}/${out_prefix}-${ind_c}.delta" \
    >> "${out_dir}/${out_prefix}-coords.txt"
    
    ### clean chromosome sequences and delta file
    rm -f "${ref_name}-${ind_c}.fa" "${name_i}-${ind_c}.fa"
    # rm -f "${out_dir}/${out_prefix}.delta"
  done
done

## rearranged assemblies ------------------------------------------------------

in_dir="${dir_base}/genomes/rearranged"
cd "${in_dir}"

### direct search, e.g. 1 vs 2
seq_arr=( $(ls *fa) )
seq_dim=$(echo "${#seq_arr[@]}")

for (( ind_i=0; ind_i<seq_dim; ind_i++ )); do
  name_i=$(echo ${seq_arr[ind_i]} | sed 's|-genome\.fa$||')
  out_prefix="${ref_name}-vs-${name_i}"
  
  ### go nucmer, go!
  nucmer --prefix="${out_dir}/${out_prefix}-wg" \
  "${ref_seq}" ${seq_arr[ind_i]}
  
  # delta-filter -u 100 "${out_dir}/${out_prefix}-wg.delta" \
  # > "${out_dir}/${out_prefix}-wg-flt.delta"
  
  show-snps -THC "${out_dir}/${out_prefix}-wg.delta" \
  > "${out_dir}/${out_prefix}-var.txt"
  
  show-coords -TH "${out_dir}/${out_prefix}-wg.delta" \
  > "${out_dir}/${out_prefix}-coords.txt"
  
  ### delta file
  # rm -f "${out_dir}/${out_prefix}.delta"
done
```

Now, we can filter for the SNPs and make a VCF file which will look
similar to the pggb vcf file (e.g. “bin-AAA-hc.vcf”). The coordinates of
assembly 2 (the query in nucmer’s jargon) may be useful so we’ll keep
them in the vcf file.

``` r
## header ---------------------------------------------------------------------

rm(list = ls())
options(stringsAsFactors = F)
library(data.table)

## settings -------------------------------------------------------------------

### fixed settings
progName <- "ground-truth"
runName <- "gt-1"
dirBase <- file.path(path.expand("~"),
                     "prog", progName, runName)

dirOut <- file.path(dirBase, "os-vcf")
dir.create(path = dirOut, showWarnings = F, recursive = T)
inDir <- file.path(dirBase, "nuc-aln")

### Buff_noaln is [BUFF]: the distance from this SNP to the nearest mismatch 
### (end of alignment, indel, SNP, etc) in the same alignment.
### Dist_seqend is [DIST]: the distance from this SNP 
### to the nearest sequence end.
hdShowSnp <- c("Pos_a1", "Allele_a1", "Allele_a2", "Pos_a2",
               "Buff_noaln", "Dist_seqend", "Strand_a1", "Strand_a2",
               "Chrom_a1", "Chrom_a2")
hdShowSnpNoC <- c("Pos_a1", "Allele_a1", "Allele_a2", "Pos_a2",
                  "Buff_noaln", "Dist_seqend", "Nrep_a1", "Nrep_a2",
                  "Strand_a1", "Strand_a2", "Chrom_a1", "Chrom_a2")

## clmnt ----------------------------------------------------------------------

stringInfo <- readLines(con = file.path(dirBase, "aux", "info-vcf.txt"),)

allFiles <- list.files(path = inDir, pattern = "var.txt$", full.names = T)

for (indF in allFiles) {
  ### set output file
  fileOut <- sub(pattern = "-var\\.txt", replacement = ".vcf",
                 x = basename(indF))
  pathOut <- file.path(dirOut, fileOut)
  dtNucmer <- fread(indF, header = F, sep = "\t")
  colnames(dtNucmer) <- hdShowSnp
  ### filter out variants from non-unique alignments, and indels
  dtSNPs <- dtNucmer[Allele_a1 != "."
                     & Allele_a2 != "."]
  nR <- nrow(dtSNPs)
  dtVCF <- data.table(dtSNPs$Chrom_a1, dtSNPs$Pos_a1, rep(".", nR),
                      dtSNPs$Allele_a1, dtSNPs$Allele_a2, rep(60, nR),
                      rep(".", nR), # FILTER
                      rep(".", nR), # INFO
                      rep("GT:FRMR:FRMQ:CHRQ:STARTQ:ENDQ", nR), # FORMAT
                      paste("1", dtSNPs$Strand_a1, dtSNPs$Strand_a2,
                            dtSNPs$Chrom_a2, dtSNPs$Pos_a2,
                            dtSNPs$Pos_a2, sep = ":"))
  ## make the header ----------------------------------------------------------
  
  headerVcf <- c("#CHROM\tPOS\tID\tREF\tALT\tQUAL\tFILTER\tINFO\tFORMAT\t")
  nameSample <- basename(indF)
  idSample <- paste(unlist(strsplit(nameSample, split = "-"))[4:5],
                    collapse = "-")
  string1 <- "##fileformat=VCFv4.3"
  string2 <- paste0("##fileDate=", date())
  string3 <- paste0("##source=nucmer v4.0.0")
  stringH <- paste0(headerVcf, idSample)
  nameRef <- paste(unlist(strsplit(nameSample, split = "-"))[1:2],
                   collapse = "-")
  pathRef <- list.files(path = file.path(dirBase, "genomes"),
                        pattern = paste0(nameRef, ".*fa$"),
                        full.names = T, recursive = T)
  stringRef <- paste0("##reference=", pathRef)
  pathRefFai <- paste0(pathRef, ".fai")
  
  dtFai <- fread(pathRefFai, header = F,
                 sep = "\t")[, c(1, 2)]
  colnames(dtFai) <- c("Chrom_id", "Len_bp")
  stringContig <- paste0("##contig=<ID=",
                         dtFai$Chrom_id, ",length=", dtFai$Len_bp, ">")
  
  cat(string1, string2, string3,
      stringRef,
      stringContig,
      stringInfo,
      stringH,
      file = pathOut, sep = "\n")
  ### append the vcf to the corresponding header (with the correct sample name)
  fwrite(file = pathOut, append = T, x = dtVCF, quote = F,
              sep = "\t", row.names = F, col.names = F)
}
```

Here, we make the multisample vcf file.

``` bash
## settings -------------------------------------------------------------------

### threads
n_threads=4

### base dir
dir_base="${HOME}/prog/ground-truth/gt-1"

### input and output folders
dir_input="${dir_base}/os-vcf"
dir_out="${dir_base}/ms-vcf"
if [[ -d "${dir_out}" ]]; then 
  rm -rf "${dir_out}"
fi
mkdir -p "${dir_out}"

## clmnt ----------------------------------------------------------------------

cd "${dir_input}"
rm -f *tbi

### compress with bgzip all the single-sample vcf files and index with tabix
for vcf_file in $(ls *vcf); do
  bgzip -c -f "${vcf_file}" > "${vcf_file}.gz"
  tabix "${vcf_file}.gz"
done

### merge to one multisample vcf with bcftools
all_files=( $(ls *vcf.gz) )
bcftools merge --output-type z --threads "${n_threads}" ${all_files[@]} \
> "${dir_out}/multis.vcf.gz"

### prepare header for the next chunk
bgzip -bc "${dir_out}/multis.vcf.gz" | grep "^#" \
> "${dir_out}/multis-genfix.vcf"
```

Finally, we can check the missing genotypes. Warning: the alignment
files (\*coord.txt) may contain reverse alignments for which “start \>
end”, e.g.: 2 765 \| 230934 230165 \| 764 770 \| 87.99 \| chrI chrI

Obviously, they correspond to SNPs with the “reverse” tag: 74 C T 230857
2 74 1 0 1 -1 chrI chrI

``` r
## header ---------------------------------------------------------------------

rm(list = ls())
options(stringsAsFactors = F)
library(data.table)

## settings -------------------------------------------------------------------

### fixed settings
progName <- "ground-truth"
runName <- "gt-1"

dirBase <- file.path(path.expand("~"),
                     "prog", progName, runName)
dirOut <- file.path(dirBase, "ms-vcf")
dir.create(path = dirOut, showWarnings = F, recursive = T)
dirIn <- file.path(dirBase, "ms-vcf")
dirAln <- file.path(dirBase, "nuc-aln")
fileOut <- "multis-genfix.vcf"
hdAlnCoords <- c("Start_a1", "End_a1", "Start_a2", "End_a2", "Len_a1", "Len_a2",
                 "Perc_iden", "Chrom_a1", "Chrom_a2")

## clmnt ----------------------------------------------------------------------

fileIn <- list.files(path = dirIn, pattern = "vcf.gz$", full.names = T)
dtInVcf <- fread(fileIn)
colnames(dtInVcf)[1] <- "CHROM"

### remove non-biallelic loci
indNonBiallelic <- grep(pattern = ",", x = dtInVcf$ALT)
nNonBiallelic <- length(indNonBiallelic)
cat("Found ", nNonBiallelic, " non-biallelic SNPs out of ", 
    nrow(dtInVcf), "\n", sep = "")
```

    ## Found 1343 non-biallelic SNPs out of 190291

``` r
dtBiallelicVcf <- dtInVcf[!indNonBiallelic]

nColBiallelic <- ncol(dtBiallelicVcf)

sampNames <- colnames(dtBiallelicVcf)[10:nColBiallelic]

### count FORMAT tags and set the new string
nTags <- length(unlist(strsplit(dtBiallelicVcf$FORMAT[1], split = ":")))
newTag <- paste(c("0", rep(".", c(nTags - 1))), collapse = ":")

for (indS in sampNames) {
  ### indexes of column indS to be checked for the REF allele
  indC <- grep(pattern = "^\\.", x = dtBiallelicVcf[[indS]])
  cat("There are ", length(indC), " alleles to be checked for sample ", indS,
      "\n", sep = "")
  ### reading the alignment
  strAln <- paste0("SGDref-hc", "-vs-", indS, "-coords.txt")
  fileAln <- list.files(dirAln, pattern = strAln, full.names = T)
  dtAln <- fread(fileAln, sep = "\t")
  colnames(dtAln) <- hdAlnCoords
  
  ### subset vcf and format loci for the comparison
  dtLoci <- data.table(Chr_ref = dtBiallelicVcf$CHROM[indC],
                       Start_ref = dtBiallelicVcf$POS[indC],
                       End_ref = dtBiallelicVcf$POS[indC],
                       Allele_ref = dtBiallelicVcf$REF[indC])
  setkey(dtLoci, Chr_ref, Start_ref, End_ref)
  ### format alignment for the comparison
  setkey(dtAln, Chrom_a1, Start_a1, End_a1)
  
  ### make the comparison using keys from the vcf file
  dtOverlap <- foverlaps(dtLoci, dtAln, nomatch = NA, mult = "all",
                          by.x = c("Chr_ref", "Start_ref", "End_ref"))
  ### reduce the overlap table removing loci that overlap multiple alignments
  boDup <- duplicated(dtOverlap, by = c("Chr_ref", "Start_ref", "End_ref"))
  dtOverlapReduced <- dtOverlap[!boDup, ]
  ### indexes of SNPs with multiple alignments in the reduced table
  dtMultiAln <- dtOverlap[boDup, ]
  strOverlapReduced <- paste(dtOverlapReduced$Chr_ref,
                             dtOverlapReduced$Start_ref,
                             sep = "_")
  strMultiAln <- paste(dtMultiAln$Chr_ref,
                       dtMultiAln$Start_ref,
                       sep = "_")
  indMultiAln <- match(strMultiAln, strOverlapReduced)
  ### boolean of SNPs within one or more alignments (TRUE)
  boAln <- !is.na(dtOverlapReduced$Perc_iden)
  ### boolean of SNPs within only one alignment (TRUE)
  boAln[indMultiAln] <- F
  
  ### update the GT tag
  nCol <- which(colnames(dtBiallelicVcf) == indS)
  dtBiallelicVcf[indC[boAln], nCol] <- newTag
}
```

    ## There are 107847 alleles to be checked for sample DBVPG6044-hc
    ## There are 141602 alleles to be checked for sample DBVPG6765-hc
    ## There are 188806 alleles to be checked for sample S288C-hc
    ## There are 113113 alleles to be checked for sample SK1-hc
    ## There are 111505 alleles to be checked for sample UWOPS034614-hc
    ## There are 122361 alleles to be checked for sample Y12-hc
    ## There are 120785 alleles to be checked for sample YPS128-hc

``` r
fwrite(x = dtBiallelicVcf, file = file.path(dirOut, fileOut), append = T,
       quote = F, sep = "\t", row.names = F, col.names = F)
```

Here, we use RepeatMasker to make the bed files with easy and hard
regions.

``` bash
```
