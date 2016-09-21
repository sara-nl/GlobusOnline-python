# Globus Online - python API
This tutorial will show you how to use the Globus Online python API. We will use the interactive mode and take you through all steps needed to transfer and monitor data between two endpoints.

## Prerequisites
To make use of Globus Online, either via the [web GUI](https://www.globus.org/app/transfer) or via any of the APIs, you need to define and activate gridFTP endpoints. These endpoints can be local machines like your laptop, a VM or the SURFsara grid UI, and gridFTP servers, e.g. the servers in the grid or SURFsara's archive.

We provide two predefined public endpoints, the gridFTP endpoint for the grid called surfsara#dCache_gridftp and the archive surfsara#archive.
To add your own laptop as an endpoint, please follow the instructions for adding a gridFTP endpoint via the [webinterface](http://docs.surfsaralabs.nl/projects/grid/en/latest/Pages/Advanced/storage_clients/globusonline.html?highlight=globus) or from [commandline](https://docs.globus.org/how-to/globus-connect-personal-linux/#globus-connect-personal-cli).

Furthermore, you need a grid certificate and access to a proxy server. If you do not have these please contact us before following the tutorial.

The python api can be installed via pip:
```
pip install globusonline --user
pip install m2crypto --user
```

In the following we will show you how to transfer files between the grid UI, the grid and the archive.

## Proxies, endpoint activitation and authentication
Before we start with the python API please make sure that your personal endpoint is activated:
```sh
cd <path to globusconnectpersonal>
$ ./globusconnectpersonal -status
```

To transfer data between two gridFTP servers or a gridFTP server and a personal endpoint we need to create a proxy. This proxy will be stored on a proxy server that can be accessed by Globus Online. This enables Globus Online to act on your behalf between the endpoints.

To create a valid proxy to transfer data to the archive, it is sufficient to create a proxy without any further specifications
```
$ myproxy-init
```

The grid works with virtual organisations (VO) and we need to specify our VO and create a proxy that contains this information.
```
$ startGridSession <VO>
$ myproxy-init --voms <VO> -l <user>
```
You will also be asked to set a password for this proxy. We will need the password to activate the endpoint either via the web interface or the python API.

You can get a list of all of your personal endpoints either via the web interface or via the globusconnect personal tools:
```
$ ssh <globususer>@cli.globusonline.org endpoint-search --scope my-endpoints
```

Open python in the interactive mode with
```sh
$ python -i -m globusonline.transfer.api_client.main cstaiger -p
```
You will be asked for your globus online password, the password you set when creating the globus account, not the password of your grid certificate.

On the whole you need three passwords:
* Grid certificate
* Globus account
* and the one you set everytime you create a proxy 

You can either go to the web interface of globus and activate the grid and archive endpoints with your proxy and then proceed with transferring data by means of the python API. Or you could use the python API to activate these endpoints.

```
>>> api.endpoint_autoactivate("surfsara#dCache_gridftp")
```

This will return a triple containing this information:
```
(200, 'OK', {u'code': u'AutoActivated.CachedCredential', u'resource': u'/endpoint/surfsara%23dCache_gridftp/autoactivate', u'DATA_TYPE': u'activation_result', u'expires_in': 42553, u'length': 0, u'endpoint': u'surfsara#dCache_gridftp', u'request_id': u'vdG4gZnI2', u'expire_time': u'2016-09-21 18:16:42+00:00', u'message': u'Endpoint activated successfully using cached credential', u'DATA': [], u'oauth_server': None, u'subject': u'/DC=org/DC=terena/DC=tcs/C=NL/O=SURFsara B.V./CN=Christine Staiger christine.staiger@surfsara.nl/CN=proxy/CN=proxy/CN=proxy'})
```
To be sure that the endpoint is activated you should always check the *code*.
Try to activate an endpoint you do not have access to and compare the output with the output above.

```
>>> api.endpoint_autoactivate("cineca#PICO")
```

**Exercise** Inspect the help of this function and activate an endpoint only if the existing activation expires in 1h or less.

**Exercise** Activate your personal endpoint. 
Note, that you need the **legacy name** of an endpoint to activate it, this name might be different from the **display name**.
Store the **legacy names** of your activated endopints in two pyton variables *source* and *destination*. We will use these two variables in the next section to actually transfer data.

## Transfer between a user interface or laptop to a gridFTP server

We will now transfer data between our personal endpoint and the grid.

```
>>> source="<your endpoint>"
>>> destination="surfsara#dCache_gridftp"
```

First, we create a submission ID and store it in a new variable
```
>>> code, message, data = api.transfer_submission_id()
>>> submission_id = data["value"]
```

The output has the same structure as the output for the endpoint activation. The actual transfer is stored in an object of the class *Transfer*.
```
>>> help(Transfer)
```

To initiate such an object we need a submission ID, the source endpoint, the destination endpoint; we can set a deadline and a synchronisation level (check the help of globus-url-copy) and a label for the transfer.
Globus Online automatically checks the MD5 checksums. However, MD5 is not support by our grid infrastructure. That is why we need to set an additional parameter *verify_checksum=False*.

Our proxy is valid for 12 hours, thus the deadline for the transfer should fall into this time interval:
```
>>> import datetime
>>> deadline = datetime.datetime.utcnow() + datetime.timedelta(hours=12)
```

If you already copied e.g. a folder or a larger file setting the sync-level will prohibit you from retransferring data that already exists at the destination. Please consult the globus-url-copy help (`man globus-url-copy`, sync-level number) for more information.

```
>>> sync_level=0
>>> label="My first data transfer"
```

Initiate an instance of *Transfer*:
```
>>> t = Transfer(submission_id, source, destination, deadline, sync_level, label, verify_checksum=False)
```

What we now need to do, is to add pairs of file source location and file destination location.
Let us create a folder containing some data on the (bash) shell on our personal endpoint:
```
$ mkdir TestData
$ for i in {000..010}; do echo "Test file ${i} and some text.">"TestData/File${i}.txt"; done
```
and add it to the transfer object (in the python shell):
```
>>> t.add_item("/home/<user>/TestData/", "/pnfs/grid.sara.nl/data/<VO>/<user>/TestData/", recursive=True)
```
You can add as many items to the transfer as you wish. Note that *recursive* is by default set to *false*. You do not need the recursive option when transferring single files.

Start the transfer task:
```
>>> code, reason, data = api.transfer(t)
>>> task_id = data["task_id"]
```
**Exercise** Inspect the code and the data. Create some invalid transfers (wrong path, unactivated endpoints , ...) and reinspect the data.

## Monitoring
With the *task_id* you can ask for the status of the transfer
```
>>> code, reason, data = api.task(task_id)
```

Inspect the *data*. You can also monitor this transfer in the webinterface of globus.

**Exercise** Write a function that takes as input the data from a *api.transfer(t)* and print the information needed for monitoring in a pretty way. (Which items do you need?)

**Exercise** Write a function that prints the data from *api.task(task_id)* in a nice way such that it simplifies the monitoring.

## 3rd arty transfers
Now that we have some data on the gridFTP server of the grid we can transfer this data further to another gridFTP server e.g. the archive.

```
>>> source = "surfsara#dCache_gridftp"
>>> destination = "surfsara#archive"
>>> code, message, data = api.transfer_submission_id()
>>> submission_id = data["value"]
>>> label="3rd party transfer"
>>> t = Transfer(submission_id, source, destination, deadline, sync_level, label, verify_checksum=False)
>>> t.add_item("/pnfs/grid.sara.nl/data/<VO>/<user>/TestData/", "/home/<user>/TestData/", recursive=True)
>>> code, reason, data = api.transfer(t)
>>> task_id = data["task_id"]
```

## Exercises putting it all together
1) Write a function to activate an endpoint. Check whether the endpoint is correctly activated and give some usefule error messages in case it is not done correctly.

2) Write a function that initialises a *Transfer* object with all necessary parameters.

