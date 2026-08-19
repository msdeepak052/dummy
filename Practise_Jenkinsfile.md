# **Real production-style Jenkins Declarative Pipelines**, not simple tutorial Jenkinsfiles.

A good common pattern is:

* Jenkins controller does **not** run builds.
* Every build gets an **ephemeral Kubernetes pod**.
* Use multiple containers inside the pod where appropriate.
* Use `skipDefaultCheckout(true)`.
* Use Jenkins Credentials rather than hardcoded secrets.
* Use `timeout`, `timestamps`, `disableConcurrentBuilds`.
* Use `cleanWs()` after execution.
* Use **Kaniko/BuildKit**, not mounting `/var/run/docker.sock`.
* For CD, prefer **GitOps**: Jenkins modifies Helm values → commits → pushes to GitHub → Argo CD/Flux deploys.

---

## 1. Terraform VPC Creation

Typical flow:

```text
GitHub
   │
   ▼
Jenkins
   │
   ▼
Kubernetes Pod
 ┌──────────────────────┐
 │ terraform container   │
 └──────────────────────┘
   │
   ├── terraform fmt
   ├── terraform init
   ├── terraform validate
   ├── terraform plan
   │
   ▼
Manual Approval
   │
   ▼
terraform apply
   │
   ▼
AWS VPC
```

### Jenkinsfile

```groovy
pipeline {

    agent {
        kubernetes {
            defaultContainer 'terraform'
            retries 2

            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-terraform
spec:
  serviceAccountName: jenkins
  containers:

  - name: terraform
    image: hashicorp/terraform:1.12
    command:
    - cat
    tty: true
'''
        }
    }

    options {
        skipDefaultCheckout(true)
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        TF_IN_AUTOMATION = 'true'
        TF_INPUT         = 'false'
        AWS_REGION       = 'ap-south-1'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Terraform Format') {
            steps {
                sh '''
                    terraform fmt -check -recursive
                '''
            }
        }

        stage('Terraform Init') {
            steps {
                sh '''
                    terraform init \
                      -input=false \
                      -upgrade
                '''
            }
        }

        stage('Terraform Validate') {
            steps {
                sh '''
                    terraform validate
                '''
            }
        }

        stage('Terraform Plan') {
            steps {
                sh '''
                    terraform plan \
                      -input=false \
                      -out=tfplan
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'tfplan',
                                     fingerprint: true
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Apply Terraform changes?',
                          ok: 'Apply'
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                sh '''
                    terraform apply \
                      -input=false \
                      -auto-approve \
                      tfplan
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }

        success {
            echo 'Terraform VPC deployment successful'
        }

        failure {
            echo 'Terraform VPC deployment failed'
        }
    }
}
```

### Production improvement

For AWS authentication, **don't put AWS access keys in the Jenkinsfile**.

Prefer:

```text
Jenkins
   │
   └── Kubernetes ServiceAccount
           │
           └── AWS IAM Role
                   │
                   └── Terraform
```

For EKS, this can be implemented with **IRSA / EKS Pod Identity** depending on your environment.

---



## 2. Jenkins Shared Libraries — Complete Example

## 1. First understand the problem

Imagine you have 50 repositories:

```text
payment-service
order-service
user-service
notification-service
inventory-service
...
```

Without Shared Libraries, every repository may have:

```text
Jenkinsfile
├── checkout
├── build
├── test
├── SonarQube
├── Trivy
├── Docker build
├── ECR login
├── Docker push
├── Slack notification
├── cleanup
└── error handling
```

You might end up maintaining:

```text
50 repositories
×
300 lines Jenkinsfile
=
15,000 lines of duplicated CI/CD logic
```

Now suppose security says:

> "Every image must be scanned before pushing to ECR."

You would potentially need to modify dozens of Jenkinsfiles.

---

# 2. Shared Library solves this

Instead:

```text
                         Jenkins
                            |
                    Shared Library
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
   payment-service    order-service    inventory-service
      Jenkinsfile       Jenkinsfile       Jenkinsfile
```

Each repository contains only:

```groovy
@Library('platform-shared-library@v2.0.0') _

ciPipeline(
    application: 'payment-service',
    dockerImage: '123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-service',
    dockerTag: "${env.BUILD_NUMBER}"
)
```

That's the main idea.

---

# 3. What is Jenkins Shared Library?

A Jenkins Shared Library is a Git repository containing reusable Jenkins Pipeline code.

For example:

```text
jenkins-shared-library
│
├── vars/
│   ├── ciPipeline.groovy
│   ├── terraformPipeline.groovy
│   └── helmDeploy.groovy
│
└── src/
    └── com/company/
        ├── Docker.groovy
        ├── AWS.groovy
        └── Kubernetes.groovy
```

The important directories are:

```text
vars/
src/
resources/
```

---

# 4. `vars/` — easiest way to understand

Anything under:

```text
vars/
```

