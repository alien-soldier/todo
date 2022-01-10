## Wallboard Api instruction and problem sets

### Instruction     

---
**Python set up part**
1.  Download and install python 3.5 - 3.7, 32bit version (3.6.8 was ok) [download](https://www.python.org/ftp/python/3.6.8/python-3.6.8.exe)    
    32bit is a must for odbc 32bit driver. and 3.5 - 3.7 needed for pyinstaller.     
    Set global Path during installation (also you could install it for all users)
    > if you not select all users you should give access to **IUSR** and **IIS_IUSRS** for your python folder (in addtion to the flask-app virtualenv) 
    > (C:\Users\Administrator\AppData\Local\Programs\Python) while depolying
    > [depolying source](https://stackoverflow.com/questions/60043072/python-exe-the-fastcgi-process-exited-unexpectedly)
2.  install pipenv using cmd `pipm install pipenv`
3.  Download and install pycharm

---
**Fetching part (Cisco project)** 

1.  Open pycharm for making the first project. ***"fetching from informix"*** project, I call it ***"Cisco"***  
    Use pipenv as interpreter of project. pycharm make virtual env using global python version.     
    >   I highly recommend to rename project folder from "untitled" to "Cisco". pipenv borrows project folder name for 
    virtual env name and its folder and it prevents future confusing and helps to distinguish between virtual envs.   
2. Install needed packages

    >   You should install them in virtual env. for that you can:  
        1)   use cmd and go the project folder, type 'pipenv shell' for activating it.
             Now run 'pipenv install 'package_name'' for installing packages    
        2)   Right click on the project folder on pycharm and click 'open in terminal'. 
             it open cmd on current path. Now run 'pipenv install 'package_name'' for installing packages   
    * sqlalchemy
    * pyodbc
    * numpy
    * pandas
    * redis
    * pyinstaller
3.  Install redis and set it as a global service
    [source](https://github.com/microsoftarchive/redis/releases/tag/win-3.2.100)
4.  Write fetching script and run it. If everything is correct you can get the result from redis cli.
    Fetching script could be like this:
    ```python
    import pandas as pd
    import pyodbc
    import redis
    import pickle
    import zlib
    
    r = redis.StrictRedis('localhost', port=6379, charset="utf-8", decode_responses=False, db=0)
    
    conn = pyodbc.connect('DSN=uccx;PWD=P@ssw0rd')
    query = "select blah, blah, blah from RtCSQsSummary"
    df = pd.read_sql(query, conn)
    
    """your processing stuff"""
    
    # adding the processing date and time
    df['date'] = datetime.now().strftime('%Y-%m-%d, %r')
    
    # set on redis
    r.set("df", zlib.compress(pickle.dumps(df)))
    ```  
---	
**Api Part (Cisco Api project)**
1.  click 'File>New project' on pycharm for making flask project. the project type can be Flask.
    select pipenv as a project interpreter and don't forget to rename project from "untitled" to "CiscoApi".
2.  Install needed packages
      
    >   You should install them in virtual env. for that you can:  
        1)   use cmd and go the project folder, type 'pipenv shell' for activating it.
             Now run 'pipenv install 'package_name'' for installing packages    
        2)   Right click on the project folder on pycharm and click 'open in terminal'. 
             it open cmd on current path. Now run 'pipenv install 'package_name'' for installing packages  
    * flask
    * flask-restful
    * redis
    * wfastcgi
    * pandas
    * flask-cors
    
        >   if you used `df.to_msgpack()` for saving dataFrame on redis. you need `pd.read_msgpack(redis.get("key"))` for
        converting the retrieved object(string) from redis object(string) to dataFrame.    
        ***but if you used pickle and zlib `zlib.compress(pickle.dumps(df)` for storing on redis 
        you does not need pandas in flask part, because the df in `df.to_json(orient='records')` is only an string which 
        have been retrieved from redis and not a dataFrame. it just named to df and is not a dataFrame***  
    
3.  write your flask app. I used flask_restful not flask itself. (below example is a minimal flask app).:
    ```python
    from flask import Flask
    app = Flask(__name__)
    @app.route("/")
    def hello():
        return "Hello from FastCGI via IIS!"
    if __name__ == "__main__":
        app.run()
    ``` 
    And my restful_flask is like this:
    ```python
    import redis
    import zlib
    import pickle
    import json
    
    from flask import Flask
    from flask_restful import Resource, Api
    
    app = Flask(__name__)
    api = Api(app)
    
    r = redis.StrictRedis('localhost', 6379, charset="utf-8", decode_responses=False, db=0)
    
    class Cisco(Resource):
        def get(self):
            df = r.get("df")  # df is not dataFrame. It is a redis value (string)
            if df is None: # The key does'nt exists on redis
                return {'successful': 'false', 'result': None}
    
            df = pickle.loads(zlib.decompress(df))
            result = json.loads(df.to_json(orient='records'))
            return {'successful': 'true', 'result': result}
    
    api.add_resource(Cisco, '/')
    
    if __name__ == '__main__':
        app.run()
    ```