3) Write a function that takes as input a list of tuples [(source data, destination data), ...] and adds it to a *Transfer* object. Watch out for recusive transfers.

4) Write a function that checks the status of a transfer.

Put it all together in one python script *myGO.py* using the template below:
```py
from globusonline.transfer.api_client import x509_proxy, Transfer, create_client_from_args

#import some additional libraries
import datetime
import time
import getpass
import re

def activate(endpoint_name):
    # Your code here
    # Think of expiring times

def display_endpoint(endpoint_name):
    code, reason, data = api.endpoint(endpoint_name)
    # Your code here
    # Print the information in a readable form

def create_task(<parameters>):
    # Set parameters for a task and create an object
    # You will need to set the input parameters

    t = Transfer(<parameters>)
    return t

def add_items_to_transfer(t, list_of_items):
    # add the pairs of source data destination data to t
    
    for sourcef, destf in list_of_items:
        #your code here

def display_task(task_id):
    # Your code here
    # Print the information in a readable form

    code, reason, data = api.task(task_id)

if __name__ == '__main__':
    api, _ = create_client_from_args()
    username=raw_input('Enter Myproxy username:')
    passwd=getpass.getpass('Enter MyProxy pass phrase:')

    #activate two endpoints
    source = <endpoint>
    destination = <endpoint>
    activate(source)
    activate(destination)

    print "Transferring data from:"
    display_endpoint(source)
    print "\nto:"
    display_endpoint(destination)

    #create a transfer
    t = create_task(<parameters>)
    #generate pairs of source files and folders and destination files and folders as list of tuples
    list_of_items = #Your code here

    add_items_to_transfer(list_of_items)

    code, reason, data = api.transfer(t)
    task_id = data["task_id"]

    #check status of the transfer
    print "Transfer status:"
    display_task(task_id)

```
Once finished the script can be called by:

```
python myGO.py 
```