can be exposed as a Pipeline step.

For example:

```text
vars/ciPipeline.groovy
```

can be called as:

```groovy
ciPipeline(...)
```

Similarly:

```text
vars/terraformPipeline.groovy
```

can be called:

```groovy
terraformPipeline(...)
```

And:

```text
vars/helmDeploy.groovy
```

can be called:

```groovy
helmDeploy(...)
```

So:

```text
vars/
├── ciPipeline.groovy
├── terraformPipeline.groovy
└── helmDeploy.groovy
```

becomes:

```text
ciPipeline()
terraformPipeline()
helmDeploy()
```

---

# 5. `src/` — reusable classes

`src/` is useful when your shared library becomes more sophisticated.

For example:

```text
src/com/company/AWS.groovy
src/com/company/Docker.groovy
src/com/company/Kubernetes.groovy
```

You can then have reusable classes:

```groovy
package com.company

class Docker implements Serializable {

    def steps

    Docker(steps) {
        this.steps = steps
    }

    def build(String image) {
        steps.sh "docker build -t ${image} ."
    }
}
```

You don't need `src/` for a simple Shared Library.

For your interview, explain:

> "`vars/` is convenient for reusable Pipeline steps, while `src/` is useful for more structured Groovy classes and reusable business logic."

---

# 6. Our complete example

Let's build a realistic library.

Repository:

```text
jenkins-shared-library/
│
├── vars/
│   ├── ciPipeline.groovy
│   ├── terraformPipeline.groovy
│   └── helmDeploy.groovy
│
├── src/
│   └── com/company/
│       └── PipelineUtils.groovy
│
└── README.md
```

For this example we'll concentrate on:

```text
ciPipeline.groovy
```

---

# 7. Step 1 — Create the Shared Library repository

Create a Git repository:

```text
jenkins-shared-library
```

For example:

```text
GitLab
└── platform
    └── jenkins-shared-library
```

Clone it:

```bash
git clone <SHARED_LIBRARY_REPO_URL>

cd jenkins-shared-library
```

Create:

```bash
mkdir vars
mkdir -p src/com/company
```

---

# 8. Step 2 — Create `ciPipeline.groovy`

Create:

```text
vars/ciPipeline.groovy
```

Complete example:

```groovy
def call(Map config = [:]) {

    pipeline {

        agent {
            kubernetes {

                defaultContainer 'tools'

                retries 2

                yaml """
apiVersion: v1
kind: Pod

metadata:
  labels:
    app: jenkins-ci-agent

spec:

  serviceAccountName: jenkins-build

  containers:

    - name: tools
      image: ${config.get('agentImage', 'my-company/ci-tools:1.0.0')}
      command:
        - cat
      tty: true
"""
            }
        }

        options {

            skipDefaultCheckout(true)

            timestamps()

            timeout(
                time: 30,
                unit: 'MINUTES'
            )

            disableConcurrentBuilds()
        }

        environment {

            APPLICATION = "${config.application}"

            DOCKER_IMAGE = "${config.dockerImage}"

            DOCKER_TAG = "${config.dockerTag ?: env.BUILD_NUMBER}"

            AWS_REGION = "${config.awsRegion ?: 'ap-south-1'}"
        }

        stages {

            stage('Checkout') {

                steps {

                    echo "Checking out ${env.APPLICATION}"

                    checkout scm
                }
            }

            stage('Build') {

                steps {

                    echo "Building ${env.APPLICATION}"

                    sh '''
                        set -e

                        echo "Build started"

                        # Application-specific build command
                        # Example:
                        # npm ci
                        # npm run build

                        echo "Build completed"
                    '''
                }
            }

            stage('Test') {

                steps {

                    echo "Running tests"

                    sh '''
                        set -e

                        echo "Running unit tests"

                        # Example:
                        # npm test
                        # mvn test
                        # pytest
                    '''
                }
            }

            stage('Security Scan') {

                steps {

                    echo "Running security scan"

                    sh '''
                        set -e

                        echo "Security scan started"

                        # Example:
                        # trivy fs .
                        # trivy image ${DOCKER_IMAGE}:${DOCKER_TAG}

                        echo "Security scan completed"
                    '''
                }
            }

            stage('AWS Identity') {

                steps {

                    sh '''
                        set -e

                        aws sts get-caller-identity
                    '''
                }
            }

            stage('Docker Build') {

                steps {

                    sh """
                        set -e

                        echo "Building image: ${DOCKER_IMAGE}:${DOCKER_TAG}"

                        # Production:
                        # Use BuildKit / Kaniko / Buildah
                        # instead of mounting Docker socket.

                        echo "Container build completed"
                    """
                }
            }

            stage('Push Image') {

                steps {

                    sh '''
                        set -e

                        echo "Pushing image to ECR"

                        # ECR authentication and push
                        # would be implemented here.
                    '''
                }
            }
        }

        post {

            always {

                echo "Cleaning workspace"

                cleanWs()
            }

            success {

                echo "CI pipeline completed successfully"
            }

            failure {

                echo "CI pipeline failed"
            }
        }
    }
}
```

