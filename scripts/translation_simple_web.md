# Google Cloud Fundamentals: Getting Started with Cloud Storage and Cloud SQL

## Objectives

- Create a Cloud Storage bucket and place an image into it.
- Create a Cloud SQL instance and configure it.
- Connect to the Cloud SQL instance from a web server.
- Use the image in the Cloud Storage bucket on a web page.

### Task 1: Deploy a web server VM instance

1.  Setup your cloud shell environment
    Open your Cloud shell terminal. Once the initialization is complete we will be ready to start.

      ```
      export PROJECT_ID=abdelrhman-allam000000000
      ```

    - Next, tell `gcloud` about our project,

      ```
      gcloud config set project $PROJECT_ID
      ```

    - If you are prompted to authorize, allow that option.

2.  Create VM instance
    We are going to create a vm instance in our `us-central1-a` zone and will use a `debian-9-stretch` image, run:

    ```
    gcloud compute instances create cutomerhost \
    --zone us-central1-a \
    --machine-type e2-medium \
    --image-project debian-cloud \
    --image debian-9-stretch-v20200910 \
    --subnet default \
    --metadata startup-script="apt-get update; apt-get install apache2 php php-mysql -y; service apache2 restart;"
    ```

    - Our VM name is `cutomerhost`
    - The `--machine-type` specifies that we want to use a `e2-medium` machine which gives us a `2 vCPUs, 4 GB memory` configuration for our VM.
    - The `--image-project` specifies the Google Cloud project against which all image and image family references will be resolved. For our instance we are going to use `debian-cloud` project and our `--image` of choice is `debian-9-stretch-v20200910`
    - The `--subnet` flag specifies the subnet the vm instance will be part of. For our instance we set it to use the `default` subnet interface. By default new projects start with a default network (an auto mode VPC network) that has one subnetwork (subnet) in each region, the subnet name is `default`.
    - `--metadata startup-script` is used to add a start up script to the VM. The script `installs php, apache2, php-mysql` and restarts apache2.

3.  Edit firewall rules
    We are going to `allow HTTP` traffic to our VM instance, run:

         ```
         gcloud compute firewall-rules create default-allow-http \
         --direction=INGRESS \
         --priority=1000 \
         --network=default \
         --action=ALLOW \
         --rules=tcp:80 \
         --source-ranges=0.0.0.0/0 \
         --target-tags=http-server
         ```

    - We are creating a firewall rule and named it `default-allow-http`.
    - We have specified the direction using the `--direction` flag. The set value is `INGRESS` (could have also typed `IN`), This allows ingress traffic to our instance.
    - `--priority` value must an integer between 0 and 65535, both inclusive. It defines the Relative priority which determines precedence of conflicting rules, e.g if a rule a another firewall rule is set that with rules that might conflict with te ones set here then the rule with a lower priority values takes precedence.
    - `--network` we are attaching to rule our `default` network.
    - `--action` flag is set to `ALLOW` which affects how `--rules` value is treated. For this instance we are allowing `tcp` protocol connections over port `80`.
    - `--source-ranges` which is a list of IP address blocks that are allowed to make inbound connections that match the firewall rule to the instances on the network. For our case we allow all incoming connections from inside or outside the network.
    - `--target-tags` this flag value is a list of instance tags indicating the set of instances on the network which may accept connections that match the firewall rule.

    To apply the firewall rule to our VM instance, run:

    ```
    gcloud compute instances add-tags cutomerhost --tags http-server
    ```

### Task 2: Create a Cloud Storage bucket using the gsutil command line

1. Create a storage bucket
   To create a storage bucket in US location, run:

   ```
   gsutil mb -l US gS://$DEVSHELL_PROJECT_ID
   ```

   - `DEVSHELL_PROJECT_ID` is a shell environment variable that is already set which is equal to the our project ID. The name of our bucket will then be equal to that value.