4.  Test the Flask app on the local flask server using pycharm run button. If everything was ok it's time to deployment.
---	
**Deployment Part (deploying flask app on IIS)**

The full guide was captured by a video.      
But the important steps are:    
1.  After installing wfastcgi you should enable it.  
    > run the command prompt as administrator and on activated virtual env run this command:     
    `wfastcgi-enable`    
    This will give you something like this which later will be used in IIS
    `c:\users\administrator\.virtualenvs\ciscoapi-wlbhpfnx\scripts\python.exe|c:\users\administrator\.virtualenvs\ciscoapi-wlbhpfnx\lib\site-packages\wfastcgi.py`
    The first part is the address of project python interpreter (virtual env) and the second one is the address of wfastcgi script module (also for the virtual env) 
    
2.  Enable the cgi feature of iis [reference](https://docs.microsoft.com/en-us/iis/configuration/system.webserver/security/isapicgirestriction/add)
3.  Open IIS [installation reference](https://docs.microsoft.com/en-us/iis/install/installing-iis-85/installing-iis-85-on-windows-server-2012-r2)
4.  Right-click on the server name and select add website.
5.  Enter the 'site name', 'physical path' and 'site binding.
    > physical path : the flask app folder address (C:\Users\Administrator\PycharmProjects\CiscoApi)    
    on site binding you can set port (like 81) 
6.  After adding site select the site name and double click on the 'handler mapping' feature from the middle pane.
    in right pane select "Add Module Mapping"   
7.  on 'Edit Module Mapping' fill:  
    `Request path` : `*`       
    `Module` : `FastCgiModule`    
    `Executable (optional)` :    
    > The result of `wfastcgi-enable` command which was:    
         `project python interperter virtual env address|fastcgi_module of virtual env address`       

    `Name: an Optional name like FlaskHandler`      
8.  Click 'Request Restrictions'. Make sure the `Invoke handler only if request is mapped to:` checkbox is unchecked
9.  click 'ok'
10. Click 'Yes' when prompt `Do you want to create a FastCGI application for this ... blah blah bla:`
11. Now go back and again select the server name on IIS and select fast CGI setting from the middle pane.
12. Double click on it (The created row), then click on the "..." for the Environment Variables collection to launch the EnvironmentVariables Collection Editor:
13. Click Add for adding two below variable:
    > `PYTHONPATH` : `Flask project physical path`  
    `WSGI_HANDLER` : `app.app`  
    My Flask app is named app.py so the value is app.app — if yours is named site.py it would be site.app or similar. In other word       
    -   The first part is: ***"The name of flask app entry point (the .py file)"***     
    -   The second part is: ***"The variable was set on the script for instancing the Flask class (often is 'app' because app = Flask(--name--)"***     
14. Click OK.
15. The final and tricky step which is granting permission: 
    > You should assign the `IUSR` and `IIS_IUSRS` user permission to the 'flask site folder' and 'python folder'. how?     
    > ### from python folder I mean installed python itself, not the virtual-env for flask_app    
    > ### python-folder: `C:\Users\Administrator\AppData\Local\Programs\Python`     
    > ### flak_app folder: `C:\Users\Administrator\PycharmProjects\CiscoApi`
    
    **new solution** (short key without need any windows permision):    
    -   right_click on folder > give access to > specific people > paste **IUSR** and click on **Add** button, paste **IIS_IUSRS** and click on **Add** button  
        Then click **Share** button.    
    Do this for both 'flask site folder' and 'python folder'    
    
    **old solution:**   
    -   For flask site folder right click on your site on IIS. or go to the flask project folder itself and right click on it.  
        Then select 'Edit permission > security > edit > add >'    
        write `IUSR` then click on 'check name' for finding it. then add it. repeat it again for `IIS_IUSRS`
    -   repeat above step but for your project python virtual env folder.
    > If "Object named IIS_IUSRS could not be found":   
    On the Add panel, click on "Locations"; Now scroll up to the top and you should see the name of your machine you are trying to configure at the top of the tree structure.       Select this entry and hit ok. IIS_IUSRS should now be correctly found for instance as [DMNNAMEOFMACHINE]\IIS_IUSRS;        
    [reference](https://stackoverflow.com/questions/30846213/object-named-iis-iusrs-could-not-be-found-when-adding-user-to-php-sessions-dir)
    
[Deployment reference](https://stackoverflow.com/questions/58261464/how-to-host-python-3-7-flask-application-on-windows-server)
    

    
    
    
    
    


---	
#### Findings summary:
    1.  For odbc.32 bit version, you should install python 32bit (x86).
    2.  For using pandas you should have "Microsoft Visual C++ Redistributable for Visual Studio 2015, 2017 and 2019" also in 32bit. (named vc_redist.x86.exe)
    3.  Pyinstaller supoersts only python 3.5 - 3.7, then I installed python 3.6.8 for making .exe file (I have written one MD file for descibing chanlenges in Pyinstaller) 
    4.  For saving pandas to redis, the builtin function df.to_msgpack() is available in pandas 0.25.1, in pandas 1.0.0 to_msgpack is deprecated in favor of pyarrow.
        Then you have three options:
            using pandas 1.0.1 and pyarrow
            using pandas 0.25.1 and df.to_msgpack()
            using pickle and zlib solution
        I tried to installing pyarrow but there was some problem in installation
        Eventually I used pickle solution in contrast to downgrading the pandas
    5. Deployment Flask on windows has some complexity which I idescribed it in a video. 
         

#### Errors set:
     
1.  ***pyodbc Error:***     
    ```pyodbc.Error: ('IM014', '[IM014] [Microsoft][ODBC Driver Manager] The specified DSN contains an architecture mismatch between the Driver and Application (0) (SQLDriverConnect)')```
    
    You have to make sure the Python version matches the ODBC driver version: 32-bit with 32-bit, 64-bit with 64-bit.
    It looks like you have 64-bit Python / pyodbc and 32-bit MS Access.
    What you'll need to do is install the 32-bit Python version, and then install pyodbc.
    
    [Answer reference]([https://stackoverflow.com/questions/36965600/pyodbc-the-specified-dsn-contains-an-architecture-mismatch-between-the-driver])


2.  ***import pandas Error:***  
    ```ImportError: DLL load failed while importing aggregations: The specified module could not be found```
    
    You may also try to install a fresher version of Microsoft Visual C++ Redistributable.
    E.g. (2015-2019)
    I had the same problem, and installing the aforementioned resolved it.
    
    [C++ Redistributable download link](https://support.microsoft.com/en-us/help/2977003/the-latest-supported-visual-c-downloads)
    
    [Answer reference](https://stackoverflow.com/questions/60767017/importerror-dll-load-failed-while-importing-aggregations-the-specified-module)

3.  ***redis does not response***   
    [redis 3 download link](https://github.com/microsoftarchive/redis/releases)  
    Add redis as environment variable   
    Open cmd then
    `redis-cli`
    
4.  ***pandas to redis***   
    `AttributeError: 'DataFrame' object has no attribute 'to_msgpack'`  
    As of pandas 0.25.1, to_msgpack is deprecated in favor of pyarrow  
    Solution with pickle:   
    ```python
    import pickle
    import redis
    import zlib
    
    # decode_responses and charset should be set for unicode 
    r = redis.StrictRedis(host='localhost', port=6379, db=0, charset="utf-8", decode_responses=False)
    # Set
    r.set("key", zlib.compress(pickle.dumps(df)))
    # Get
    df = pickle.loads(zlib.decompress(r.get("key")))
    ```
    [Answer reference](https://stackoverflow.com/questions/37943778/how-to-set-get-pandas-dataframe-to-from-redis)
    
    and using pyarrow it can be
    ```python
    import pyarrow as pa
    import redis
   
    r = redis.StrictRedis(host='localhost', port=6379, db=0, charset="utf-8", decode_responses=False)
    context = pa.default_serialization_context()
    # Set
    r.set("key", context.serialize(df).to_buffer().to_pybytes())
    # get
    context.deserialize(r.get("key"))
    ```
    [Answer reference](https://stackoverflow.com/questions/57949871/how-to-set-get-pandas-dataframes-into-redis-using-pyarrow/57986261#57986261)

# For Allowing CORS
1.  pip install flask-cors
2.  In the simplest case, initialize the Flask-Cors extension with default arguments in order to allow CORS for all domains on all routes       
    ```python
    from flask_cors import CORS
    
    app = Flask(__name__)
    CORS(app)
    api = Api(app)    
    ```     
    [Reference](https://flask-cors.readthedocs.io/en/latest/)


# For depolying client part on IIS:
1.  you can use the IIS 'Default Web Site'. by exploring its folder and replace your own files.(index, script, css)
>   There is no need to extra steps.    
    some good reference about it:       
    [installing IIS](https://computingforgeeks.com/install-and-configure-iis-web-server-on-windows-server/)     
    https://www.atlantic.net/hipaa-compliant-hosting/how-to-build-static-website-iis/   
    https://www.guru99.com/deploying-website-iis.html   
    https://www.codeproject.com/Tips/826803/How-to-Access-the-Site-using-Domain-Name-Instead-o      
    https://support.rackspace.com/how-to/modify-your-hosts-file/        
    CMD run as administartor and run:
        notepad c:\windows\system32\drivers\etc\hosts
    
