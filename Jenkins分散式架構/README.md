# 開 Jenkins 自動化 (用 GCP 建立 Jenkins帳號)

Update 26Orc2021, 在 GCP Project 內, 建一個IAM, 用這個帳號, 從Market開1台 Jenkin。


Using Jenkins for distributed builds on Compute Engine
======================================================

 
https://cloud.google.com/architecture/nusing-jenkins-for-distributed-builds-on-compute-engine

This tutorial shows you how to:

*   Create a Jenkins continuous integration system to run your builds using on-demand Jenkins agents in Compute Engine.
*   Store your build artifacts in Cloud Storage.
*   Apply a lifecycle policy to move older build artifacts in Cloud Storage to less expensive storage options.

Architecture
------------

The following diagram outlines the tutorial architecture.

![Architecture that shows how a service account pushes artifacts through Compute Engine to Cloud Storage.](https://cloud.google.com/architecture/images/jenkins-ce-architecture.svg)

In the diagram, a service account is added to Jenkins for it to be able to create agent instances and push artifacts to Cloud Storage for long-term storage. Jenkins provisions the instances on the fly as it runs builds. As the build artifacts get older, they move through various storage classes to limit their retention cost.

Objectives
----------

*   Create a base image with [Packer](https://www.packer.io/docs) for running your Jenkins builds.
*   Provision Jenkins using [Cloud Marketplace](https://cloud.google.com/marketplace/docs).
*   Configure Jenkins to deploy ephemeral build agents.
*   Upload build artifacts to Cloud Storage.
*   Configure lifecycle policies to optimize your Cloud Storage costs.

Before you begin
----------------

1.  In the Google Cloud Console, on the project selector page, select or create a Google Cloud project.
    
    **Note**: If you don't plan to keep the resources that you create in this procedure, create a project instead of selecting an existing project. After you finish these steps, you can delete the project, removing all resources associated with the project.
    
    [Go to project selector](https://console.cloud.google.com/projectselector2/home/dashboard)
    
2.  Make sure that billing is enabled for your Cloud project. [Learn how to confirm that billing is enabled for your project](https://cloud.google.com/billing/docs/how-to/modify-project#confirm_billing_is_enabled_on_a_project).
    
3.  Enable the Compute Engine API.
    
    [Enable the API](https://console.cloud.google.com/flows/enableapi?apiid=compute_component)
    

Setting up your environment
---------------------------

In this section, you configure the infrastructure and identities required to complete the tutorial. You execute the rest of the tutorial from inside Cloud Shell.

[Open Cloud Shell](https://console.cloud.google.com/?cloudshell=true)

### Configure IAM

Create an Identity and Access Management (IAM) service account to delegate permissions to Jenkins. This account enables Jenkins to store data in Cloud Storage and launch instances in Compute Engine. Jenkins runs your builds in ephemeral instances and stores build artifacts in Cloud Storage.

#### Create a service account

1.  Create the service account itself:
    
    <pre>gcloud iam service-accounts create jenkins --display-name jenkins</pre>
    
2.  Store the service account email address and your current Google Cloud project ID in environment variables for use in later commands:
    
    <pre>export SA_EMAIL=$(gcloud iam service-accounts list \
        --filter="displayName:jenkins" --format='value(email)')
    export PROJECT=$(gcloud info --format='value(config.project)')</pre>
    
3.  Bind the following roles to your service account:
    
    <pre>
    gcloud projects add-iam-policy-binding $PROJECT \
        --role roles/storage.admin --member serviceAccount:$SA\_EMAIL
    gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.instanceAdmin.v1 \
        --member serviceAccount:$SA\_EMAIL
    gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.networkAdmin \
        --member serviceAccount:$SA\_EMAIL
    gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.securityAdmin \
        --member serviceAccount:$SA\_EMAIL
    gcloud projects add-iam-policy-binding $PROJECT --role roles/iam.serviceAccountActor \
        --member serviceAccount:$SA\_EMAIL
    <pre>

#### Download the service account key

Now that you've granted the service account the appropriate permissions, you need to create and download its key. Keep the key in a safe place. You'll use it later step when you configure the JClouds plugin to authenticate with the Compute Engine API.

1.  Create the key file:
    
    <pre>gcloud iam service-accounts keys create jenkins-sa.json --iam-account $SA_EMAIL</pre>
    
2.  In Cloud Shell, click **More** _more\_vert_, and then click **Download file**.
    
3.  Type `jenkins-sa.json`.
    
4.  Click **Download** to save the file locally.
    

### Create a Jenkins agent image

Next, you create a reusable Compute Engine image that contains the software and tools needed to run as a Jenkins executor.

#### Create an SSH key for Cloud Shell

Later in this tutorial, you use [Packer](https://www.packer.io/intro#what-is-packer) to build your images, which requires the `ssh` command to communicate with your build instances. To enable SSH access, create and upload an SSH key in Cloud Shell:

1.  Create a SSH key pair. If one already exists, this command uses that key pair; otherwise, it creates a new one:
    
    <pre>ls ~/.ssh/id\_rsa.pub || ssh-keygen -N ""</pre>
    
2.  Add the Cloud Shell public SSH key to your project's [metadata](https://cloud.google.com/compute/docs/instances/adding-removing-ssh-keys#project-wide):
    
    <pre>gcloud compute project-info describe \\
        --format=json | jq -r '.commonInstanceMetadata.items\[\] | select(.key == "ssh-keys") | .value' > sshKeys.pub
    echo "$USER:$(cat ~/.ssh/id\_rsa.pub)" >> sshKeys.pub
    gcloud compute project-info add-metadata --metadata-from-file ssh-keys=sshKeys.pub
    </pre>

#### Create the baseline image

The next step is to use [Packer](http://packer.io/) to create a baseline virtual machine (VM) image for your build agents, which act as ephemeral build executors in Jenkins. The most basic Jenkins agent only requires Java to be installed. You can customize your image by adding shell commands in the `provisioners` section of the Packer configuration or by adding other [Packer provisioners](https://www.packer.io/docs/provisioners). PackerGCP參考(Packer - Google Compute Builder) => https://www.packer.io/docs/builders/googlecompute

1.  In Cloud Shell, download and unpack the most recent release of Packer. The following example uses Packer 1.6.6. You can [check the Hashicorp website](https://www.packer.io/downloads) to see if there's a more recent version:
    
    <pre>
    wget https://releases.hashicorp.com/packer/1.6.6/packer\_1.6.6\_linux\_amd64.zip
    unzip packer\_1.6.6\_linux\_amd64.zip
    </pre>
    
2.  Create the configuration file for your Packer image builds:
    
    <pre>

    export PROJECT=$(gcloud info --format='value(config.project)')
cat > jenkins-agent.json <<EOF
{
    "builders": [
    {
        "type": "googlecompute",
        "project_id": "$PROJECT",
        "source_image_family": "ubuntu-2004-lts",
        "source_image_project_id": "ubuntu-os-cloud",
        "zone": "us-central1-a",
        "disk_size": "10",
        "image_name": "jenkins-agent-{{timestamp}}",
        "image_family": "jenkins-agent",
        "ssh_username": "skyap"
    }
    ],
    "provisioners": [
    {
        "type": "shell",
        "inline": ["sudo apt-get update && sudo apt-get install -y default-jdk"]
    }
    ]
}
EOF
    </pre>
    
3.  Build the image by running Packer:
    
    <pre>./packer build jenkins-agent.json</pre>
    
    When the build completes, the name of the disk image is displayed with the format `jenkins-agent-[TIMESTAMP]`, where `[TIMESTAMP]` is the epoch time when the build started.
    
    \==> Builds finished. The artifacts of successful builds are:
    --> googlecompute: A disk image was created: jenkins-agent-1612997575
    

Installing Jenkins
------------------

In this section, you use [Cloud Marketplace](https://cloud.google.com/marketplace) to provision a Jenkins instance. You customize this instance to use the agent image you created in the previous section.

1.  Go to the [Cloud Marketplace solution for Jenkins](https://console.cloud.google.com/marketplace/details/bitnami-launchpad/jenkins?q=jenkins&_ga=2.160297531.705065107.1635110664-1169492087.1632552470).
    
2.  Click **Launch**.
    
3.  Change the **Machine Type** field to **4 vCPUs 15 GB Memory, n1-standard-4**.
    
    ![Machine type selection for Jenkins deployment.](https://cloud.google.com/architecture/images/jenkins-ce-machine-type.png)
    
4.  Click **Deploy** and wait for your Jenkins instance to finish being provisioned. When it is finished, you will see:
    
    ![Jenkins has been deployed.](https://cloud.google.com/architecture/images/jenkins-ce-deploy.png)
    
5.  Open your Jenkins instance in the browser by clicking the **Site Address** link.
    
6.  Log in to Jenkins using the **Admin user** and **Admin password** displayed in the details pane.
    
    ![Details pane with credentials and other deployment details.](https://cloud.google.com/architecture/images/jenkins-ce-details-pane.png)
    

Your Jenkins instance is now ready to use.

Configuring Jenkins plugins
---------------------------

Jenkins requires plugins to create on-demand agents in Compute Engine and to store artifacts in Cloud Storage. You need to install and configure these plugins.

### Install plugins

1.  In the Jenkins UI, select **Manage Jenkins**.
2.  Click **Manage Plugins**.
3.  Click the **Available** tab.
4.  Use the **Filter** bar to find the following plugins and select the boxes next to them:
    
    *   Compute Engine plugin
    *   Cloud Storage plugin
    
    The following image shows the Cloud Storage plugin selected:
    
    ![Cloud Storage plugin.](https://cloud.google.com/architecture/images/jenkins-ce-filter-plugins.png)
    
5.  Click **Download now and install after restart**.
    
6.  Click the **Restart Jenkins when installation is complete and no jobs are running** checkbox. Jenkins restarts and completes the plugin installations.
    

### Create plugin credentials

You need to create `Google Credentials` for your new plugins:

1.  Log in to Jenkins again, and click **Jenkins**.
2.  Click **Credentials**.
3.  Click **System**.
4.  In the main pane of the UI, click **Global credentials (unrestricted)**.
5.  Create the Google credentials:
    
    1.  Click **Add Credentials**.
    2.  Set **Kind** to **Google Service Account from private key**.
    3.  In the **Project Name** field, enter your Google Cloud project ID.
    4.  Click **Choose file**.
    5.  Select the `jenkins-sa.json` file that you previously downloaded from Cloud Shell.
    6.  Click **OK**.
        
        ![JSON key credentials.](https://cloud.google.com/architecture/images/jenkins-ce-google-credentials.png)
        
6.  Click **Jenkins**.
    

### Configure the Compute Engine plugin

Configure the Compute Engine plugin with the credentials it uses to provision your agent instances.

1.  Click **Manage Jenkins**.
2.  Click **Configure System**.
3.  Click **Add a new Cloud**.
4.  Click **Compute Engine**.
5.  Set the following settings and replace `[YOUR_PROJECT_ID]` with your Google Cloud project ID:
    
    *   **Name**: `gce`
    *   **Project ID**: `[YOUR_PROJECT_ID]`
    *   **Instance Cap**: `8`
6.  Choose the service account from the **Service Account Credentials** drop-down list. It is listed as your Google Cloud project ID.
    

### Configure Jenkins instance configurations

Now that the Compute Engine plugin is configured, you can configure Jenkins instance configurations for the various build configurations you'd like.

1.  On the **Configure System** page, click **Add** _add_ for **Instance Configurations**.
2.  Enter the following **General** settings:
    
    *   **Name**: `ubuntu-2004`
    *   **Description**: `Ubuntu agent`
    *   **Labels**: `ubuntu-2004`
3.  Enter the following for **Location** settings:
    
    *   **Region<**: `us-central1`
    *   **Zone**: `us-central1-f`
4.  Click **Advanced**.
    
5.  For **Machine Configuration**, choose the **Machine Type** of **n1-standard-1**.
    
6.  Under **Networking**, choose the following settings:
    
    *   **Network**: Leave at default setting.
    *   **Subnetwork**: Leave at default setting.
    *   Select **Attach External IP?**.
7.  Select the following for **Boot Disk** settings:
    
    *   For **Image project**, choose your Google Cloud project.
    *   For **Image name**, select the image you built earlier using Packer.
8.  Click **Save** to persist your configuration changes.
    
    ![Compute Engine configurations for Jenkins.](https://cloud.google.com/architecture/images/jenkins-gce-config.png)
    

Creating a Jenkins job to test the configuration
------------------------------------------------

Jenkins is configured to automatically launch an instance when a job is triggered that requires an agent with the `ubuntu-2004` label. Create a job that tests whether the configuration is working as expected.

1.  Click **Create new job** in the Jenkins interface.
2.  Enter `test` as the item name.
3.  Click **Freestyle project**, and then click **OK**.
4.  Select the **Execute concurrent builds if necessary** and **Restrict where this project can run** boxes.
5.  In the **Label Expression** field, enter `ubuntu-2004`.
6.  In the **Build** section, click **Add build step**.
7.  Click **Execute Shell**.
8.  In the command box, enter a test string:
    
    echo "Hello world!"
    
    ![Hello World typed in the command box for Jenkins.](https://cloud.google.com/architecture/images/jenkins-ce-hello-world.png)
    
9.  Click **Save**.
    
10.  Click **Build Now** to start a build.
    
    ![Build Now.](https://cloud.google.com/architecture/images/jenkins-ce-build-now.png)
    