2. Copy image to our storage bucket.
   We will copy an image hosted on a different cloud storage bucket into our bucket, run:

   ```
   gsutil cp gs://cloud-training/gcpfci/my-excellent-blog.png gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png
   ```

   - The command takes the item to copy `gs://cloud-training/gcpfci/my-excellent-blog.png` and the the location to copy to `gS://$DEVSHELL_PROJECT_ID/my-excellent-blog.png`.

3. Verify item was copied.
   To check that the image was copied to our storage bucket, run:

   ```
   gsutil ls -l gs://$DEVSHELL_PROJECT_ID/
   ```

   - The output should be show that include the details of the our bucket content and return on object i.e our image.

4. Make image accessible public
   To make the image publicly available to everyone we wil adjust the access control for our image, run:

   ```
   gsutil acl ch -u allUsers:R gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png
   ```

### Task 3: Create the Cloud SQL instance

1. Create a `MySQL cloud SQL instance`
   We are going to create MySQL instanace `wbsite-db` with root password set to `mypassword`and in the`us-central1-a` zone.

   ```
   gcloud sql instances create wbsite-db  --root-password mypassword  --zone us-central1-a --database-version MYSQL_5_6
   ```

2. Create a user in our instance
   We are going to add a user account to the `wbsite-db` instance, run:

   ```
   gcloud sql users create root --instance wbsite-db --host % --password mypassowrd
   ```

   - `root` is the username
   - `--instance` is the sql instance to which the user will belong
   - `--host` is the Cloud SQL user's host name expressed as a specific IP address or address range. The `%` denotes an unrestricted host name.
   - `--password` the password the user will use to access the sql instance.

3. Update authorized networks
   We are going to add the external IP address for our VM into sql instance authorized networks list. First we wil grab the `external IP` for our `cutomerhost` vm, run:

   ```
   export cutomerhost_xip=$(gcloud compute instances describe cutomerhost --zone us-central1-a --format 'get(networkInterfaces[0].accessConfigs[0].natIP)')
   ```

   - We are saving the `cutomerhost` vm external IP to `cutomerhost_xip` external environment variable. We will reference it later.

   To add the IP address to the `blog-db` instance authorized networks, run:

   ```
   gcloud sql instances patch blog-db --authorized-networks $cutomerhost_xip/32 --quiet
   ```

   - The command adds our vm external IP into the authorized address on our SQL instance.

### Task 4: Configure an application in a Compute Engine instance to use Cloud SQL

1. SSH into `cutomerhost` vm

```
gcloud compute ssh cutomerhost --zone us-central1-a --quiet
```

2. Navigate to the web server index file
   Switch the command prompt path server default html directory, run:

   ```
   cd /var/www/html
   ```

3. Open the `index.php` to edit it
   Using nano lets to create and edit the `index.php` file, run:

   ```
   sudo nano index.php
   ```

   Paste in the following content into the file:

   ```php
   <html>
   <head><title>Welcome to my excellent blog</title></head>
   <body>
   <h1>Welcome to my excellent blog</h1>
   <?php
   $dbserver = "CLOUDSQLIP";
   $dbuser = "root";
   $dbpassword = "root";
   // In a production blog, we would not store the MySQL
   // password in the document root. Instead, we would store it in a
   // configuration file elsewhere on the web server VM instance.

   $conn = new mysqli($dbserver, $dbuser, $dbpassword);

   if (mysqli_connect_error()) {
    echo ("Database connection failed: " . mysqli_connect_error());
   } else {
    echo ("Database connection succeeded.");
   }
   ?>
   </body></html>
   ```

   - Replace the `root` with your instance password, fo rour case this will be `1234567`
   - To get the IP address for the `blog-db` sql instance. Open another tab and run:

     ```
     gcloud sql instances describe blog-db --format 'get(ipAddresses.ipAddress)'
     ```

   - Copy the output. Go back to `cutomerhost` vm ssh terminal and replace `CLOUDSQLIP` with the copied value.
   - Press `Ctrl+O`, and then press `Enter` to save your edited file. Press `Ctrl+X` to exit the nano text editor.
