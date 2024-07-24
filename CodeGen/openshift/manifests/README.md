<h1 align="center" id="title">Deploy CodeGen on OpenShift Cluster</h1>

## Prerequisites

1. **Red Hat OpenShift Cluster** with dynamic *StorageClass* to provision *PersistentVolumes* e.g. **OpenShift Data Foundation**)
2. Exposed image registry to push there docker images (https://docs.openshift.com/container-platform/4.16/registry/securing-exposing-registry.html).
3. Access to S3-compatible object storage bucket (e.g. **OpenShift Data Foundation**, **AWS S3**) and values of access and secret access keys and S3 endpoint (https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.16/html/managing_hybrid_and_multicloud_resources/accessing-the-multicloud-object-gateway-with-your-applications_rhodf#accessing-the-multicloud-object-gateway-with-your-applications_rhodf)\
4. Access to https://huggingface.co/ to get access token with *Read* permissions. Update the access token in your repository:
```
cd GenAIExamples/CodeGen/openshift/manifests/xeon
export HUGGINGFACEHUB_API_TOKEN="YourOwnToken"
sed -i "s/insert-your-huggingface-token-here/${HUGGINGFACEHUB_API_TOKEN}/g" codegen.yaml
sed -i "s/insert-your-huggingface-token-here/${HUGGINGFACEHUB_API_TOKEN}/g" servingruntime-magicoder.yaml
```
You can also customize the "MODEL_ID" and "model-volume".

## Deploy model in Red Hat Openshift AI

1. Log in to the Openshift Container Platform web console and select **Operators** > **OperatorHub**.
2. Search for **Red Hat Openshift AI**, open it and click **Install**.  In the same way install rest of required operators: **Red Hat OpenShift Serverless** and **Red Hat OpenShift Service Mesh**.
3. Open **Red Hat OpenShift AI** dashboard: go to **Networking** -> **Routes** and find location for *rhods-dashboard*).
4. Go to **Data Science Project** and clik **Create data science project**. Fill the **Name** and click **Create**.
5. Go to **Workbenches** tab and clik **Create workbench**. Fill the **Name**, under **Notebook image** choose *Standard Data Science*, under **Cluster storage** choose *Create new persistent storage* and change **Persistent storage size** to 40 GB. Under **Data connections** choose *Create new data connection* and fill all required fields for s3 access. Click **Create workbench**.
6. Open newly created Jupiter notebook and run following commands to download the model and upload it on s3:
```
%env S3_ENDPOINT=<S3_RGW_ROUTE>
%env S3_ACCESS_KEY=<AWS_ACCESS_KEY_ID>
%env S3_SECRET_KEY=<AWS_SECRET_ACCESS_KEY>
%env HF_TOKEN=<PASTE_HUGGINGFACE_TOKEN>
```
```
!pip install huggingface-hub
```
```
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
```
```
snapshot_download("ise-uiuc/Magicoder-S-DS-6.7B", cache_dir=f'./models', token=hf_token)
```
```
files = (file for file in glob.glob(f'{path}/**/*', recursive=True) if os.path.isfile(file) and "snapshots" in file)
for filename in files:
    s3_name = filename.replace(path, '')
    print(f'Uploading: {filename} to {path}{s3_name}')
    bucket.upload_file(filename, f'{path}{s3_name}')
```
7. Back to **Red Hat OpenShift AI** dashboard, go to **Settings** -> **Serving runtimes** and click **Add serving runtime**. Choose *Single-model serving platform* and *REST* as API protocol. Upload the file or copy the content of *servingruntime-magicoder.yaml*.

8. Go to your project, then "Models" tab and click **Deploy model** under *Single-model serving platform*. Fill the **Name**, choose newly created **Serving runtime**: *ise-uiuc/Magicoder-S-DS-6.7B on CPU*, **Model framework**: *llm* and change **Model server size** to *Custom*: 16 CPUs and 64 Gi memory. Under **Model location** choose *Existing data connection* and **Path**: *models*. Click **Deploy**.

## Deploy CodeGen On Xeon

1. Update TGI endpoint

Login to OpenShift CLI and find the URL of TGI_LLM_ENDPOINT:
```
oc get service.serving.knative.dev
```
Update the TGI_LLM_ENDPOINT in your repository:
```
cd GenAIExamples/CodeGen/openshift/manifests/xeon
export TGI_LLM_ENDPOINT="YourURL"
sed -i "s/insert-your-tgi-url-here/${TGI_LLM_ENDPOINT}/g" codegen.yaml
```

2. Build docker images locally (https://github.com/opea-project/GenAIExamples/tree/main/CodeGen/docker/xeon#-build-docker-images)

- LLM Docker Image:
```
git clone https://github.com/opea-project/GenAIComps.git
cd GenAIComps
docker build -t opea/llm-tgi:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/llms/text-generation/tgi/Dockerfile .
```
- MegaService Docker Image:
```
git clone https://github.com/opea-project/GenAIExamples
cd GenAIExamples/CodeGen/docker
docker build -t opea/codegen:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f Dockerfile .
```
- UI Docker Image:
```
cd GenAIExamples/CodeGen/docker/ui/
docker build -t opea/codegen-ui:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f ./docker/Dockerfile .
```
To verify run the command: `docker images`.

3. Login to podman, tag the images and push it to image registry with following commands:

```
podman login -u <user> -p $(oc whoami -t) <openshift-image-registry_route> --tls-verify=false
podman tag <image_id> <openshift-image-registry_route>/<namespace>/<image_name>:<tag>
podman push <openshift-image-registry_route>/<namespace>/<image_name>:<tag>
```
4. Update images names in *codegen.yaml*

```
cd GenAIExamples/CodeGen/openshift/manifests/xeon
export IMAGE_LLM_TGI="YourImage"
export IMAGE_CODEGEN="YourImage"
export IMAGE_CODEGEN_UI="YourImage"
sed -i "s/insert-your-image-llm-tgi/${IMAGE_LLM_TGI}/g" codegen.yaml
sed -i "s/insert-your-image-codegen/${IMAGE_CODEGEN}/g" codegen.yaml
sed -i "s/insert-your-image-codege-ui/${IMAGE_CODEGEN_UI}/g" codegen.yaml
```

5. Deploy CodeGen with command:
```
oc apply -f codegen.yaml
oc expose svc codegen
```

6. Check the *codegen* route with command `oc get routes` and update the route in *server-ui.yaml* file: 
```
cd GenAIExamples/CodeGen/openshift/manifests/xeon
export CODEGEN_ROUTE="YourCodegenRoute"
sed -i "s/insert-your-codegen-route/${CODEGEN_ROUTE}/g" ui-server.yaml
```

7. Deploy UI with command:
```
oc apply -f ui-server.yaml
oc expose deployment ui-server
oc expose svc ui-server
```

## Verify Services

Make sure all the pods are running, and restart the codegen-xxxx pod if necessary.

```
oc get pods
curl http://${CODEGEN_ROUTE}/v1/codegen -H "Content-Type: application/json" -d '{
     "messages": "Implement a high-level API for a TODO list application. The API takes as input an operation request and updates the TODO list in place. If the request is invalid, raise an exception."
     }'
```

## Launch the UI

To access the frontend, find the route for *ui-server* with command `oc get routes` and open it in the browser.
