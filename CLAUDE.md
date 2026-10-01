# Aim

Create a colab notebook for fitting two or more protein structures in PDB or mmcif format based on kearsley fitting of atoms.

# Stages in colabnotebook

## Stage 1: install dependencies

Install all dependencies/ git pull

## Stage 2: Input

User should be able to upload structure as files or optionaly enter a pdbid. This should be one row of fields by defailt with option to add more rows 
The first field is set as the the refernace structure. All the following structure will be fitted to this.
If two inputs have  the same  filename rename as origina_n where n is an integer. in this case 

### Additional optional parameters.

SOdc: Euclidian distance cutoff used to calculate structure overlap. All selected atoms (defined in next section) of one structure should be withing SOdc of the corresponding atoms of the reference strucutre for the superimposition to be reported. default is 1.5 angstrom (all distances and units in angstrom). 
RMSDc: Maximum value for RMSD above which the alignment is considered meaningless. Default is 1.5A.

## Stage 3: Input checks

Parse the input and create structure objects (see Program architecture section).
List total atoms,residues and chains parsed for each structure.
Seperately list the heteroatom residue names and residue ids

## Stage 3: Atom specification

Using a hierarchila selection method, select exact set of atoms to fit for each structure. 
The user first selects the model and chain from a dropdown list, then the second dropdown list show the list of residue which is selected and then atom names with atom ids in bracket. optionally the user should also be able to enter the atom id (which is unique for each strucutre) and make the selection. In addition this manual step can be skipped by uploading the matomid pairs in a csv file with the first column as the referance and each additional column as structures to fit. the headders of each of these columns should be the file name. 

Fitallmodels: check box, if selected fit each model using the atom specifier. The model specifed in the selection will be used as the referance if a seperate referance is not provided.

The stage should print the atom pairs and also check if the number of atoms match. If they do not match its an error. If they match and the atom type or element type is different it should be a warning. 

Atlease two set of atoms should be provided. These can however be from the same input file. in such cass make a copy of the structure and treat it ths same as inputs with same filename. 

## Stage 4: Fitting

Fit the structures with the fitting module. and make the output strucutres and files. 
List the superimposition statistics.

## stage 5: Download
Download zipped files.

# Program architecture 

Use kearsley-numba 0.1.0 for superimposition: https://github.com/bigbigdumdum/kearsley_numbajit. git clone to see the exact usage in README.md file

The program is to be made as 2 repositories (two seperate git repos)

## Repository 1: Python engine
Modules 
 * Parser: Parser input PDB and mmcif files
 * Input checker: Modules to check if the number of atoms that are specified are the same. If the atom type or element type is different, issue warning.
 * Fitter: Run kearsley fitting between referance the rest of the inputs. Transform all cordinates in each input except the referance based on the rotation and translation
 * output: make new files in the same format as the input by swapping the cordinates to the superimposed cordinates. make a .txt file with Detained report. Report should start with the input mode (user upload or PDBid), file name and check sum (SHA-256), and cutoff parameters used. List the number of atoms superimposed for each referance pair. In a seperate csv file wirte the exact pair or atoms used for superimposition for each supeirmposition. with the rotation and translation matrices

This repository should be independent of anything related for the colab notebook as it is to be reused for many other programs

## Repository 2: Colab 

This propsitory contains the collab notebook which uses the modules from repository 1 to execute the different stages.

# Strict agent instrucitons
This file cannot be modified by the agent at any cost. Any modification to the git repo should need permissions. 

# Programming practices
All programs to be written in a modular resuable manner.
All functionas and methods should have comments and instructions for future human modificaiton and anaysis.
Program should be simple and should avoid external dependencies unless necessary.

# identifier

unique identifier for each atom is the string f"{modelnumber}-{atomid}" this id is to be refered as matomid

# Copyright

Copyright (C) 2026 Mukundan S on top of each source file and Readme. 

# testing

use  two structures of GFP, 1EMA and 6L26 and fit by the chromophore carbon atoms.