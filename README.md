# Extracting target genes from Braconidae reference genomes

## 1. Download reference genomes

From the [NCBI Datasets genome page for Braconidae](https://www.ncbi.nlm.nih.gov/datasets/genome/?taxon=7402), download all reference genomes marked with the green check mark.

Put all reference FASTA files into one folder and compress them:

```bash
cd your_reference_folder
gzip *.fasta
```

## 2. Install the tools

Install compleasm, miniprot and SeqKit:

```bash
sudo apt install python3-pandas

# compleasm
wget https://github.com/huangnengCSU/compleasm/releases/download/v0.2.9/compleasm-0.2.9_x64-linux.tar.bz2
tar -jxvf compleasm-0.2.9_x64-linux.tar.bz2
rm *.tar.bz2

# miniprot: https://github.com/lh3/miniprot
wget https://github.com/lh3/miniprot/releases/download/v0.18/miniprot-0.18_x64-linux.tar.bz2
tar -jxvf miniprot-*
rm *.tar.bz2
sudo mv miniprot-0.18_x64-linux/miniprot /usr/local/bin/
rm -r miniprot-*

# SeqKit: https://github.com/shenwei356/seqkit/
# Usage: https://bioinf.shenwei.me/seqkit/usage/
wget https://github.com/shenwei356/seqkit/releases/download/v2.13.0/seqkit_linux_amd64.tar.gz
tar -zxvf seqkit*
sudo cp seqkit /usr/local/bin/
rm seqkit_linux_amd64.tar.gz
rm seqkit
seqkit
```

## 3. Prepare the inputs

After all installations are completed, you can extract your genes from all the reference files and create new `.fas` files for your target genes.

First, set the path to your genomes:

```bash
genomes=/mnt/c/Braconidae_Ref
```

To extract the reference genes from your FASTA files, you need a `ref.fas` file containing the amino acid sequences of your target genes (see [ref.fas](ref.fas)).

Lastly, you need a text file listing the reference FASTA files you have:

```bash
ls GCA* > IDs.txt
```

## 4. Run the extraction

Now we have everything we need.

```bash
for i in $(cut -f1 IDs.txt)
     do echo "qacc qlen qstart qend strand sacc slen sstart send nmatch length mqual alnscore alnscoreNoIntrons pmatch distStart distEnd protCIGAR DifferenceString" | awk -v OFS="\t" '$1=$1' > miniprot.paf
     $HOME/compleasm_kit/miniprot -I $(ls "$genomes"/*.gz | grep $i) ref.fas >> miniprot.paf
     awk '$5 == "+"' miniprot.paf > for.txt
     for k in $(cut -f1 for.txt | uniq)
     do grep $k for.txt > uresults.txt
       while read line
       do echo $line | awk '{print $8 - 100, $9 + 100}' | sed -r 's/(\s*)-[0-9.e-]+/\11/g' > temp.txt
          seqkit grep -p $(echo $line | cut -d ' ' -f6) $(ls ""$genomes""/*.gz | grep $i) |
          seqkit subseq -r $(awk '{print $1}' temp.txt):$(awk '{print $2}' temp.txt) |
          seqkit replace -p "\s.+" |
          sed s/$(echo $line | cut -d ' ' -f6)/$i.$(grep $i IDs.txt | cut -f2).$(echo $line | cut -d ' ' -f6).$(echo $line | cut -d ' ' -f8)..$(echo $line | cut -d ' ' -f9)/ >> $k.contigs.fas
       done < uresults.txt
     done
     awk '$5 == "-"' miniprot.paf > rev.txt
     for k in $(cut -f1 rev.txt | uniq)
     do grep $k rev.txt > uresults.txt
       while read line
       do echo $line | awk '{print $8 - 100, $9 + 100}' | sed -r 's/(\s*)-[0-9.e-]+/\11/g' > temp.txt
          seqkit grep -p $(echo $line | cut -d ' ' -f6) $(ls ""$genomes""/*.gz | grep $i) |
          seqkit subseq -r $(awk '{print $1}' temp.txt):$(awk '{print $2}' temp.txt) |
          seqkit replace -p "\s.+" |
          seqkit seq -p -r -v -t DNA |
          sed s/$(echo $line | cut -d ' ' -f6)/$i.$(grep $i IDs.txt | cut -f2).$(echo $line | cut -d ' ' -f6).$(echo $line | cut -d ' ' -f8)..$(echo $line | cut -d ' ' -f9)/ >> $k.contigs.fas
       done < uresults.txt
     done
  done
```
