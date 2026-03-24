# 05.4-GCP-Workload-Identity-Federation-WIF-with-Docker-Java-No-Service-Account-Key-Access-to-GCS
how to securely access Google Cloud Storage (GCS) from a Docker container using Workload Identity Federation (WIF) — without using service account keys or installing gcloud inside the container

Key Concepts Covered:
- WIF architecture (external identity → STS → service account impersonation)
- Docker + Java integration with GCP
- GOOGLE_APPLICATION_CREDENTIALS usage with WIF config
- Secure cloud authentication best practices (keyless)

Tools Used:
- Google Cloud Platform (GCP)
- Workload Identity Federation (WIF)
- Docker
- Java (Google Cloud Storage SDK)
- gcloud CLI (for initial setup only)


Phase 1: Infrastructure Setup (Admin)
-------------------------------------
Run these commands on your local machine or Cloud Shell to set up the GCP environment.
#Step 1.1: Set Variables
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
export POOL_NAME="vm-poc-pool-id06"
export PROVIDER_NAME="vm-poc-oidc-provider"
export SA_NAME="wif-poc-sa"
export BUCKET_NAME="wif-poc-bucket-${PROJECT_ID}"
export LOCATION="global"
# We allow the Google CLI audience to simulate an external IDP for this PoC
export ALLOWED_AUDIENCE="32555940559.apps.googleusercontent.com"

#explain aud, 
You can define it — BUT both sides must match
Token "aud"  ==  WIF provider allowed_audience

##for audience, u can get using
gcloud auth login

Step 1.2: Enable APIs
gcloud services enable iam.googleapis.com sts.googleapis.com iamcredentials.googleapis.com storage.googleapis.com

Step 1.3: Create Resources (Bucket & Service Account)
# Create Bucket
gcloud storage buckets create gs://$BUCKET_NAME --location=US
echo "WIF Authentication Successful" > sample.txt
gcloud storage cp sample.txt gs://$BUCKET_NAME/sample.txt

# Create Service Account
gcloud iam service-accounts create $SA_NAME --display-name="WIF PoC Service Account"

# Grant Access (Service Account can view objects)
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/storage.admin"

Step 1.4: Configure Workload Identity Federation
# Create the Pool
gcloud iam workload-identity-pools create $POOL_NAME \
    --location=$LOCATION \
    --display-name="vm-poc-test"

# Create the OIDC Provider
# logic: We map the external token's subject (sub) to google.subject
gcloud iam workload-identity-pools providers create-oidc $PROVIDER_NAME \
    --workload-identity-pool=$POOL_NAME \
    --location=$LOCATION \
    --issuer-uri="https://accounts.google.com" \
    --allowed-audiences=$ALLOWED_AUDIENCE \
    --attribute-mapping="google.subject=assertion.sub"

Step 1.5: Bind Identity to Service Account
This step authorizes the WIF pool to impersonate the Service Account.
# Allow any token from this pool to impersonate the SA (Scoped for PoC)

##need 1 Role (either workloadIdentityUser or serviceAccountTokenCreator)
#update policy binding
gcloud iam service-accounts add-iam-policy-binding \
wif-poc-sa@winppk-app01.iam.gserviceaccount.com \
--role="roles/iam.workloadIdentityUser" \
--member="principal://iam.googleapis.com/projects/259655456214/locations/global/workloadIdentityPools/vm-poc-pool08/subject/113332787600104397101"

#explain subject=subject is caller, who is calling. Eg; if u use github,
caller is git hub account repository,
if u use google, in this lab, caller is default sa which is attaching to
created test vm.

##verification if you bind
gcloud iam service-accounts get-iam-policy \
  wif-poc-sa@winppk-app01.iam.gserviceaccount.com


Phase 2: Workload Configuration (On the VM)
-------------------------------------------
Create VM
gcloud compute instances create wif-vm \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --scopes=https://www.googleapis.com/auth/cloud-platform
  
Log in to your Linux VM. The following script automates the application build and execution.
<put below scripts>

# 1. Install Docker if missing
if ! command -v docker &> /dev/null; then
    echo "Installing Docker..."
    sudo apt-get update && sudo apt-get install -y docker.io
    sudo usermod -aG docker $USER
    sudo chmod 666 /var/run/docker.sock
fi

# 2. Create Java Application (Source, POM, Dockerfile)
mkdir -p src/main/java/com/example/wif target

cat <<EOF > src/main/java/com/example/wif/App.java
package com.example.wif;
import com.google.cloud.storage.Bucket;
import com.google.cloud.storage.Storage;
import com.google.cloud.storage.StorageOptions;

public class App {
    public static void main(String[] args) {
        System.out.println("Starting Java WIF App...");
        try {
            Storage storage = StorageOptions.getDefaultInstance().getService();
            System.out.println("Success! Authenticated via WIF.");
            System.out.println("Listing Buckets:");
            for (Bucket bucket : storage.list().iterateAll()) {
                System.out.println(" - " + bucket.getName());
            }
        } catch (Exception e) {
            System.err.println("Error: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
EOF

cat <<EOF > pom.xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>wif-poc</artifactId>
    <version>1.0-SNAPSHOT</version>
    <dependencies>
        <dependency><groupId>com.google.cloud</groupId><artifactId>google-cloud-storage</artifactId><version>2.30.1</version></dependency>
    </dependencies>
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>
    <build><plugins><plugin>
        <artifactId>maven-assembly-plugin</artifactId><version>3.6.0</version>
        <configuration><descriptorRefs><descriptorRef>jar-with-dependencies</descriptorRef></descriptorRefs><archive><manifest><mainClass>com.example.wif.App</mainClass></manifest></archive></configuration>
        <executions><execution><id>make-assembly</id><phase>package</phase><goals><goal>single</goal></goals></execution></executions>
    </plugin></plugins></build>
</project>
EOF

# Dockerfile
cat <<EOF > Dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*-jar-with-dependencies.jar app.jar
RUN mkdir -p /app/secrets /var/secrets
CMD ["java", "-jar", "app.jar"]
EOF

# 3. Generate Credentials and Config
echo "Generating WIF Config and Identity Token..."
gcloud iam workload-identity-pools create-cred-config \
    projects/259655456214/locations/global/workloadIdentityPools/vm-poc-pool08/providers/vm-poc-oidc-provider \ #modify according to data
    --service-account="wif-poc-sa@winppk-app01.iam.gserviceaccount.com" \
    --credential-source-file="/var/secrets/token.jwt" \
    --output-file=oidc-config.json \
    --service-account-token-lifetime-seconds 600 ##can set min 600s~3600s(10min-1hr)

gcloud auth print-identity-token > my-token.jwt

# 4. Build and Run Docker
echo "Building and Running Container..."
docker build -t java-wif-app:latest .
docker run --rm \
  --name java-wif-app \
  -v $(pwd)/oidc-config.json:/app/secrets/gcp-config.json \
  -v $(pwd)/my-token.jwt:/var/secrets/token.jwt \
  -e GOOGLE_APPLICATION_CREDENTIALS=/app/secrets/gcp-config.json \
  java-wif-app:latest