---

# 9. What does `def call()` mean?

This is extremely important.

We have:

```groovy
def call(Map config = [:]) {
```

Because the file is:

```text
vars/ciPipeline.groovy
```

Jenkins exposes the `call()` method as:

```groovy
ciPipeline(...)
```

So:

```groovy
ciPipeline(
    application: 'payment-service'
)
```

internally invokes:

```groovy
call(
    application: 'payment-service'
)
```

---

# 10. Why use a Map?

We don't want this:

```groovy
ciPipeline(
    'payment-service',
    '123456789012...',
    'ap-south-1',
    '1.0'
)
```

It's difficult to understand.

Instead:

```groovy
ciPipeline(
    application: 'payment-service',
    dockerImage: '...',
    dockerTag: '1.0',
    awsRegion: 'ap-south-1'
)
```

Much clearer.

---

# 11. Application Jenkinsfile

Now the application repository contains:

```text
payment-service/
│
├── src/
├── Dockerfile
└── Jenkinsfile
```

And the Jenkinsfile becomes:

```groovy
@Library('platform-shared-library@v2.0.0') _

ciPipeline(
    application: 'payment-service',

    dockerImage:
        '123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-service',

    dockerTag:
        "${env.BUILD_NUMBER}",

    awsRegion:
        'ap-south-1'
)
```

That's it.

---

# 12. Another application

`order-service/Jenkinsfile`:

```groovy
@Library('platform-shared-library@v2.0.0') _

ciPipeline(
    application: 'order-service',

    dockerImage:
        '123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service',

    dockerTag:
        "${env.BUILD_NUMBER}",

    awsRegion:
        'ap-south-1'
)
```

Another:

```groovy
@Library('platform-shared-library@v2.0.0') _

ciPipeline(
    application: 'inventory-service',

    dockerImage:
        '123456789012.dkr.ecr.ap-south-1.amazonaws.com/inventory-service',

    dockerTag:
        "${env.BUILD_NUMBER}",

    awsRegion:
        'ap-south-1'
)
```

All three get:

```text
Checkout
Build
Test
Security Scan
AWS Identity
Docker Build
ECR Push
Cleanup
```

from the same centralized implementation.

---

# 13. What does `@Library` do?

This:

```groovy
@Library('platform-shared-library@v2.0.0') _
```

means:

> Load the Jenkins Shared Library named `platform-shared-library`, version `v2.0.0`.

The `_` tells Jenkins to load the library for the Pipeline.

---

# 14. Where does Jenkins know the library comes from?

You configure it globally.

Go to:

```text
Manage Jenkins
   ↓
System
   ↓
Global Pipeline Libraries
```

Add:

```text
Name:
platform-shared-library
```

Then:

```text
Default version:
v2.0.0
```

Repository:

```text
<SHARED_LIBRARY_REPO_URL>
```

For example:

```text
GitLab
   |
   v
platform/jenkins-shared-library
```

Jenkins then knows:

```text
platform-shared-library
        ↓
Git repository
        ↓
Shared library code
```

---

# 15. Versioning — very important

Don't blindly use:

```groovy
@Library('platform-shared-library') _
```

for production.

Why?

Imagine:

```text
50 repositories
        |
        v
Shared Library
        |
        v
Someone changes ciPipeline.groovy
```

Suddenly all 50 pipelines might behave differently.

Instead use versions:

```groovy
@Library('platform-shared-library@v2.0.0') _
```

Now:

```text
Application A → v2.0.0
Application B → v2.0.0
Application C → v2.0.0
```

You can release:

```text
v2.0.0
v2.1.0
v3.0.0
```

---

# 16. Version upgrade

Suppose you release:

```text
v2.1.0
```

You can test it in one application:

```groovy
@Library('platform-shared-library@v2.1.0') _
```

If successful:

```text
payment-service → v2.1.0
```

Then migrate:

```text
order-service → v2.1.0
inventory-service → v2.1.0
...
```

This is much safer than automatically changing every application.

---

# 17. Why Shared Library is valuable

Suppose security requires:

```text
Every build must run Trivy.
```

Without Shared Library:

```text
Repo A → modify Jenkinsfile
Repo B → modify Jenkinsfile
Repo C → modify Jenkinsfile
...
Repo Z → modify Jenkinsfile
```

With Shared Library:

```text
Shared Library
      |
      v
ciPipeline.groovy
      |
      +-- Trivy
```

Update once.

Applications consume the new version.

---

# 18. But don't put EVERYTHING in the Shared Library

This is an important Senior DevOps point.

Don't make:

```text
ciPipeline.groovy
```

contain:

```text
every possible technology
every possible application
every possible deployment
every possible cloud
```

