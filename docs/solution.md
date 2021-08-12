
## Challenge 0: Goal: Setup all prerequisites.
Minikube version:
-----------------
```
PS C:\Users\seshanmugavel> minikube version
minikube version: v1.22.0
commit: a03fbcf166e6f74ef224d4a63be4277d017bb62e
```

Terraform version:
------------------
```
C:\Users\seshanmugavel\Documents\Desktop Backup\Senthil\freeletics_techops_challenge\techops-challenge\terraform>terraform -version
Terraform v0.15.5
on windows_amd64
+ provider registry.terraform.io/carlpett/sops v0.5.2
+ provider registry.terraform.io/hashicorp/helm v1.3.2
+ provider registry.terraform.io/hashicorp/kubernetes v1.13.3
+ provider registry.terraform.io/hashicorp/null v2.1.2

```

sops version:
-------------
C:\Users\seshanmugavel>sops -v
```
sops 3.4.0
[info] sops 3.7.1 is available, update with `go get -u go.mozilla.org/sops/cmd/sops`

```

## Challenge 1: Goal: Access the Jenkins
-------------------------------------

When I execute terraform plan, I got below error.

```
Error: Error getting data key: 0 successful groups required, got 0
│
│   with data.sops_file.secrets,
│   on secrets.tf line 1, in data "sops_file" "secrets":
│    1: data "sops_file" "secrets" {

```

I tried to troubleshoot the issue, issue seems to be a version mismatch. I'm not able to upgrade sops to latest version (v3.7.1).

Update as on 12-8-2021:

Checksum doesn't match for carlpett/sops provider plugin.

```
C:\Users\seshanmugavel\Documents\Desktop Backup\Senthil\freeletics\techops-challenge\terraform>terraform init
Initializing modules...
- kubedoom in ..\modules\kubedoom

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/null from the dependency lock file
- Reusing previous version of carlpett/sops from the dependency lock file
- Reusing previous version of hashicorp/helm from the dependency lock file
- Reusing previous version of hashicorp/kubernetes from the dependency lock file
- Installing carlpett/sops v0.5.2...
- Installing hashicorp/helm v1.3.2...
- Installed hashicorp/helm v1.3.2 (signed by HashiCorp)
- Installing hashicorp/kubernetes v1.13.3...
- Installed hashicorp/kubernetes v1.13.3 (signed by HashiCorp)
- Installing hashicorp/null v2.1.2...
- Installed hashicorp/null v2.1.2 (signed by HashiCorp)
╷
│ Error: Failed to install provider
│
│ Error while installing carlpett/sops v0.5.2: archive has incorrect checksum zh:b4fbadc9217534821862c0f9870b07d4ff8442aa8a137492f75abfea3ed58d46 (expected
│ zh:8c2a99e69c9628985c418fa37a8e8986af3aa7f1de3e04cfa69b4ad8f3901b7b)
```

Note: I will be proceeding with my solution without coding.

**Solution:**
Inorder to make the Jenkins accessible from local machine, Kubetnetes service should be defined with the type as NodePort so Jenkins can be accessed outside the Kubernetes cluster.

Below code needs to be placed in modules/kubedoom/main.tf

```
resource "kubernetes_service" "kubedoom-service" {
  metadata {
    name = "kubedoom-service"
  }
  spec {
    selector = {
      App = "kubedoom"
    }
    port {
      node_port   = 30201
      port        = 80
      target_port = 5900
    }

    type = "NodePort"
  }
}

```

The encryption seems to be asymmetric encryption algorithm. The public key is used for encryption. The person with the public key can and can only encrypt, and it can be distributed to any organization or individual; The private key is used for decryption, and can only be used to decrypt the information encrypted by the public key paired with the private key. Anyone with the private key can decrypt it.  

As the credentials are already encrypted, using sops tool, Below id and password should be decrypted using below command.

```
 sops -d environment.json
```
-Look for the output for the Key: Jenkins.Users

