# dcc-ops

## About

This repository contains our Docker-compose and setup bootstrap scripts used to create a deployment of the [UCSC Genomic Institute's](http://ucsc-cgl.org) Computational Genomics Platform for AWS.  The system is designed to receive genomic data, run analysis at scale on the cloud, and return analyzed results to authorized users.  It uses, supports, and drives development of several key GA4GH APIs and open source projects. In many ways it is the generalization of the [PCAWG](https://dcc.icgc.org/pcawg) cloud infrastructure developed for that project and a potential reference implementation for the [NIH Commons](https://datascience.nih.gov/commons) concept.

## Components

The system has components fulfilling a range of functions, all of which are open source and can be used independently or together.

![Computational Genomics Platform architecture](docs/dcc-arch.png)

These components are setup with the install process available in this repository:

* [Boardwalk](boardwalk/README.md): our file browsing portal on top of Redwood

These are related projects that are either already setup and available for use on the web or are used by components above.

* [Dockstore](http://dockstore.org): our workflow and tool sharing platform
* [Toil](https://github.com/BD2KGenomics/toil): our workflow engine, these workflows are shared via Dockstore

## Installing the Platform

These directions below assume you are using AWS.  We will include additional cloud instructions as `dcc-ops` matures.

### Collecting Information

Make sure you have:

* you know what region you're running in e.g. `us-west-2`

### Starting an AWS VM

Use the AWS console or command line tool to create a host. For example:

* Ubuntu Server 16.04
* r4.xlarge
* 250GB disk

We will refer to this as the host VM throughout the documentation below and it is the machine running all the Docker containers for each of the components below.

You should make a note of your security group name and ID and ensure you can connect via ssh.

**NOTE:** We have had problems when uploading big files to Virginia (~25GB). If possible, set up your AWS anywhere else but Virginia.

### AWS Tasks

Make sure you do the following:

* assign an Elastic IP (a static IP address) to your instance
* open inbound ports on your security group
    * 80 <- world
    * 22 <- world
    * 443 <- world
    * all TCP <- the elastic IP of the VM (Make sure you add /32 to the Elastic IP)
    * all TCP <- the security group itself

#### Adding private SSH key to your VM

Add your private ssh key under `~/.ssh/<your_key>.pem`, this is typically the same key that you use to SSH to your host VM, regardless it needs to be a key created on the AWS console so Amazon is aware of it. Then do `chmod 400 ~/.ssh/<your_key>.pem` so your key is not publicly viewable.

#### TODO:

* Guide on choosing AWS instance type... make sure it matches your AMI.
* AMI, use an ubuntu 16.04 base box, you can use the official Ubuntu release.  You may need to make your own AMI with more storage! Needs to be in your region!  You may want to google to start with the official Ubuntu images for your region.

### Setup for Boardwalk

Here is a summary of what you need to do. See the Boardwalk [README](boardwalk/README.md) for details.

#### Create a Google Oauth2 app

You need to create a Google Oauth2 app to enable Login and token download from the dashboard. If you don't want to enable this on the dashboard during installation, simply enter a random string when asked for the *Google Client ID* and the *Google Client Secret*. You can consult [here](http://bitwiser.in/2015/09/09/add-google-login-in-flask.html#creating-a-google-project) under "Creating A Google Project" if you want to read more details. Here is a summary of what you need to do:

* Go to [Google's Developer Console](https://console.developers.google.com/).
* On the upper left side of the screen, click on the drop down button.
* Create a project by clicking on the plus sign on the pop-up window.
* On the next pop up window, add a name for your project.
* Once you create it, click on the "Credentials" section on the left hand side.
* Click on the "OAuth Consent Screen". Fill out a product name and choose an email for the Google Application. Fill the rest of the entries as you see fit for your purposes, or leave them blank, as they are optional. Save it.
* Go to the "Credentials" tab. Click on the "Create Credentials" drop down menu and choose "OAuth Client ID".
* Choose "Web Application" from the menu. Assign it a name.
* Under "Authorized JavaScript origins", enter `http://<YOUR_SITE>`. Press Enter. Add a second entry, same as the first one, but use *https* instead of *http*
* Under "Authorized redirect URIs", enter `http://<YOUR_SITE>/gCallback`. Press Enter. Add a second entry, same as the first one, but use *https* instead of *http*
* Click "Create". A pop up window will appear with your Google Client ID and Google Client Secret. Save these. If you lose them, you can always go back to the Google Console, and click on your project; the information will be there.

Please note: at this point, the dashboard only accepts login from emails with a 'ucsc.edu' domain. In the future, it will support different email domains.

### Running the Installer

Once the above setup is done, clone this repository onto your server and run the bootstrap script

    # note, you may need to checkout the particular branch or release tag you are interested in...
    git clone https://github.com/BD2KGenomics/dcc-ops.git && cd dcc-ops && sudo bash install_bootstrap

#### Installer Question Notes

The `install_bootstrap` script will ask you to configure each service interactively.

* Boardwalk
  * Install in prod mode
  * On question `What is your Google Client ID?`, put your Google Client ID. See [here](http://bitwiser.in/2015/09/09/add-google-login-in-flask.html#creating-a-google-project)
  * On question `What is your Google Client Secret?`, put your Google Client Secret. See [here](http://bitwiser.in/2015/09/09/add-google-login-in-flask.html#creating-a-google-project)
* Common
  * Installing in `dev`mode will use letsencrypt's staging service, which won't exhaust your certificate's limit, but will install fake ssl certificates. `prod` mode will install official SSL certificates.  
  
  
Once the installer completes, the system should be up and running. Congratulations! See `docker ps` to get an idea of what's running.

## Post-Installation

### TODO

Here are things we need to explain how to do post install:

* first of all, how to go to the website and confirm things are working e.g. https://ops-dev.ucsc-cgl.org or whatever the domain name is
* user log in via google, retrieve token
* Get the reference data used by the RNASeq-CGL pipeline:
*    Instructions for downloading reference data for RNASeq-CGL are located here: https://github.com/BD2KGenomics/toil-rnaseq/wiki/Pipeline-Inputs 
* Test data inputs for the RNASeq-CGL pipeline are locate here: https://github.com/UCSC-Treehouse/pipelines/tree/master/samples 
* update the decider manually to point to these new reference URLs (via exec into the Docker container)
* get sample fastq data
    * ...
* upload sample fastq data
* trigger indexing so you can immediately see fastq data in the file browser e.g. https://ops-dev.ucsc-cgl.org/file_browser.html, `sudo docker exec -it boardwalk_dcc-metadata-indexer_1 bash -c "/app/dcc-metadata-indexer/cron.sh"`
* monitor running of Consonance logs and worker nodes to see running data
* download RNASeq-CGL analysis results from the portal

### Confirm Proper Function

To test that everything installed successfully, you can run `cd test && ./integration.sh`. This will do an upload and download with core-client and check the results.

### Running RNA-Seq Analysis on Sample Data

To do RNA-Seq Analysis, you must first upload reference files to Redwood. You can obtain the reference files by running from within dcc-ops:

```
reference/download_reference.sh
```

This will download the files under `reference/samples`. You can then use the core client to do a spinnaker upload as described previously and use the _manifest.tsv_ within the `reference` folder. 

Once you have successfully uploaded the reference files, you can start submitting fastq files to redwood to run analysis on them. See the help section on the file browser for more information on the template. Use `RNA-Seq` or `scRNA-Seq` when filling out the *Submitter Experimental Design* column on your manifest.

### Troubleshooting

If something goes wrong, you can [open an issue](https://github.com/BD2KGenomics/dcc-ops/issues/new) or [contact a human](https://github.com/BD2KGenomics/dcc-ops/graphs/contributors).

### Tips

* This [blog post](https://www.digitalocean.com/community/tutorials/how-to-remove-docker-images-containers-and-volumes) is helpful if you want to clean up previous images/containers/volumes.

### To Do

* the bootstrapper should install Java, Dockstore CLI
* Consonance config.template includes hard-coded Consonance token, needs to be generated and written to .env file just like Beni does
