# Alma_Linux_Workstation
Setup basic dependencies for VFX work usecase

on machines with a 5080/90 atleast currently as of alma linux 9.6 it is required to add a line to the kernel options at the end of the line that starts with linux

press e to edit the options and add nomodeset rdblacklist=nouveau

this will cause the installer to run without the free nouveau drivers but doesnt need to be done for the 40 series cards or bellow

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------

setup machine as workstation, with gnome applications and internet applications
