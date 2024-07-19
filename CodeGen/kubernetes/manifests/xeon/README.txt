1. Prerequisites: ODF + image registry (exposed - https://docs.openshift.com/container-platform/4.14/registry/securing-exposing-registry.html)

2. Install operators: Red Hat Openshift AI, Red Hat OpenShift Serverless, Red Hat OpenShift Service Mesh

3. Open RedHat OpenShift AI dashboard (Networking -> Routes -> rhods-dashboard)

4. Go to "Data Science Projects" and create the new one

5. Go to "Data connections" tab under new project and add new data connection 
(tested with noobaa as s3 https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.14/html/managing_hybrid_and_multicloud_resources/accessing-the-multicloud-object-gateway-with-your-applications_rhodf#accessing-the-Multicloud-object-gateway-from-the-terminal_rhodf)

6. Go to "Workbenches" tab under new project and create the new one with Image HabanaAI
and Persistent storage ~ 40 GB with existing data connection created in previous step

7. Open notebook and run following commands to download the model and upload it on s3

%env S3_ENDPOINT=<insert-your-endpoint>
%env S3_ACCESS_KEY=<insert-your-access-key>
%env S3_SECRET_KEY=<insert-your-secret-key>
%env HF_TOKEN=<insert-your-huggingface-token>
 
!pip install huggingface-hub

import os
import boto3
import botocore
import glob
from huggingface_hub import snapshot_download
bucket_name = 'first.bucket'
s3_endpoint = os.environ.get('S3_ENDPOINT')
s3_accesskey = os.environ.get('S3_ACCESS_KEY')
s3_secretkey = os.environ.get('S3_SECRET_KEY')
path = 'models'
hf_token = os.environ.get('HF_TOKEN')
session = boto3.session.Session()
s3_resource = session.resource('s3',
                               endpoint_url=s3_endpoint,
                               verify=False,
                               aws_access_key_id=s3_accesskey,
                               aws_secret_access_key=s3_secretkey)
bucket = s3_resource.Bucket(bucket_name)
snapshot_download("ise-uiuc/Magicoder-S-DS-6.7B", cache_dir=f'./models', token=hf_token)
files = (file for file in glob.glob(f'{path}/**/*', recursive=True) if os.path.isfile(file) and "snapshots" in file)
for filename in files:
    s3_name = filename.replace(path, '')
    print(f'Uploading: {filename} to {path}{s3_name}')
    bucket.upload_file(filename, f'{path}{s3_name}')

8. Back to Red Hat OpenShift AI dashboard, go to "Settings" -> "Serving runtimes" and add the new one with "Single-model serving platform" and "REST". Upload servingruntime yaml (insert your hugginface token).

9. Go to your project and "Models" tab to create the new one with newly created serving runtime and custom server size (min. 16 Cores and 64 Gi). Use existing data connection with path "models". Deploy the model.

10. Go to terminal, login to OCP and check the URL of route.serving.knative.dev/magicoder-predictor in your project and use it as a TGI_LLM_ENDPOINT in codegen yaml.

11. Build images (https://github.com/polszewska/GenAIExamples/blob/codegen/CodeGen/docker/xeon/README.md)

12. Login to podman, tag the image and push it to image registry

podman login -u <user> -p $(oc whoami -t) <openshift-image-registry_route> --tls-verify=false
podman tag <image_id> <openshift-image-registry_route>/<namespace>/<image_name>:<tag>
podman push <openshift-image-registry_route>/<namespace>/<image_name>:<tag>

13. Update images names in codegen.yaml + deploy on xeon (https://github.com/polszewska/GenAIExamples/blob/codegen/CodeGen/kubernetes/manifests/README.md)

14. Deploy UI