You end up with:

```text
1000+ line
God Pipeline
```

Instead design reusable components.

For example:

```text
vars/
├── ciPipeline.groovy
├── terraformPipeline.groovy
├── helmDeploy.groovy
├── dockerBuild.groovy
├── securityScan.groovy
└── notifyBuild.groovy
```

---

# 19. Example: Terraform Shared Pipeline

You already have a Terraform Kubernetes Agent Pipeline.

That is a perfect candidate.

Create:

```text
vars/terraformPipeline.groovy
```

Example:

```groovy
def call(Map config = [:]) {

    pipeline {

        agent {
            kubernetes {

                defaultContainer 'terraform'

                retries 2

                yaml """
apiVersion: v1
kind: Pod

spec:

  serviceAccountName: jenkins-build

  containers:

    - name: terraform
      image: ${config.get(
          'terraformImage',
          'hashicorp/terraform:1.12'
      )}
      command:
        - cat
      tty: true
"""
            }
        }

        options {

            skipDefaultCheckout(true)

            timestamps()

            timeout(
                time: 30,
                unit: 'MINUTES'
            )

            disableConcurrentBuilds()
        }

        environment {

            AWS_REGION =
                "${config.awsRegion ?: 'ap-south-1'}"

            TF_IN_AUTOMATION = 'true'

            TF_INPUT = 'false'
        }

        stages {

            stage('Checkout') {

                steps {
                    checkout scm
                }
            }

            stage('Terraform Format') {

                steps {

                    sh '''
                        terraform fmt -check -recursive
                    '''
                }
            }

            stage('Terraform Init') {

                steps {

                    sh '''
                        terraform init \
                          -input=false
                    '''
                }
            }

            stage('Terraform Validate') {

                steps {

                    sh '''
                        terraform validate
                    '''
                }
            }

            stage('Terraform Plan') {

                steps {

                    sh '''
                        terraform plan \
                          -input=false \
                          -out=tfplan
                    '''
                }
            }

            stage('Approval') {

                steps {

                    timeout(
                        time: 10,
                        unit: 'MINUTES'
                    ) {

                        input(
                            message:
                                'Apply Terraform changes?',
                            ok:
                                'Apply'
                        )
                    }
                }
            }

            stage('Terraform Apply') {

                steps {

                    sh '''
                        terraform apply \
                          -input=false \
                          -auto-approve \
                          tfplan
                    '''
                }
            }
        }

        post {

            always {
                cleanWs()
            }

            success {
                echo 'Terraform deployment successful'
            }

            failure {
                echo 'Terraform deployment failed'
            }
        }
    }
}
```

---

# 20. Application Terraform Jenkinsfile

Now the Terraform repository only needs:

```groovy
@Library('platform-shared-library@v2.0.0') _

terraformPipeline(
    awsRegion: 'ap-south-1',
    terraformImage: 'hashicorp/terraform:1.12'
)
```

That's incredibly small.

---

# 21. Shared Library architecture

A good organization might look like:

```text
jenkins-shared-library
│
├── vars/
│   │
│   ├── ciPipeline.groovy
│   │
│   ├── terraformPipeline.groovy
│   │
│   ├── helmDeploy.groovy
│   │
│   ├── dockerBuild.groovy
│   │
│   ├── securityScan.groovy
│   │
│   └── notifyBuild.groovy
│
├── src/
│   └── com/company/
│       │
│       ├── AWS.groovy
│       ├── Docker.groovy
│       └── Kubernetes.groovy
│
├── resources/
│   └── com/company/
│       └── templates/
│
└── README.md
```

---

# 22. How the execution works

Let's follow:

```groovy
@Library('platform-shared-library@v2.0.0') _

ciPipeline(
    application: 'payment-service'
)
```

### Step 1

Jenkins reads:

```text
@Library(...)
```

---

### Step 2

Jenkins finds:

```text
platform-shared-library
```

---

### Step 3

Jenkins checks out:

```text
v2.0.0
```

from the configured Git repository.

---

### Step 4

Jenkins discovers:

```text
vars/ciPipeline.groovy
```

---

### Step 5

Jenkins executes:

```groovy
call(config)
```

---

### Step 6

The shared library creates the Pipeline:

```text
Kubernetes Agent
      ↓
Checkout
      ↓
Build
      ↓
Test
      ↓
Security Scan
      ↓
AWS
      ↓
Image Build
      ↓
ECR
```

---

# 23. Where does the Agent come from?

This is connected directly to your previous Jenkins/EKS discussion.

The Shared Library contains:

```groovy
agent {
    kubernetes {
        ...
    }
}
```

Therefore:

```text
Application Jenkinsfile
        |
        v
Shared Library
        |
        v
Kubernetes Plugin
        |
        v
EKS
        |
        v
Ephemeral Agent Pod
```

So the application repository doesn't need to repeat the Pod configuration.

