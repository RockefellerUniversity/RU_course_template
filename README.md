# Teaching at RU
A template repository from which to build workshops and other teaching materials in a standard manner to allow integration with other RU material. 

data/image insetion examples in ex and main site
what if reqs arent in list?


The pacakge
rename both folde and Rproj
DESCRIPTION file
exercises (inst/doc)
extdata 
course.yml
Descriptions
2 types:  CourseOverview.Rmd
          Session1Overview.Rmd


## Structure
These are the main files and directories you will need. There will often be others as well you can add that are self explanatroy: notes, scripts, outputs.....


### Index
This is the web homepage for the specific course. This will link to the BRC RU hub, along with all the information that is needed to prepare for the course. 

### Compiler
This compiles your READMEs (the course itself, along with exercises and questions), into html (both single page and presentation), along with just the code. This may need to be customized (see RU_github course) to allow you to run all the code during the compilation (also be wary that this code is run several times to get the various markdown outputs).

*Essential: This is NOT for public. When you are making your workshop public, this file must be removed from GitHub and added to your ,gitignore so you can maintain a copy in your local repository). 


### .gitignore
All files you want to be maintained locally, but that you do not want to end up on the repository on GitHub can be added here. It is essential the compiler is added to this before your repository is made public. 

### r_course
This directory contains all your course content in it. There are several directories within this. 

#### customCSS
This contains the foramtting css files that dictate the appearance of your course. We keep this consistent between workshops for uniformity.

#### dataset
Any datsets that you are working with can go here. One big thing to be wary of is that you do not want them to be too big as GitHub has size limits for files.

#### Exercises and Answers
Exercises and Answers for these exercises will be put in these separate diorectories. These must be markdown files. These will be compiled by the compiler script to make the html files in the same directory. 

#### Presentations
You will not need to fill this section,
Internally there will be a r_code, singlepage and slide directories, but these will be populated by the compilation script from the presRaw directory

#### presRaw
The markdown for the course content will be kept in here (or markdowns if you are running multiple split sessions). From here it will be copied and compiled into the various outputs by the compiler script. 

#### img
All images that you want inserted into the markdowns will be stored in here. 

