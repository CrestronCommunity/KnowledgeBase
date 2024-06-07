# SIMPL Windows Tips & Tricks

## .NET Compact Framework 3.5 Installer
When trying to compile a SIMPL Windows program, if you see the error "(LGCMCVT277) Microsoft .NET Compact Framework v3.5 is required for compilation, but cannot be found", download and install [NETCFSetupv35.msi](https://stolhcdn.blob.core.windows.net/olhblob/OLHContent/ProgramFiles/NETCFSetupv35.msi). You should not need to restart your computer or SIMPL Windows. (See [SIMPL: Installer 404 Error Downloading .NET Compact Framework 3.5](https://community.crestron.com/s/article/id-1001084) if the download link doesn't work)

## Comment Symbols
The contents of the `comment` symbol in SIMPL Windows will be included in the program comments (`progcomments`); multiple symbols will be concatinated (without any delimiter)

## Apostrophes in File Names
LPZ/CPZ files can't have an apostrophe in the name. If they do, loading the program will fail and the file will be deleted from the processor.

The error is something like:

    Looking for *.lpz/*.cpz in the current program directory for App 1.Error: Unable to copy current files to backup folder. Aborting program load for App 1. No program running.Unable to load new program

or

    Looking for *.lpz/*.cpz in the current program directory for App 1.
    Unable to load new program.

