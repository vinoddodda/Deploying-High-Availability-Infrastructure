# Deploying HA Infrastructure

The first step in this project will be deploying infrastructure that you can run Prometheus and Grafana on. You will then use the servers you deployed to create an SLO/SLI dashboard. Next, you will modify existing infrastructure templates and deploy a highly-available infrastructure to AWS in multiple zones using Terrafrom. With this you will also deploy a RDS database cluster that has a replica in the alternate zone.

## Getting Started

Clone the appropriate git repo with the starter code. There will be 2 folders. Zone1 and zone2. This is where you will run the code from in your AWS Cloudshell terminal.

### Dependencies

```
- helm
- Terraform
- Postman
- kubectl
- Prometheus and Grafana
```

### Installation

1. Open your AWS console and ensure it is set for region `us-east-1`. Open the CloudShell by clicking the little shell icon in the toolbar at the top near the search box. 

<!-- 1. Set up your aws credentials from Udacity AWS Gateway locally
    - https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html
    - Set your region to `us-east-1` -->

2. Copy the AMI to your account

   **Restore image**

    ```shell
    aws ec2 create-restore-image-task --object-key ami-0ec6fdfb365e5fc00.bin --bucket udacity-srend --name "udacity-<your_name>"
    ```
    <!-- - Replace the owner field in `_data.tf` with your Amazon owner ID assigned on the AMI (you can get this in the console by going to EC2 - AMIs and selecting the Owned by me at the top filter) -->
    - Take note of that AMI ID the script just output. Copy the AMI to `us-east-2` and `us-west-1`:
        - `aws ec2 copy-image --source-image-id <your-ami-id-from-above> --source-region us-east-1 --region us-east-2 --name "udacity-tscotto"`
        - `aws ec2 copy-image --source-image-id <your-ami-id-from-above> --source-region us-east-1 --region us-west-1 --name "udacity-tscotto"`

    - Make note of the ami output from the above 2 commands. You'll need to put this in the `ec2.tf` file for `zone1` for `us-east-2` and in `ec2.tf` file for `zone2` for `us-west-1` respectively

    <!-- - Set your aws cli config to `us-east-2` -->

3. Close your CloudShell. Change your region to `us-east-2`. From the AWS console create an S3 bucket in `us-east-2` called `udacity-tf-<your_name>` e.g `udacity-tf-tscotto`
    - click next until created.
    - Update `_config.tf` in the `zone1` folder with your S3 bucket name where you will replace `<your_name>` with your name

4. Change your region to `us-west-1`. From the AWS console create an S3 bucket in `us-west-1` called `udacity-tf-<your_name>-west` e.g `udacity-tf-tscotto`
    - click next until created.
    - Update `_config.tf` in the `zone2` folder with your S3 bucket name where you will replace `<yourname>` with your name

5. Create a private key pair for your EC2 instances
    - Do this in **BOTH** `us-east-2` and `us-west-1`
    - Name the key `udacity`

6. Setup your CloudShell. Open CloudShell in the `us-east-2` region. Install the following:
- helm
    - `export VERIFY_CHECKSUM=false`
    - `curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash`

- terraform
    - `wget https://releases.hashicorp.com/terraform/1.0.7/terraform_1.0.7_linux_amd64.zip`
    - `unzip terraform_1.0.7_linux_amd64.zip`
    - `mkdir ~/bin`
    - `mv terraform ~/bin`
    - `export TF_PLUGIN_CACHE_DIR="/tmp"`

- kubectl
    - `curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl`
    - `chmod +x ./kubectl`
    - `mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin`
    - `echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc`

7. Deploy Terraform infrastructure
    - Clone the starter code from the git repo to a folder CloudShell
    - `cd` into the `zone1` folder
    - `terraform init`
    - `terraform apply`

**NOTE** The first time you run `terraform apply` you may see errors about the Kubernetes namespace or an RDS error. Running it again AND performing the step below should clear up those errors.

8. Setup Kubernetes config so you can ping the EKS cluster
   - `aws eks --region us-east-2 update-kubeconfig --name udacity-cluster`
   - Change kubernetes context to the new AWS cluster
     - `kubectl config use-context <cluster_name>`
       - e.g ` arn:aws:eks:us-east-2:139802095464:cluster/udacity-cluster`
   - Confirm with: `kubectl get pods --all-namespaces`
   - Then run `kubectl create namespace monitoring`
   <!-- - Change context to `udacity` namespace
     - `kubectl config set-context --current --namespace=udacity` -->

<!-- 5. Once the script finishes **Configure nginx** 
`sudo nano /etc/nginx/sites-enabled/default`

```
server {
    listen 80;
    location / {
        proxy_pass http://127.0.0.1:8000;
    }
}
``` -->

<!-- 6. Then save.
7. Be sure to check for errors, then reload nginx:
```
sudo nginx -t
sudo systemctl restart nginx
``` -->

9. Login to the AWS console and copy the public IP address of your Ubuntu-Web EC2 instance. Ensure you are in the us-east-2 region.

10. Edit the `prometheus-additional.yaml` file and replace the `<public_ip>` entries with the public IP of your Ubuntu Web. Save the file.

11. Install Prometheus and Grafana

    Change directories to your project directory `cd ../..`

    `kubectl create secret generic additional-scrape-configs --from-file=prometheus-additional.yaml --namespace monitoring`

    `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`

    `helm install prometheus prometheus-community/kube-prometheus-stack -f "values.yaml" --namespace monitoring`

    `helm install prometheus-blackbox-exporter prometheus-community/prometheus-blackbox-exporter -f "blackbox-values.yaml" --namespace monitoring`

<!-- `helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring` -->

<!-- 10. Port forward
`kubectl -n monitoring  port-forward svc/prometheus-grafana  8888:80`
`kubectl -n monitoring  port-forward svc/prometheus-kube-prometheus-prometheus 8889:9090` -->

<!-- Point your local web browser to http://localhost:8888 for Grafana access and http://localhost:8889 for Prometheus access -->

Get the DNS of your load balancer provisioned to access Grafana. You can find this by opening your AWS console and going to EC2 -> Load Balancers and selecting the load balancer provisioned. The DNS name of it will be listed below that you can copy and paste into your browser. Type that into your web browser to access Grafana.

Login to Grafana with `admin` for the username and `prom-operator` for the password.

12. Install Postman from [here](https://www.postman.com/downloads/). See additional instructions for [importing the collection, and enviroment files](https://learning.postman.com/docs/getting-started/importing-and-exporting-data/#importing-postman-data)

13. Open Postman and load the files `SRE-Project-postman-collection.json` and `SRE-Project.postman_environment.json`
    1. At the top level of the project in Postman, create the `public-ip`, `username` and `email` variable in the Postman file with the public IP you gathered from above and click Save. You can choose whatever you like for the username and email.

    2. Run the `Initialize the Database` and `Register a User` tasks in Postmane. In the register tasks, you will output a token. Make note of this token. 

    3. Edit the `Create Event` task and under the Authorization tab put in the token from above under the Token field. Click SAVE! Do the same for the `Get all events` and `Get Event by Id` tasks. Click SAVE!

    4. Run `Create Event` for 100 iterations by clicking the top level `SRE Project` folder in the left-hand side and select just `Create Event` and click the Run icon in the toolbar.

    5. Run `Get all events` for 100 iterations by clicking the top level `SRE Project` folder in the left-hand side and slect just `Get All Events` and click the Run icon in the toolbar.

    <!-- 2. Run the Postman runners to generate some traffic. Use 100 iterations -->
