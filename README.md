bash-onedrive-upload
====================

Upload files to [Microsoft OneDrive](https://onedrive.live.com) via linux command line using the [REST API](http://onedrive.github.io/).

Use `git clone --recursive` to checkout the repository including all required submodules.

If you download the contents of this repository as .zip, you also need to manually download the [bash-json-parser](https://github.com/fkalis/bash-json-parser/archive/master.zip) and extract it into `./libs/json`, because submodules are not included in the ZIP file.

Prerequisites
-------------

- `curl` is used for accessing the API.
- `grep` is used for filtering the parsed json output.
- `cut` is used for value extraction from the filtered json output.
- `xargs` is used for multi-threaded file uploads.
- `dd` is used for chunked file uploads.
- `stat` is used to determine the filesize.

Getting started (OneDrive Personal)
---------------

Before you can use this tool, create an application in the [https://apps.dev.microsoft.com/#/appList/create/sapi](https://apps.dev.microsoft.com/#/appList/create/sapi) to generate your custom Client ID and Client secret. Notice, that you need to add a new platform of type `Mobile application`. If you have not created Live SDK apps before you will not see menu to add it, because they are deprecated, instead use link above (enter app name & press "create application").


When you are already logged in into your Microsoft account, you will be redirected to the right form. In case you sign in first, pay attention to create your application in the `Live SDK applications` section, **not** in the `Converged applications` section, as the newer application type is not (yet) supported by `onedrive-authorize`. Please also ensure that the Application ID (Client ID) does not contain any dashes.


Afterwards your overview should show your freshly created credentials in the form

    Application ID: 00000000A2B3C495
    Applications secrets: qOFCYaZKjm6e13aq3fdGiNz

Please insert these values in the matching variables in `onedrive.cfg`:

    export api_client_id="00000000A2B3C495"
    export api_client_secret="88qC3kX2Cbd0tXV2sBnqYbS321abcDEF"

After the initial configuration you must authorize the app to use your OneDrive account. Run

    $ ./onedrive-authorize

and follow the steps. You will need a web browser.

After the authorization process has successfully completed you can upload files.

Getting started (OneDrive for Business)
---------------

Before you can use this tool, create an application in the [Microsoft Azure Management Portal](https://manage.windowsazure.com) to generate your custom Client ID and Reply URL.

1. Navigate to *Manage Azure Active Directory*.
2. Navigate to the *App registrations* tab on the left pane, and then press *New registration*. Here, you can type any name that you want for your application. But you **must** choose the *MultiTenant* option, and **not** the *SingleTenant* option. This is very important, as the MultiTenant option will allow the scripts to easily find the application on OneDrive.
3. On the overview page of your created application, please note down the Client ID (which can also be referred to as the Application ID).
4. Navigate to the *Authentication* tab on the left pane, and then press *Add a platform*. Here, we should choose *Mobile and desktop applications*, because our scripts will run locally on a desktop environment. Now, select one of the given *Redirect URIs*, and note it down, because that will be used as the Reply URL for the scripts.

Afterwards you should have these two components of authorization process:

    Client ID: a66d1076-4c04-4a33-3bb2-2578c4891886
    Reply URL: https://login.live.com/oauth20_desktop.srf

Please insert these values in the matching variables in `onedriveb.cfg`:

    export api_client_id="a66d1076-4c04-4a33-3bb2-2578c4891886"
    export api_reply_url="https://login.live.com/oauth20_desktop.srf"

After the initial configuration you must authorize the app to use your OneDrive for Business account. You will only need to do this once, because later on, the scripts will use the refresh token instead. To authorize the app, run:

    $ ./onedriveb-authorize

and follow the steps. You will need a web browser, which can be run from anywhere (and not necessarily on the same machine that the upload scripts will run in).

After the authorization process has successfully completed you can upload files.
If you see error `AADSTS70002: Error validating credentials. AADSTS50012: Invalid client secret is provided`, try to add a new key and use new secret.

Usage
-----

To upload a single file simply type

    # OneDrive Personal
    $ ./onedrive-upload file1

    # OneDrive for Business
    $ ./onedriveb-upload file1

For uploading a large file, you should use the debug flag to get more progress info

    # OneDrive Personal
    $ ./onedrive-upload -d file1

    # OneDrive for Business
    $ ./onedriveb-upload -d file1

You can also upload multiple files, either by explicitly specifying each one

    # OneDrive Personal
    $ ./onedrive-upload file1 file2

    # OneDrive for Business
    $ ./onedriveb-upload file1 file2

or just use wildcards (globbing)

    # OneDrive Personal
    $ ./onedrive-upload file*.png

    # OneDrive for Business
    $ ./onedriveb-upload file*.png

It is also possible to recursively upload a whole folder

    # OneDrive Personal
    $ ./onedrive-upload /path/to/folder

    # OneDrive for Business
    # Not yet supported

If this folder contains hidden files (files starting with a dot) and you want to include them, just type

    # OneDrive Personal
    $ ./onedrive-upload --dotfiles /path/to/folder

    # OneDrive for Business
    # Not yet supported

You can also specify a destination folder relative to the root folder configured in `onedrive.cfg`:

    # OneDrive Personal
    $ ./onedrive-upload -f "relative/path" file

    # OneDrive for Business
    $ ./onedriveb-upload -f "relative/path" file

This command will automatically determine all of the needed folder ids and recursively create all subfolders that do not yet exist.

If you need your file to be uploaded with a different filename, you can activate the renaming mode:

    # OneDrive Personal
    $ ./onedrive-upload -r ./file1.txt renamed_file1.txt ./file2.txt renamed_file2.txt

    # OneDrive for Business
    $ ./onedriveb-upload -r ./file1.txt renamed_file1.txt ./file2.txt renamed_file2.txt

Be aware that for each file you specify you must provide the remote filename as the subsequent parameter. This feature can lead to an unexpected behavior when combined with wildcards (globbing) because the pathname expansion is performed by bash before the execution of the script. Also do not use this when recursively uploading folders.

Configuration
-------------

### Specify an alternate root folder for uploads (OneDrive Personal)

If you want to use a folder other than the root folder of your OneDrive as your upload root folder, just enter the human readable, slash separated path into

    export api_root_folder="/path/to/folder"

You can also use any of the [special folders](https://dev.onedrive.com/items/special_folders.htm) provided by the API by simply using the combination

    export api_drive_resource="special/approot"
    export api_root_folder="/path/relative/to/special/folder"

### Specify an alternate root folder for uploads (OneDrive for Business)

If you want to use a folder other than the root folder of your OneDrive as your upload root folder, you need to retrieve its unique id by using this approach:

    Upload in debug mode any file to non-existent folder, which will be used as root folder.

    ./onedriveb-upload -d -f Backup testfile
    2016-08-26 09:33:56 Searching for 'Backup' in ''
    2016-08-26 09:33:56 Creating folder 'Backup' in ''
    2016-08-26 09:33:58 api_folder_id is now '01KB37ZVWEMG6F4SS6TVGZAM7ACZKOEERY'
    2016-08-26 09:33:58 Size of testfile is less than or equal to 104857600 bytes, will use simple upload
    2016-08-26 09:34:01 Uploading 'testfile' as 'testfile' into 01KB37ZVWEMG6F4SS6TVGZAM7ACZKOEERY

    Copy `api_folder_id` and place it in your `onedriveb.cfg`

    export api_root_folder="items/01KB37ZVWEMG6F4SS6TVGZAM7ACZKOEERY"

You can also specify any of the [special folders](https://dev.onedrive.com/items/special_folders.htm) provided by the API by simply using

    export api_root_folder="special/approot"
    export api_root_folder="special/documents"
    export api_root_folder="special/photos"
    ...

The default is to use

    export api_root_folder="root"

which identifies the root folder of your account.

Keep in mind that this will be the new root folder of your OneDrive as seen by this script. The script does *not* support escaping the root by using `../`.

### Less permissions for uploads

If you don't want the script to have access to all your folders and files you can request less permissions during the authorization process. Simply change the value

    export api_permissions="onedrive.readwrite"

to

    export api_permissions="onedrive.approot"

and run the authorization process. You might need to remove the apps permissions prior to that ([Link](https://account.live.com/consent/Manage)) if you have already requested `onedrive.readwrite` permissions.

### Number of simultaneous uploads

If the script does not fully utilize your bandwidth, you can maybe speed things up a little bit by increasing the value of `max_upload_threads`.

When you start the upload of more than one file, the script will start up to `max_upload_threads` parallel uploads, but only one thread per file.

### Configure threshold for session based upload

For small files the script uses the [simple upload](https://dev.onedrive.com/items/upload_put.htm) of the api. For files that are larger than 4 MiB the script uses the [session based upload](https://dev.onedrive.com/items/upload_large_files.htm).

If you want to use the session based upload for files smaller than 4 MiB you can change the value of `max_simple_upload_size` to any positive value smaller than 4194304:

    export max_simple_upload_size=2097152

You can also change the value of `max_chunk_size` to any positive value. Note that the smaller the chunk size, the less efficient the upload will be, thus can lead to lower upload speed. But with larger chunk size, you might risk of not receiving any outputs at all, since it can take a very long time for a chunk to finish.

if you want to use smaller or larger chunks than the default:

    export max_chunk_size=62914560

Development
-----------

Some good articles to read before starting on development:
- [Authentication with Microsft Graph API](https://docs.microsoft.com/en-us/onedrive/developer/rest-api/getting-started/graph-oauth)
- [Simple upload of small files](https://docs.microsoft.com/en-us/onedrive/developer/rest-api/api/driveitem_put_content)
- [Chucked upload of large files](https://docs.microsoft.com/en-us/onedrive/developer/rest-api/api/driveitem_createuploadsession)

In order to extend the repository with more features, please consult Microsoft documentation:
- [https://docs.microsoft.com/en-us/onedrive/developer/rest-api/](https://docs.microsoft.com/en-us/onedrive/developer/rest-api/)


Contributors
------------

- [rootik](https://github.com/rootik) (OneDrive for Business support)
- [huming2207](https://github.com/huming2207) (OSX support)
- [laufi](https://github.com/laufi)

Thank you all for your valuable contributions.
