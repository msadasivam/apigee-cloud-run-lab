# Apigee Cloud Run Hello World

 * [Setup Cloud Run service](#setup-cloud-run-service)
 * [Setup Apigee](#setup-apigee)
 * [Test API](#test-api)

This sample shows how to deploy a Cloud Run service and expose it as an API managed by Apigee.

## Setup Cloud Run service 

1. [Set up for Cloud Run development](https://cloud.google.com/run/docs/setup)

2. Clone this repository:

    ```sh
    git clone https://github.com/GoogleCloudPlatform/nodejs-docs-samples.git
    ```

3. Deploying Cloud Run Service

   ```sh
    cd nodejs-docs-samples/run/helloworld/
    npm install

    export GOOGLE_CLOUD_PROJECT=<your-gcp-project>

    gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/helloworld
    gcloud run deploy helloworld \
      # Needed for Manual Logging sample.
      --set-env-vars GOOGLE_CLOUD_PROJECT=${GOOGLE_CLOUD_PROJECT} \
      --image gcr.io/${GOOGLE_CLOUD_PROJECT}/helloworld
   ```
   Pick any cloud region, auth enabled

4. Test Cloud Run Service

   ```
    curl (your-cloud-run-url) \
        -H "Authorization: Bearer $(gcloud auth print-identity-token)"
   ```
   
    Expected Response

   ```
    Hello World!
   ```

## Setup Apigee

The following repo contains a Shared Flow that can fetch GCP OAuth access or identity token using a service account. To deploy the SharedFlow and setup service account in Apigee follow the below steps.

1. Clone apigee/devrel repository:

    ```sh
    git clone https://github.com/apigee/devrel.git
    ```

2. Jump to IAM in cloud console and download "Compute Engine default service account" JSON key for the service account. See GCP
   [docs](https://cloud.google.com/iam/docs/creating-managing-service-account-keys). 
   
   *Note*: Compute Engine default service account chosen for simplicity of steps in this labs. In a prod setup a dedicated service account with right Apigee role will be used.

3. Deploy the Shared Flow

    Provide path to service account to initialize the kvm value. This script will
    * Create the GCP Service Account Token Shared Flow
    * Initialize an environement Cache Resource
    * Initialize a KVM
    * Add a service account key to that kvm


   ```sh
    # set your Apigee org variables:
    export APIGEE_X_ORG=<your org>
    export APIGEE_X_ENV=<your environment>
    export APIGEE_X_HOSTNAME=<hostname of your environment>
    
    cd devrel/references/gcp-sa-auth-shared-flow/
    ./deploy.sh --googleapi (path-to-sa-key.json)
   ```

4. Shared Flow invocation from within an API proxy

    1. Add the following KVMOperations policy to pick the right service account key file from KVM. Attach the policy to "default" Target Endpoint in the PreFlow phase.

        ```xml
        <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
        <KeyValueMapOperations name="KV.Lookup-SA-Key" mapIdentifier="gcp-sa-devrel">
            <ExpiryTimeInSecs>300</ExpiryTimeInSecs>
            <Get assignTo="private.gcp.service_account.key">
                <Key>
                    <Parameter>apigee@iam.gserviceaccount.com</Parameter>
                </Key>
            </Get>
        </KeyValueMapOperations>
        ```

    1. Add the following AssignMessage policy as the second policy to set the right audience in the GCP identity token. Update the "Value" to point to your Cloud Run URL. Attach it after the earlier policy.

        ```xml
        <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
        <AssignMessage name="AM.GCPScopes">
            <AssignVariable>
                <Name>gcp.target_audience</Name>
                <Value>https://(cloud-run-URL-here)</Value>
            </AssignVariable>
        </AssignMessage>
        ```

    1. Add the following FlowCallout policy to invoke the Shared Flow to create a GCP service account identity token. Attach it as the third policy after the earlier one.

        ```xml
        <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
        <FlowCallout continueOnError="false" enabled="true" name="GCP-sa-auth-v1">
            <DisplayName>GCP-sa-auth-v1</DisplayName>
            <FaultRules/>
            <Properties/>
            <SharedFlowBundle>gcp-sa-auth-v1</SharedFlowBundle>
        </FlowCallout>
        ```

    1. Add the following AssignMessage policy to use the identity token required to invoke the Cloud Run endpoint.

        ```xml
        <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
        <AssignMessage continueOnError="false" enabled="true" name="AMGCPAuth">
            <DisplayName>AM.GCPAuth</DisplayName>
            <Properties/>
            <Set>
                <Headers>
                    <Header name="Authorization">Bearer {private.gcp.access_token}</Header>
                </Headers>
            </Set>
            <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
            <AssignTo createNew="false" transport="http" type="request"/>
        </AssignMessage>
        ```

1. Go to API Proxies and open "Hipster-Products-API". Update Target URL in Target Endpoint "default" xml configuration. Identity the URL and update it to the Cloud RUN URL.

    ```xml
        <HTTPTargetConnection>
            <URL>https://(cloud-run-URL-here)</URL>
        </HTTPTargetConnection>
    ```

1. Save and Deploy API to the available environment.

## Test API
Enable "Debug" and invoke the API. You will find the identity token created and injected into the request. 

```sh
    curl -k -i https://$DOMAIN/hello-world-run \
        --resolve "$DOMAIN:443:$APIGEE_EXTERNAL_IP"
```

### Reference
* [Google Cloud Run Node.js Samples README](https://github.com/GoogleCloudPlatform/nodejs-docs-samples/tree/master/run)