---

# 24. Centralized security

This is one of the strongest real-world advantages.

Suppose your organization requires:

```text
1. Trivy scan
2. SonarQube
3. SBOM
4. Secret scanning
5. ECR vulnerability scanning
6. Image signing
```

Instead of asking every developer:

> "Please add these seven stages."

You implement them centrally:

```text
ciPipeline.groovy
│
├── Checkout
├── Build
├── Test
├── SonarQube
├── Secret Scan
├── Trivy
├── SBOM
├── Image Build
├── Sign
├── ECR Push
└── Notification
```

Application repository:

```groovy
ciPipeline(
    application: 'payment-service'
)
```

---

# 25. Centralized governance

You can enforce:

```text
No privileged containers
No Docker socket
Approved agent images
Approved Terraform versions
Approved security scans
Approved AWS authentication
Required timeouts
Required cleanup
Required notifications
```

For example, your Shared Library can always configure:

```groovy
options {
    timestamps()
    timeout(time: 30, unit: 'MINUTES')
    disableConcurrentBuilds()
}
```

Applications don't need to remember to add them.

---

# 26. But what if applications need customization?

That's why we use configuration.

Example:

```groovy
ciPipeline(
    application: 'payment-service',

    dockerImage: '...',

    runSecurityScan: true,

    runTests: true,

    deployToEKS: true
)
```

Inside:

```groovy
if (config.runSecurityScan) {

    stage('Security Scan') {
        ...
    }
}
```

Now the framework is reusable without becoming completely rigid.

---

# 27. Better pattern: configuration over code duplication

Bad:

```groovy
ciPipeline()

// Then copy 200 lines
// and modify them
```

Good:

```groovy
ciPipeline(
    application: 'payment-service',
    language: 'java',
    dockerImage: '...',
    runSecurityScan: true
)
```

The Shared Library owns the implementation.

The repository owns the configuration.

---

# 28. A realistic company structure

Imagine your Platform Engineering team owns:

```text
platform/jenkins-shared-library
```

Application teams own:

```text
payments/payment-service
payments/order-service
retail/inventory-service
```

Architecture:

```text
                  Platform Team
                       |
                       v
             Jenkins Shared Library
                       |
          +------------+------------+
          |            |            |
          v            v            v
      Payment        Order       Inventory
      Jenkinsfile   Jenkinsfile  Jenkinsfile
```

Platform team controls:

```text
CI/CD standards
Security
Agent images
AWS integration
Kubernetes integration
Notifications
```

Application team controls:

```text
Application name
Build parameters
Repository
Application-specific configuration
```

This is a very strong **Platform Engineering** model.

---

# 29. Shared Library + EKS Agent + AWS IAM

Now combine everything you've been learning.

```text
Developer
    |
    v
Application Git Repo
    |
    | Jenkinsfile
    v
Jenkins Controller
    |
    | @Library(...)
    v
Shared Library
    |
    | Kubernetes Agent definition
    v
Kubernetes Plugin
    |
    v
EKS
    |
    v
Ephemeral Agent Pod
    |
    | ServiceAccount
    v
EKS Pod Identity
    |
    v
IAM Role
    |
    +---- ECR
    +---- S3
    +---- STS
    +---- EKS
```

This is a very strong architecture to explain in a Senior DevOps interview.

---

# 30. Shared Library versioning strategy

A practical approach:

```text
v1.x.x
```

for old pipeline framework.

```text
v2.x.x
```

for newer framework.

Example:

```text
v2.0.0
    |
    +-- New security scanning
    +-- New EKS agent image
    +-- BuildKit
```

Applications migrate deliberately:

```text
payment-service      → v2.0.0
order-service        → v1.8.2
inventory-service    → v1.8.2
```

Then after testing:

```text
payment-service      → v2.0.0
order-service        → v2.0.0
inventory-service    → v2.0.0
```

---

# 31. What if Shared Library has a bug?

This is why versioning matters.

Suppose:

```text
v2.0.0 → stable
v2.1.0 → new security feature
```

You test:

```text
payment-service → v2.1.0
```

If something breaks:

```text
@Library('platform-shared-library@v2.0.0') _
```

You can roll back.

That's much safer than every repository consuming a moving branch.

---

# 32. Branch vs Tag

You might see:

```groovy
@Library('platform-shared-library@main') _
```

This is convenient for development but risky for production.

Better:

```groovy
@Library('platform-shared-library@v2.0.0') _
```

because it gives deterministic behavior.

---

# 33. Testing the Shared Library

This is another Senior-level point.

Don't manually test every change through 50 application repositories.

Use:

```text
Shared Library
      |
      v
Automated tests
      |
      v
Test application
      |
      v
Release v2.1.0
```

A good workflow:

```text
Developer modifies library
          ↓
Pull Request
          ↓
Unit/Test Pipeline
          ↓
Test with sample applications
          ↓
Security checks
          ↓
Merge
          ↓
Create version/tag
          ↓
Applications upgrade
```