```json
"users": [
			{
				"id": "ENC[AES256_GCM,data:wd5Y1I0=,iv:nJmk1kZNwAAvYd97vz1BZHxfoEotmSE6hxTMz5kHj0E=,tag:Lst9yO/jtP4uT9cOCyhZ1g==,type:str]",
				"password": "ENC[AES256_GCM,data:tj32NYzfBv9o3w==,iv:adUYiTDsI5569XC6dBVNjTxUTeXUyoWBvjO3TMqXGUg=,tag:4rA9WeAIekMHe5K8w7PQXA==,type:str]"
			},
			{
				"id": "ENC[AES256_GCM,data:Iw5P+gaVjw==,iv:kKpwhb7IBzDr2/mQzO7Dj+abBcTXkkH9dxdhePukUlc=,tag:oRednh9TAxTJ8jDu5EGuzw==,type:str]",
				"password": "ENC[AES256_GCM,data:dOSTlo/tfZ0=,iv:zyC9Scc5PEKnMp+fqHIEuguEvjgjVO66TjNe4KpRzHI=,tag:NlcUW7aemckmz3iaeQ1jBg==,type:str]"
			}
		]
```

I tried to decrypt the environment.json file looks like the fields might get reorderd, Message Authentication Code (MAC) Mismatch Error occurs when the computed MAC does not match the expected ones

```
C:\Users\seshanmugavel\Documents\Desktop Backup\Senthil\freeletics_techops_challenge\techops-challenge\terraform>sops -d environment.json
MAC mismatch. File has F3B838355B8685969C570D3D6448D6B247FD3F8588D1EDE8F9205BC08AE16ECFD5C1A811E685B1584ABB024A85D7077B4298E69A3502000A789EC96EEDC8FFCE, computed 5A634FABD8A3C227E1D846465EF9CFC6CDA02609E368328F208F41FAD209749200B9048D6ABA756B3061EA8D03D0BFCD448CEE43EB88E0CC59EEDF16010FABFA

```

I saw the code base has been updated with new MAC. I tried with the new MAC. It works.

Here is the Jenkins Users Credentials.

```
"users": [
                        {
                                "id": "admin",
                                "password": "t0p!secret"
                        },
                        {
                                "id": "visitor",
                                "password": "Ock*7C)2"
                        }
                ]

```

## Challenge 2: Goal: Cipher all credentials
----------- 

**Solution:**

We should be able to pick the fields needs to be encrypted by including the keys in encrypted_regex field.
The below code from .sops.yaml file doesn't include the key "string" in encrypted_regex, so the encryption is not performed.

```
creation_rules:
  - path_regex: .*\.json$
    encrypted_regex: '^(user.*|pass.*|secret.*|.*[Bb]earer.*|.*[Kk]ey|.*[Kk]eys|salt|sentry.*|.*[Tt]oken)$'
    pgp: >-
        BCD0654D6FA4AB48E40CDE7F1C854CC81B4C2229

```

But in general, we should not leave the plain text secrets in json file unless it's required. As a best practice, we should include the "string" key in encrypted_regex.


## Challenge 3a: Goal: Make Kubedoom work on Minikube with a single terraform apply.
-------------

**Solution:**

Users expect applications to be available all the time and developers are expected to deploy new versions of them several times a day. In Kubernetes this is done with rolling updates. 

Rolling updates allow Deployments’ update to take place with zero downtime by incrementally updating Pods instances with new ones. The new Pods will be scheduled on Nodes with available resources.

Rolling updates allow us to update the configuration of the pods gradually, and Deployments offer many options to control this process.

The most important option to configure rolling updates is the update strategy. In Deployment manifest, spec.strategy.type has two possible values:

RollingUpdate: New pods are added gradually, and old pods are terminated gradually
Recreate: All old pods are terminated before any new pods are added

In most cases, RollingUpdate is the preferable update strategy for Deployments. Recreate can be useful when running a pod as a singleton, and having a duplicate pod for even a few seconds is not acceptable.

When using the RollingUpdate strategy, there are two more options that let fine-tune the update process:

maxSurge: The number of pods that can be created above the desired amount of pods during an update
maxUnavailable: The number of pods that can be unavailable during the update process

In order to use the rolling update for kubedoom app, below code needs to be added in kubernetes_deployment resource

```
strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

Below code needs to be added in kubernetes_deployment resource under spec by providing the limit, requests resources
```
resources {
            limits = {
              cpu    = "0.5"
              memory = "512Mi"
            }
            requests = {
              cpu    = "250m"
              memory = "50Mi"
            }
          }
```