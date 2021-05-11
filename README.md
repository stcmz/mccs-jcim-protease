# MCCS Case Study for SARS-CoV-2 Main Protease

This repository stores the experimental data of the study on characterizing the binding features of inhibitors and modulators in SARS-CoV-2 main protease using MCCS.

1_original_3cl_pdb_data
--------

This directory includes a complete list of 243 SARS-CoV-2 3CL protease that have known 3D structures determined by either [X-Ray crystallography](https://en.wikipedia.org/wiki/X-ray_crystallography) or [cryoEM technique](https://en.wikipedia.org/wiki/Cryogenic_electron_microscopy).

All are PDB/CIF files included were fetched from https://www.rcsb.org/ directly.

The list snapshot date is March 29, 2021.

2_separated_molecules
--------

We separated the involving molecules from each structure into standalone PDB files. Structures in CIF format were not considered.

3_cleaned_and_prepared
--------

We then manually cleaned up the separated PDB files keeping only the binding protein-ligand pair and undertook the MCCS preparation procedure.

To start with, you need to install [docker](https://docs.docker.com/engine/install/) on your system (Windows/macOS/Linux). Below is the script for the preparation.

```
# In a commandline, checkout the latest version of our MCCS image
docker pull stcmz/mccs

# Change directory to where the cleaned folder locates (assuming name 3_cleaned) and launch a MCCS docker container
cd "path-to-parent-folder-of-cleaned-data"
docker run --rm -it "$(pwd):/data" stcmz/mccs

# Now in the docker container shell
cd /data/3_cleaned

# Run the MCCS preparation
for i in $(find -name *AminoAcids.pdb);
do
    path=$(dirname $i)

    cd $path
    receptor=$(basename $i)
    receptorf=fixed.pdb
    ligand=$(ls *.pdb | grep -vP '(fixed|AminoAcids)\.pdb')

    echo "path=$path receptor=$receptor ligand=$ligand"

    propka3 $ligand >> /data/propka.log
    chimera --nogui --script ~/incompleteSideChains.py $receptor >> /data/chimera.log

    vega $ligand -o ${ligand}qt -f VINA -c Gasteiger -p VINA -l GEN -r APOLAR -w -j FLEX >> /data/vega.log
    vega $receptor -o ${receptor}qt -f VINA -c Gasteiger -p VINA -l GEN -r APOLAR -w >> /data/vega.log
    vega $receptorf -o ${receptorf}qt -f VINA -c Gasteiger -p VINA -l GEN -r APOLAR -w >> /data/vega.log

    # The following lines belong to step 4 - scoring with jdock
    output=/data/4_jdock_scoring_output
    pdb=$(basename $path)
    type=$(basename $(dirname $path))

    mkdir -p $output
    jdock -r ${receptor}qt -l ${ligand}qt -o $output/${type}_original/$pdb -sp >> /data/jdock_original.log
    jdock -r ${receptorf}qt -l ${ligand}qt -o $output/${type}_fixed/$pdb -sp >> /data/jdock_fixed.log

    cd - > /dev/null;
done

# Now that the cleaned data is prepared, rename it
cd /data
mv 3_cleaned 3_cleaned_and_prepared
```

4_jdock_scoring_output
--------

Then [jdock](https://github.com/stcmz/jdock) was used to generate the per residue energy contribution.

See the script above for details.

5_mccsx_collate_output
--------

Finally, [mccsx](https://github.com/stcmz/mccsx) collate was run for vector clustering and heatmap generation.

Below is the script for the generation.

```
# Assuming you are still in the MCCS docker container
# Run mccsx to collate the results
cd /data/4_jdock_scoring_output
output=/data/5_mccsx_collate_output

mccsx collate -l noncovalent_original -o $output/noncovalent_original -rvxcHw --naming dirpath -s correlation -m correlation -M correlation
mccsx collate -l noncovalent_original -o $output/noncovalent_original -rvxcHw --naming dirpath -s correlation -m cosine -M cosine
mccsx collate -l noncovalent_fixed -o $output/noncovalent_fixed -rvxcHw --naming dirpath -s correlation -m correlation -M correlation
mccsx collate -l noncovalent_fixed -o $output/noncovalent_fixed -rvxcHw --naming dirpath -s correlation -m cosine -M cosine

mccsx collate -l covalent_original -o $output/covalent_original -rvxcHw --naming dirpath -s correlation -m correlation -M correlation
mccsx collate -l covalent_original -o $output/covalent_original -rvxcHw --naming dirpath -s correlation -m cosine -M cosine
mccsx collate -l covalent_fixed -o $output/covalent_fixed -rvxcHw --naming dirpath -s correlation -m correlation -M correlation
mccsx collate -l covalent_fixed -o $output/covalent_fixed -rvxcHw --naming dirpath -s correlation -m cosine -M cosine

mccsx collate -l modulator_original -o $output/modulator_original -rvxcHw --naming dirpath -s correlation -m correlation -M correlation
mccsx collate -l modulator_original -o $output/modulator_original -rvxcHw --naming dirpath -s correlation -m cosine -M cosine
mccsx collate -l modulator_fixed -o $output/modulator_fixed -rvxcHw --naming dirpath -s correlation -m correlation -M correlation
mccsx collate -l modulator_fixed -o $output/modulator_fixed -rvxcHw --naming dirpath -s correlation -m cosine -M cosine
```