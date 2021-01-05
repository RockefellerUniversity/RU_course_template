# Teaching at RU
A template repository from which to build workshops and other teaching materials in a standard manner to allow integration with other RU material. 


## How to use

Use this as a template. Take a fork and work from there.  

All the course content is contained in the package. This structure is important as we will use this, along with the GitHub actions to automatically compile your course.  

You will need to first rename the package and the associated R project. From there you can add in the appropriate files for the course content, and update the config files. There are placeholders along with examples and formatting guides for most files. Anything surrounded by double question marks [??] is text to help, but should be replaced. 

Another thing to look out for is links. You will need tp update any paths that point into the package to reflect the packages name.

## Course Content

### Course slides
Find these at _MyCoursePackage/inst/extdata/presRaw/*_  
Check out the example to get some idea of formatting within the Rmd

### Exercises
Find these at _MyCoursePackage/inst/doc/*_  
Check out the example to get some idea of formatting within the Rmd

## Config files

### DESCRIPTION
Find this at *MyCoursePackage/DESCRIPTION*  
Basic package description file. Ensure you have added in all dependencies you use in the package. Otherwise it will not compile. 

If you have non-R dependencies put them in the SystemRequirements field. Use the name associated with the conda installation. We will use Herper to install the software. 

### Descriptions
Find these at *MyCoursePackage/inst/doc/Descriptions*  
These files contain descriptive text, from which the cover page for the course is built. The placeholders show roughly what to expect. There are two types. 

1. The Course Overview: This Rmd contains a description of the overall course.
2. The session overview: There will be one of these for each session you break the course down into. There could just be one, if you just have a single session. 

### course.yml
Find this at *MyCoursePackage/inst/doc/_course.yml*  
This yml will contain the name for all the Rmd files.
You will need to update the CourseName. 
If any of the Rmds have a different name to the template, update the .yml to reflect that. The order is important so the RMd for your first section should be first. 
Each Rmd should be separated by a space. Except for exercises. Exercises should be in the order they appear. Exercises in the same section will be comma separated. There will then be a space between sections i.e. in the template the first 2 exercises are asssocaited with the first session, while the 3rd exercise is associated with the final session. 


## .github files for compiling
