設定你的 GCP 環境
---------------------------

您可以從 Cloud Shell 進行以下指令

[開啟 Cloud Shell](https://console.cloud.google.com/?cloudshell=true)

### Configure IAM

建立一個 IAM (Identity and Access Management) 服務帳號，將權限給 Jenkins. This account enables Jenkins to store data in Cloud Storage and launch instances in Compute Engine. Jenkins runs your builds in ephemeral instances and stores build artifacts in Cloud Storage.

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
    