---

# 34. Common Shared Library mistakes

### ❌ Huge Jenkinsfile

```text
500–1000 lines
```

### ❌ Huge `ciPipeline.groovy`

```text
2000 lines
```

### ❌ No versioning

```groovy
@Library('library@main')
```

for production.

### ❌ Hardcoded credentials

Never put:

```groovy
AWS_SECRET_KEY = '...'
```

in the library.

### ❌ Hardcoded environment-specific values

Don't put:

```text
production-account-id
```

everywhere in the library.

Use configuration.

### ❌ Breaking changes without migration

Changing:

```groovy
ciPipeline(application: ...)
```

without maintaining compatibility can break dozens of repositories.

---

# 35. Best practice architecture

I would structure your Shared Library like this:

```text
jenkins-shared-library/
│
├── vars/
│   ├── ciPipeline.groovy
│   ├── terraformPipeline.groovy
│   ├── helmDeploy.groovy
│   ├── securityScan.groovy
│   └── notifyBuild.groovy
│
├── src/
│   └── com/company/
│       ├── AWS.groovy
│       ├── Docker.groovy
│       └── Kubernetes.groovy
│
├── resources/
│   └── com/company/
│       └── templates/
│
└── README.md
```

And repositories:

```text
payment-service/
└── Jenkinsfile

order-service/
└── Jenkinsfile

inventory-service/
└── Jenkinsfile

terraform-vpc/
└── Jenkinsfile
```

---

# 36. What stays in the application repository?

Keep:

```text
Jenkinsfile
Application source
Dockerfile
Application tests
Application configuration
```

Example:

```groovy
@Library('platform-shared-library@v2.0.0') _

ciPipeline(
    application: 'payment-service',
    dockerImage: '123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-service',
    dockerTag: env.BUILD_NUMBER
)
```

---

# 37. What belongs in Shared Library?

Centralized:

```text
Kubernetes Agent configuration
Build framework
Security scanning
AWS authentication patterns
ECR push
Notifications
Timeouts
Workspace cleanup
Standard logging
Error handling
Governance
```

---

# 38. What belongs in Jenkins Global Configuration?

Things like:

```text
Shared Library repository
Credentials needed by Jenkins
Jenkins Cloud
Kubernetes configuration
Plugin configuration
Global tools where appropriate
```

Don't confuse:

```text
Shared Library
```

with:

```text
Jenkins Global Configuration
```

The library is **code**.

Global configuration tells Jenkins **where/how to load and execute that code**.

---

# 39. Interview scenario

### Interviewer:

> "You have 100 microservices. How would you avoid duplicating Jenkins pipeline code?"

### Strong answer:

> "I would introduce a versioned Jenkins Shared Library managed by the Platform Engineering team. The common CI/CD lifecycle—checkout, build, test, security scanning, container build, ECR push, notifications, cleanup, and Kubernetes agent configuration—would live in reusable library functions under `vars/` and, where appropriate, reusable Groovy classes under `src/`. Each application repository would keep a very small Jenkinsfile containing the shared-library version and application-specific configuration. I'd version the library using Git tags so application teams can upgrade in a controlled manner rather than consuming an unpinned branch. This gives us centralized governance and security while still allowing application-specific configuration."

That's a strong Senior-level answer.

---

# 40. Even better interview answer — connect it to EKS

If they ask specifically about your EKS implementation:

> "Our Jenkins Controller runs inside EKS, but builds don't execute on the Controller. The Shared Library defines standardized Kubernetes-based ephemeral agents. When an application invokes the shared pipeline, the Kubernetes plugin creates an Agent Pod from the shared Pod template. The Agent uses a dedicated Kubernetes ServiceAccount, which is mapped to an AWS IAM role through EKS Pod Identity. This allows the pipeline to access ECR, S3 or other AWS services without static AWS credentials. The Shared Library centralizes this architecture, so individual repositories don't have to duplicate Kubernetes, AWS authentication, security scanning and cleanup logic."

That connects:

```text
Shared Library
       ↓
Kubernetes Plugin
       ↓
Ephemeral Agent
       ↓
ServiceAccount
       ↓
EKS Pod Identity
       ↓
IAM Role
       ↓
AWS
```

which is exactly the kind of end-to-end understanding interviewers look for.

---

# 41. The final mental model

Remember this:

```text
┌─────────────────────────────────────────────┐
│ Application Repository                      │
│                                             │
│ Jenkinsfile                                 │
│                                             │
│ @Library('platform-shared-library@v2.0.0')  │
│                                             │
│ ciPipeline(...)                             │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│ Jenkins Shared Library                      │
│                                             │
│ vars/                                       │
│ ├── ciPipeline.groovy                       │
│ ├── terraformPipeline.groovy               │
│ └── helmDeploy.groovy                       │
│                                             │
│ src/                                        │
│ └── com/company/                            │
│     ├── AWS.groovy                          │
│     └── Kubernetes.groovy                   │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
             Jenkins Controller
                      │
                      ▼
             Kubernetes Plugin
                      │
                      ▼
               EKS Agent Pod
                      │
                      ▼
              ServiceAccount
                      │
                      ▼
              EKS Pod Identity
                      │
                      ▼
                  IAM Role
                      │
             ┌────────┼────────┐
             ▼        ▼        ▼
            ECR       S3       EKS
```

### The one sentence to remember

> **"The application repository declares what it wants; the Shared Library defines how the organization executes it."**

That is the core idea behind Jenkins Shared Libraries.


---

# 3. CI Pipeline

This is the classic:

```text
Developer
    │
    ▼
GitHub PR
    │
    ▼
Jenkins
    │
    ▼
Kubernetes Ephemeral Agent
    │
    ├── Checkout
    ├── Lint
    ├── Unit Tests
    ├── SAST
    ├── Dependency Scan
    ├── Docker Build
    ├── Image Scan
    └── Push Image
             │
             ▼
            ECR
```

For Kubernetes-based Jenkins, I'd use **Kaniko** instead of Docker-in-Docker.

### Jenkinsfile

```groovy
pipeline {

    agent {
        kubernetes {
            defaultContainer 'tools'
            retries 2

            yaml '''
apiVersion: v1
kind: Pod
spec:

  containers:

  - name: tools
    image: alpine:3.22
    command:
    - cat
    tty: true

  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.23.2
    command:
    - /busybox/cat
    tty: true
'''
        }
    }

    options {
        skipDefaultCheckout(true)
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = '123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-service'
        IMAGE_TAG  = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Lint') {
            steps {
                container('tools') {
                    sh '''
                        echo "Running lint"
                        # ./gradlew check
                        # or npm run lint
                        # or ruff check .
                    '''
                }
            }
        }

        stage('Unit Tests') {
            steps {
                container('tools') {
                    sh '''
                        echo "Running unit tests"
                        # ./gradlew test
                    '''
                }
            }

            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: '**/test-results/**/*.xml'
                }
            }
        }

        stage('Security Scan') {
            steps {
                container('tools') {
                    sh '''
                        echo "Running SAST / dependency scan"
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('kaniko') {
                    sh '''
                        /kaniko/executor \
                          --context "${WORKSPACE}" \
                          --dockerfile "${WORKSPACE}/Dockerfile" \
                          --destination "${IMAGE_NAME}:${IMAGE_TAG}" \
                          --destination "${IMAGE_NAME}:latest"
                    '''
                }
            }
        }

        stage('Image Scan') {
            steps {
                container('tools') {
                    sh '''
                        echo "Scanning container image"
                        # trivy image ${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }
    }

    post {

        always {
            cleanWs()
        }

        success {
            echo "CI successful: ${IMAGE_NAME}:${IMAGE_TAG}"
        }

        failure {
            echo 'CI pipeline failed'
        }
    }
}
```

### Important interview question

**Why not Docker socket?**

Bad:

```text
Jenkins Pod
    │
    └── /var/run/docker.sock
             │
             ▼
        Kubernetes Node
```

Because the container gets effectively powerful access to the Docker daemon/node.

Better:

```text
Jenkins Pod
    │
    └── Kaniko
          │
          ▼
        Registry
```

No Docker daemon required.

---

# 4. CD Pipeline — Helm Change → GitHub

For a modern Platform Engineering setup, I strongly recommend **not having Jenkins directly run `helm upgrade` against production**.

Instead:

```text
Application CI
      │
      ▼
Container Registry
      │
      │ image = v1.25
      ▼
Jenkins CD
      │
      ├── checkout GitOps repo
      │
      ├── modify Helm values
      │
      ├── helm lint
      │
      ├── git commit
      │
      └── git push
              │
              ▼
          GitHub
              │
              ▼
            ArgoCD
              │
              ▼
             EKS
```

This is a **GitOps CD pipeline**.

### Jenkinsfile

