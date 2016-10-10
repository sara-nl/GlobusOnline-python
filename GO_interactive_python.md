# Globus Online - python API
This tutorial will show you how to use the Globus Online python API. We will use the interactive mode and take you through all steps needed to transfer and monitor data between two endpoints.

## Prerequisites

### Accounts
We assume that you are working on a linux machine.
You need several accounts and certficates:

- A [globus online account](https://www.globus.o]). Some stepd require to connect to the server cli.globusonline.org via commandline. Athentication on this server works with rsa keys. Please follow the instructions [here](https://docs.globus.org/faq/command-line-interface/#how_do_i_generate_an_ssh_key_to_use_with_the_globus_command_line_interface) to link your globus account to rsa keys and make sure that all servers and personal computers you want to use for this tutorial are setup with a valid public-private key pair.

- A grid certificate (usercert.pem and userkey.pem)
- Access to a proxy server (comes when you obtained an account on ui.grid.sara.nl)

If you do not have these please contact us before following the tutorial.

### Endpoints
To make use of Globus Online, either via the [web GUI](https://www.globus.org/app/transfer) or via any of the APIs, you need to define and activate gridFTP endpoints. These endpoints can be local machines like your laptop, a VM or the SURFsara grid UI, and gridFTP servers, e.g. the servers in the grid or SURFsara's archive.

We provide two predefined public endpoints, the gridFTP endpoint for the grid called surfsara#dCache_gridftp and the archive surfsara#archive.

To add your own laptop, your account on ui.grid.sara.nl or any other personal computer as an endpoint, please follow the instructions for adding a gridFTP endpoint via the [webinterface](http://docs.surfsaralabs.nl/projects/grid/en/latest/Pages/Advanced/storage_clients/globusonline.html?highlight=globus) or from [commandline](https://docs.globus.org/how-to/globus-connect-personal-linux/#globus-connect-personal-cli). 
The latter requires logging in to the server cli.globusonline.org. 
Authentication on this server is done by rsa key. Please follow the instructions [here](https://docs.globus.org/faq/command-line-interface/#how_do_i_generate_an_ssh_key_to_use_with_the_globus_command_line_interface) to export your public key.  

### Python libraries
The python api can be installed via pip:
```sh
pip install globusonline --user
pip install m2crypto --user
```

You should install these libraries on all computers you will run the python API on.

In the following we will show you how to transfer files between the grid UI, the grid and the archive.

## Final checks
- Make sure you are running globusconnectpersonal on your personal endpoint:
```sh
cd <path to globusconnectpersonal>
$ ./globusconnectpersonal -status
```
- Make sure you installed the python packages
```sh
python -c "from globusonline.transfer import api_client; print api_client.__path__"
``` 
- Make sure you have all of the endpoints at hand
```sh
$ ssh <globususer>@cli.globusonline.org endpoint-search --scope my-endpoints
$ ssh <globususer>@cli.globusonline.org endpoint-search --scope all SURFsara
```

You should have the following endpoints:
- Your previously added personal endpoint
- surfsara#dCache_gridftp
- surfsara#archive

## Proxies, endpoint activitation and authentication
In this section we will show you how to activate gridFTP endpoints (not your personal endpoint but the actual gridFTP servers).
To this end you need your grid certficate and access to a proxy server.

The worflow is as follows:
- We will create a proxy on ui.grid.sara.nl
- You will create a new password for this proxy
- With the python API we will delegate the proxy to Globus Online and activate the grid endpoint and the archive endpoint. 

### Check endpoints
Before we start let us see if all endpoints we need are available.You can get a list of all of your personal endpoints either via the web interface or via the globusconnect personal tools:
```sh
$ ssh <globususer>@cli.globusonline.org endpoint-search --scope my-endpoints
$ ssh <globususer>@cli.globusonline.org endpoint-search --scope all SURFsara
```

You should have the following endpoints:
- Your previously added personal endpoint
- surfsara#dCache_gridftp
- surfsara#archive

### Create a proxy

First we need to create a valid proxy. This can be done on the grid user interface ui.grid.sara.nl.
The proxy is created from your certificate and stored on a dedicated proxy server (px.grid.sara.nl).
The advantage is that once the proxy is created you can delegate this proxy to Globus Online from any computer you wish. 
I.e. we can create the proxy on the user interface machine and then continue working with the python API from your personal computer.

```sh
$ ssh <user>@ui.grid.sara.nl
```
 
To transfer data to the archive, it is sufficient to create a proxy without any further specifications
```sh
$ myproxy-init
```

The grid works with virtual organisations (VO) and we need to specify our VO and create a proxy that contains this information.
```sh
$ startGridSession <VO>
$ myproxy-init --voms <VO> -l <user>
```
You will also be asked to set a password for this proxy. We will need the password to activate the endpoint either via the web interface or the python API.

The output in both cases looks like:

```sh
Your identity: /DC=org/DC=terena/DC=tcs/C=<Country>/O=<Your organisation>/CN=<Your name and e-mail>
Enter GRID pass phrase for this identity:
Creating proxy .................................................................
......................... Done
Proxy Verify OK
Your proxy is valid until: Mon Oct 17 16:20:15 2016
Enter MyProxy pass phrase:
Verifying - Enter MyProxy pass phrase:
A proxy valid for 168 hours (7.0 days) for user <user> now exists on px.grid.sara.nl.
```

Now the proxy is stored on the proxy server at SURFsara px.grid.sara.nl. n the following we will use the python API to dleegate the proxy and activate the endpoints. You do not need to follow these parts on the ui.grid.sara.nl but can switch to any computer where the globus online python API is installed. 

### Activating endpoints with the globus online python API

Open python in interactive mode:

```sh
$ python -i -m globusonline.transfer.api_client.main <user> -p
```
You will be asked for your globus online password, the password you set when creating the globus account, not the password of your grid certificate.
The command creates a TransferAPIClient instance called *api* with the credentials
passed on the command line, which you can use to make requests.

The general command to activate an endpoint is:

```sh
>>> api.endpoint_autoactivate("surfsara#dCache_gridftp")
```
However, since we have not passed information on the proxy, this is going to fail.
To activate **surfsara#dCache_gridftp** we need to gather the follwoing requirements:

```sh
>>> reqs = api.endpoint_activation_requirements("surfsara#dCache_gridftp", type="myproxy")[2]
>>> reqs.set_requirement_value("myproxy", "hostname", "px.grid.sara.nl")
>>> reqs.set_requirement_value("myproxy", "username", "<username>")
>>> reqs.set_requirement_value("myproxy", "passphrase", "<passwd>")
>>> reqs.set_requirement_value("myproxy", "lifetime_in_hours",str(5))
```

This will create a json/python dictionary with the following requirements:

```
>>> reqs
{u'DATA_TYPE': u'activation_requirements', 
u'expires_in': 17681, 
u'auto_activation_supported': True, 
u'activated': True, 
u'length': 5, 
u'expire_time': u'2016-10-10 19:38:18+00:00', 
u'DATA': [
    {
    u'description': u'The hostname of the MyProxy server to request a credential from.', 
    u'DATA_TYPE': u'activation_requirement', u'required': True, u'private': False, 
    u'value': 'px.grid.sara.nl', 
    u'ui_name': u'MyProxy Server', u'type': u'myproxy', u'name': u'hostname'
    }, 
    {
    u'description': u'The username to use when connecting to the MyProxy server.', 
    u'DATA_TYPE': u'activation_requirement', u'required': True, u'private': False, 
    u'value': '<username>', 
    u'ui_name': u'Username', u'type': u'myproxy', u'name': u'username'
    }, 
    {u'description': u'The passphrase to use when connecting to the MyProxy server.', 
    u'DATA_TYPE': u'activation_requirement', u'required': True, u'private': True, 
    u'value': '<passwd>', 
    u'ui_name': u'Passphrase', u'type': u'myproxy', u'name': u'passphrase'
    }, 
    {
    u'description': u"The distinguished name of the MyProxy server, formated with '/' as the separator. 
        This is only needed if the server uses a non-standard certificate and the hostname does not match.", 
        u'DATA_TYPE': u'activation_requirement', u'required': False, u'private': False, u'value': u'', 
        u'ui_name': u'Server DN', u'type': u'myproxy', u'name': u'server_dn'
    }, 
    {
    u'description': u"The lifetime for the credential to request from the server, in hours. 
        Depending on the MyProxy server's configuration, this may not be respected if it's too high. 
        If no lifetime is submitted, the value configured as the default on the  server will be used.", 
    u'DATA_TYPE': u'activation_requirement', u'required': False, u'private': False, 
    u'value': '5', u'ui_name': u'Credential Lifetime (hours)', 
    u'type': u'myproxy', u'name': u'lifetime_in_hours'
    }
], u'oauth_server': None}
```

The initial activation is done by:

```sh
>>> result = api.endpoint_activate("surfsara#dCache_gridftp", reqs)
```
which returns:

```sh
>>> result
(200, 'OK', 
{u'code': u'Activated.MyProxyCredential', 
u'resource': u'/endpoint/surfsara%23dCache_gridftp/activate', 
u'DATA_TYPE': u'activation_result', 
u'expires_in': 17999, 
u'length': 0, 
u'endpoint': u'surfsara#dCache_gridftp', 
u'request_id': u'fLsnzScqG', 
u'expire_time': u'2016-10-10 19:38:18+00:00', 
u'message': u'Endpoint activated successfully using a credential fetched from a MyProxy server.', 
u'DATA': [], 
u'oauth_server': None, 
u'subject': u'/DC=org/DC=terena/DC=tcs/C=NL/O=<organisation>/CN=<name and e-mail>/CN=proxy/CN=proxy/CN=proxy'})
```

Check again the response for the autoactivation:

```sh
>>> api.endpoint_autoactivate("surfsara#dCache_gridftp")
```

This will return a triple containing this information:

```sh
(200, 'OK', {u'code': u'AutoActivated.CachedCredential', u'resource': u'/endpoint/surfsara%23dCache_gridftp/autoactivate', u'DATA_TYPE': u'activation_result', u'expires_in': 42553, u'length': 0, u'endpoint': u'surfsara#dCache_gridftp', u'request_id': u'vdG4gZnI2', u'expire_time': u'2016-09-21 18:16:42+00:00', u'message': u'Endpoint activated successfully using cached credential', u'DATA': [], u'oauth_server': None, u'subject': u'/DC=org/DC=terena/DC=tcs/C=NL/O=SURFsara B.V./CN=Christine Staiger christine.staiger@surfsara.nl/CN=proxy/CN=proxy/CN=proxy'})
```
To be sure that the endpoint is activated you should always check the *code*.
Try to activate an endpoint you do not have access to and compare the output with the output above.

```sh
>>> api.endpoint_activate("cineca#PICO", reqs)
>>> api.endpoint_autoactivate("cineca#PICO")
```

Alternatively, you can also use the web interface of globus and activate the grid and archive endpoints with your proxy and then proceed with transferring data by means of the python API.

**Exercise** Activate *surfsara#archive*.

**Exercise** Inspect the help of the function *endpoint_autoactivate* and activate an endpoint only if the existing activation expires in 1h or less.

**Exercise** Activate your personal endpoint with ths function. 
Note, that you need the **legacy name** of an endpoint to activate it, this name might be different from the **display name**.
Store the **legacy names** of your activated endopints in two pyton variables *source* and *destination*. We will use these two variables in the next section to actually transfer data.

## Transfer between a user interface or laptop to a gridFTP server

We will now transfer data between our personal endpoint and the grid.

```sh
>>> source="<your endpoint>"
>>> destination="surfsara#dCache_gridftp"
```

First, we create a submission ID and store it in a new variable

```sh
>>> code, message, data = api.transfer_submission_id()
>>> submission_id = data["value"]
```

The output has the same structure as the output for the endpoint activation. The actual transfer is stored in an object of the class *Transfer*.

```sh
>>> help(Transfer)
```

To initiate such an object we need a submission ID, the source endpoint, the destination endpoint; we can set a deadline and a synchronisation level (check the help of globus-url-copy) and a label for the transfer.
Globus Online automatically checks the MD5 checksums. However, MD5 is not support by our grid infrastructure. That is why we need to set an additional parameter *verify_checksum=False*.

Our proxy is valid for 12 hours, thus the deadline for the transfer should fall into this time interval:

```sh
>>> import datetime
>>> deadline = datetime.datetime.utcnow() + datetime.timedelta(hours=12)
```

If you already copied e.g. a folder or a larger file setting the sync-level will prohibit you from retransferring data that already exists at the destination. Please consult the globus-url-copy help (`man globus-url-copy`, sync-level number) for more information.

```sh
>>> sync_level=0
>>> label="My first data transfer"
```

Initiate an instance of *Transfer*:

```sh
>>> t = Transfer(submission_id, source, destination, deadline, sync_level, label, verify_checksum=False)
```

What we now need to do, is to add pairs of file source location and file destination location.
Let us create a folder containing some data on the (bash) shell on our personal endpoint:

```sh
$ mkdir TestData
$ for i in {000..010}; do echo "Test file ${i} and some text.">"TestData/File${i}.txt"; done
```
and add it to the transfer object (in the python shell):

```sh
>>> t.add_item("/home/<user>/TestData/", "/pnfs/grid.sara.nl/data/<VO>/<user>/TestData/", recursive=True)
```
You can add as many items to the transfer as you wish. Note that *recursive* is by default set to *false*. You do not need the recursive option when transferring single files.

Start the transfer task:

```sh
>>> code, reason, data = api.transfer(t)
>>> task_id = data["task_id"]
```
**Exercise** Inspect the code and the data. Create some invalid transfers (wrong path, unactivated endpoints , ...) and reinspect the data.

## Monitoring
With the *task_id* you can ask for the status of the transfer

```sh
>>> code, reason, data = api.task(task_id)
```

Inspect the *data*. You can also monitor this transfer in the webinterface of globus.

**Exercise** Write a function that takes as input the data from a *api.transfer(t)* and print the information needed for monitoring in a pretty way. (Which items do you need?)

**Exercise** Write a function that prints the data from *api.task(task_id)* in a nice way such that it simplifies the monitoring.

## 3rd party transfers
Now that we have some data on the gridFTP server of the grid we can transfer this data further to another gridFTP server e.g. the archive.

```sh
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

```sh
python myGO.py 
```


