###**PyInstaller**

1.  ***Failed to execute script pyi_rth_pkgres:***  
    `error: failed to execute script pyi_rth_pkgres`    
    
    Build the application by adding the --hidden-import argument to the command.    
    For example, running the command given below worked for me. 
    *   `pyinstaller --hidden-import=pkg_resources.py2_warn example.py`
    *   `pyinstaller --hidden-import=pkg_resources.py2_warn --onefile example.py`
    
    [Answer reference](https://stackoverflow.com/questions/37815371/pyinstaller-failed-to-execute-script-pyi-rth-pkgres-and-missing-packages)
    
    Also some guys said:
    
    * Remove PyInstaller pip uninstall pyinstaller.
    *   Download the zip from [github](https://github.com/pyinstaller/pyinstaller/archive/develop.zip).
        or `pip install https://github.com/pyinstaller/pyinstaller/archive/develop.zip`
    *   Unzip file.
    *   Make sure you are in the directory with "setup.py" and run: python setup.py install
    
    [Answer reference](https://github.com/pyinstaller/pyinstaller/issues/2137)
    
2.  ***hidden import problem:***
    
    [Good descption refrence](https://web.archive.org/web/20200601130821/https://pyinstaller.readthedocs.io/en/stable/when-things-go-wrong.html#listing-hidden-imports)
    
    ***Listing Hidden Imports***
    
    If Analysis thinks it has found all the imports, but the app fails with an import error, the problem is a hidden import; 
    
    that is, an import that is not visible to the analysis phase.   
    Hidden imports can occur when the code is using __import__, imp.find_module() or perhaps exec or eval.      
    Hidden imports can also occur when an extension module uses the Python/C API to do an import.   
    When this occurs, Analysis can detect nothing. There will be no warnings, only an ImportError at run-time.  
    To find these hidden imports, build the app with the --debug=imports flag (see Getting Python’s Verbose Imports above) and run it.    
    Once you know what modules are needed, you add the needed modules to the bundle using the --hidden-import= command option, or by editing the spec file, or with a hook file (see Understanding PyInstaller Hooks below).    