```groovy
pipeline {

    agent {
        kubernetes {
            defaultContainer 'helm'
            retries 2

            yaml '''
apiVersion: v1
kind: Pod
spec:

  containers:

  - name: helm
    image: alpine/helm:3.18.6
    command:
    - cat
    tty: true

  - name: git
    image: alpine/git:2.50.1
    command:
    - cat
    tty: true
'''
        }
    }

    options {
        skipDefaultCheckout(true)
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        GIT_REPO    = 'git@github.com:company/platform-gitops.git'
        GIT_BRANCH  = 'main'

        APP_NAME    = 'payment-service'
        IMAGE_TAG   = "${BUILD_NUMBER}"

        VALUES_FILE = 'apps/payment-service/values-prod.yaml'
    }

    stages {

        stage('Checkout GitOps Repository') {
            steps {

                container('git') {

                    sshagent(credentials: ['github-deploy-key']) {

                        sh '''
                            git clone \
                              --branch "${GIT_BRANCH}" \
                              "${GIT_REPO}" \
                              gitops

                            cd gitops

                            git config user.name "jenkins"
                            git config user.email "jenkins@company.com"
                        '''
                    }
                }
            }
        }

        stage('Update Helm Values') {
            steps {

                container('helm') {

                    dir('gitops') {

                        sh '''
                            echo "Updating image tag..."

                            sed -i \
                              "s/^  tag:.*/  tag: ${IMAGE_TAG}/" \
                              "${VALUES_FILE}"

                            cat "${VALUES_FILE}"
                        '''
                    }
                }
            }
        }

        stage('Helm Lint') {
            steps {

                container('helm') {

                    dir('gitops') {

                        sh '''
                            helm lint \
                              ./charts/${APP_NAME} \
                              -f "${VALUES_FILE}"
                        '''
                    }
                }
            }
        }

        stage('Git Diff') {
            steps {

                container('git') {

                    dir('gitops') {

                        sh '''
                            git diff
                            git status
                        '''
                    }
                }
            }
        }

        stage('Approval') {
            steps {

                timeout(time: 10, unit: 'MINUTES') {

                    input message: 'Push Helm change to GitOps repository?',
                          ok: 'Push'
                }
            }
        }

        stage('Commit and Push') {
            steps {

                container('git') {

                    dir('gitops') {

                        sshagent(credentials: ['github-deploy-key']) {

                            sh '''
                                git add "${VALUES_FILE}"

                                git commit \
                                  -m "chore: update ${APP_NAME} image to ${IMAGE_TAG}"

                                git push origin "${GIT_BRANCH}"
                            '''
                        }
                    }
                }
            }
        }
    }

    post {

        always {
            cleanWs()
        }

        success {
            echo 'GitOps change successfully pushed'
        }

        failure {
            echo 'CD pipeline failed'
        }
    }
}
```

---

# How these 4 differ

| Pipeline           | Main Purpose                | Kubernetes Agent    | Important Tools           |
| ------------------ | --------------------------- | ------------------- | ------------------------- |
| **Terraform VPC**  | Infrastructure provisioning | Terraform Pod       | Terraform, AWS            |
| **Shared Library** | Pipeline standardization    | Dynamic Pod         | Groovy                    |
| **CI**             | Build/test/image            | Multi-container Pod | Git, tests, Kaniko, Trivy |
| **CD / GitOps**    | Deployment promotion        | Helm + Git Pod      | Helm, Git, GitHub, ArgoCD |

---

# The architecture I'd recommend you remember for interviews

```text
                         GitHub
                           │
              ┌────────────┴────────────┐
              │                         │
         Application Repo          GitOps Repo
              │                         ▲
              ▼                         │
           Jenkins CI ────────────────┐ │
              │                       │ │
              ▼                       │ │
             ECR                     │ │
              │                       │ │
              ▼                       │ │
           Jenkins CD ──► Helm ──► GitHub
                                      │
                                      ▼
                                    ArgoCD
                                      │
                                      ▼
                                     EKS
```

And Jenkins itself:

```text
                    Jenkins Controller
                           │
                           │ Kubernetes Plugin
                           ▼
                ┌─────────────────────┐
                │ Kubernetes Cluster  │
                │                     │
                │  Ephemeral Pod      │
                │  ┌───────────────┐  │
                │  │ terraform     │  │
                │  │ helm          │  │
                │  │ git           │  │
                │  │ kaniko        │  │
                │  │ trivy         │  │
                │  └───────────────┘  │
                └─────────────────────┘
```

### Senior-level best practices to mention

1. **Never build on the Jenkins controller.**
2. Use **ephemeral Kubernetes agents**.
3. Use `defaultContainer` and dedicated sidecar containers.
4. Don't mount **Docker socket** into Jenkins pods.
5. Use **IAM roles / workload identity**, not static AWS keys.
6. Store secrets in **Jenkins Credentials / external secret manager**.
7. Pin container/tool versions rather than using `latest`.
8. Use `timeout`, `timestamps`, `disableConcurrentBuilds`.
9. Use `skipDefaultCheckout(true)` when you want explicit checkout control.
10. Keep reusable pipeline logic in **Shared Libraries**.
11. For CD, prefer **GitOps + Argo CD** over Jenkins directly modifying Kubernetes.
12. Use **PR-based promotion** for production rather than blindly pushing to `main`.
13. Add **SAST, dependency scanning and container scanning** to CI.
14. Make Terraform deployment **plan → approval → apply**.
15. Make pipelines **idempotent and retry-safe**.

For your **Senior DevOps interview**, these four examples cover four very different Jenkins concepts: **IaC, pipeline abstraction, CI, and GitOps CD**.
