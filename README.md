# Deploying a Flask API on AWS

This API is part of my training on UDACITY, and allows students to understand the deployment of APIs on the AWS cluster.

The purpose of this application is to provide learners with the skills to containerize and deploy a Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

#### The functionalities covered by the API are :

  1. Display health check as text.
  2. Authentication with email and password.
  3. Display unencrypted content after authentication.

## Getting started
To work with this project, you must know python precisely the FLASK framework, and Docker.

### Configuration for local development

#### Install depencies
1. **Python version between 3.7 and 3.9** - Follow instructions to install the latest version of python for your platform in the [python docs](https://docs.python.org/3/using/unix.html#getting-and-installing-the-latest-version-of-python)
2. **Virtual Environment** - We recommend working within a virtual environment whenever using Python for projects. This keeps your dependencies for each project separate and organized. Instructions for setting up a virual environment for your platform can be found in the [python docs](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/)
3. **PIP Dependencies** - Once your virtual environment is setup and running, install the required dependencies with following command:

```bash
pip install -r requirements.txt
```
### Prerequisites

* Docker Desktop - Installation instructions for all OSes can be found [here](https://docs.docker.com/install/?target=_blank).
* Git: [Download and install Git](https://git-scm.com/downloads/?target=_blank) for your system. 
* Code editor: You can [download and install VS code](https://code.visualstudio.com/download/?target=_blank) here.
* AWS Account
* Command line utilities : AWS CLI, EKSCTL AND KUBECTL

### Update configurations
Before starting the application, you must create an ".env_file" file to put the values ​​of JWT_SECRET and LOG_LEVEL

To run the server, execute:

```bash
python main.py
```
Default URL should be http://127.0.0.1:8080/

### Testing
Tests are not required to run the API. But if you contribute, please run the tests before pushing to GitHub.

## API Reference

### Getting Started
- Base URL: http://127.0.0.1:8080/
- Authentication: This the application require authentication to get content.

### Error Handling
All HTTP errors are handled and returned as JSON objects in the following format:

```bash
{
  "error": 404,
  "message": "The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.",
  "success": false
}
```

#### Endpoints

#### Get health check
```http
  GET /
```
```http
  POST /
```
- General: returns the response 'Healthy'.
- Permissions: not require
- Sample: ```bash curl http://127.0.0.1:8080/ ```

```bash
{
  "Healthy" 
}
```

#### Authentication
```http
  POST /auth
```
| Parameter | Type     | Description                       |
| :-------- | :------- | :-------------------------------- |
| `email`      | `string` | **Required**. Your email |
| `password`      | `string` | **Required**. Your password |

- General: returns a JWT token.
- Permissions: not require
- Sample: `curl -X POST -H "Content-Type: application/json" -d '{"email": "demo@demo.com", "password": "demo"}' http://127.0.0.1:8080/auth`

```bash
{
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NjM2Mzk4NDMsIm5iZiI6MTY2MjQzMDI0MywiZW1haWwiOiJkZW1vQGRlbW8uY29tIn0.nf8iwPUVLT9LSQHEk..........."
}
```

#### Get content
```http
  GET /contents
```
| Parameter | Type     | Description                       |
| :-------- | :------- | :-------------------------------- |
| `token`      | `string` | **Required**. Your authentication token |

- General: returns the un-encrpyted contents of that token.
- Permissions: requires a valid token
- Sample: `curl -H "Authorization: Bearer {token}" http://127.0.0.1:8080/contents `
```bash
{
  "email": "demo@demo.com", 
  "exp": 1663639843, 
  "nbf": 1662430243
}
```

## Deployment on AWS

### Container test

Before deployment, you need to build the image and run the container to make sure everything works fine locally.

You need to start creating the application image.
Navigate to the project folder where the Dockerfile is located and run the following command:

```bash
  docker build --tag name:tag .
```
After creating the image, you need to launch the container and test

```bash
  docker run -d -p 5000:8080 --name myFlaskContainer image-name
```
`image-name` is name of your image.

If your container is working and responding to requests, then you can continue with the deployment. If not, this must be corrected before continuing.

### Deployment

### Create a cluster EKS and IAM role

First, create a cluster with eksctl command line tool
```bash
  eksctl create cluster --name simple-jwt-api
```
The command will create a cluster with the name "simple-jwt-api" and a nodegroup containing two "m5.large" nodes, wait for status to be CREATE_COMPLETE

Navigate in the project to find the "trust.json" file. replace <ACCOUNT_ID> with your account ID.

Use the following command to get your account ID:
```bash
  aws sts get-caller-identity --query Account --output text
```
Save the file and type the following command to create the role:
```bash
  aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'
```
It must be in the folder that the file. This IAM role allows CodeBuild to access the EKS cluster.

Now you need to create the permission to allow certain actions.

Navigate to the project folder to find the "iam-role-policy.json" file then run the following command:
```bash
  aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json
```
### Allow the new role to access the cluster
You must add the role to "aws-auth ConfigMap" for it to access the cluster.

Get the current config map and save it to a file with the following command:

For Mac/Linux: the file will be created in the path "/System/Volumes/Data/private/tmp/aws-auth-patch.yml".

```bash
  kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml
```

For Windows: the file will be created in the current working directory
```bash
  kubectl get -n kube-system configmap/aws-auth -o yaml > aws-auth-patch.yml
```
Open the aws-auth-patch.yml file using any editor, such as the VS Code editor

* Mac/Linux
```bash
  code /System/Volumes/Data/private/tmp/aws-auth-patch.yml
```
* Windows
```bash
  code aws-auth-patch.yml
```
Add the following group in the data → mapRoles section of this file.

```bash
    - groups:
      - system:masters
      rolearn: arn:aws:iam::<ID_COMPTE>:role/UdacityFlaskDeployCBKubectlRole
      username: build
  ```
replaces <ACCOUNT_ID> with your account ID.

Update your cluster configuration map with the following command:

* Mac/Linux
```bash
  kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
```
* Windows
```bash
  kubectl patch configmap/aws-auth -n kube-system --patch "$(cat aws-auth-patch.yml)"
```
The command above should show you "configmap/aws-auth patched" as a response.

### Generate a Github access token
Log in to your GitHub account to generate an access token that will allow CodePipeline to monitor changes.

A token can be generated [here](https://github.com/settings/tokens/?target=_blank). You need to generate the token with full control of private repositories as shown in the image below. Be sure to save the token in a secure location.

### Create a pipeline using the CloudFormation template

Navigate to the project folder to find the file
CloudFormation template "ci-cd-codepipeline.cfn.yml" and adapt configurations.

The "ci-cd-codepipeline.cfn.yml" file is the template used to create the CodePipeline and CodeBuild pipeline.

Modify the following values ​​in the file

| Parameter | Field     | Possible value                       |
| :-------- | :------- | :-------------------------------- |
| `EksClusterName`      | `Default` | « simple-jwt-api » Name of the EKS cluster you created |
| `GitSourceRepo`      | `Default` | « deployment-and-containerization » Github repository name |
| `GitBranch`      | `Default` | « main » Name of the branch you want to link to the Pipeline |
| `GitHubUser`      | `Default` | Your Github username |
| `KubectlRoleName`      | `Default` | « UdacityFlaskDeployCBKubectlRole » We created this role earlier |

Save the file. Go to the CloudFormation service in the AWS console. Press Create stack button and choose "ci-cd-codepipeline.cfn.yml" as model file.

Before you finish, you enter your github token and other information.

Finish creating the stack and wait for the stack to have the status "CREATE_COMPLETE".

### Set a secret using AWS Parameter Store
Add the following to the end of the buildspec.yml file (pay attention to indentation):
```bash
    env :
       parameter-store :
       JWT_SECRET : JWT_SECRET
```

Put the secret in AWS Parameter Store with the following command:
```bash
    aws ssm put-parameter --name JWT_SECRET --overwrite --value "YourSecretJWT" --type SecureString
```
To delete, run the following command:
```bash
    aws ssm delete-parameter --name JWT_SECRET
```
**Note**: In the buildspec.yml file, use the same KUBECTL version (or closer) that you used when creating an EKS cluster (run "kubectl version" in your local terminal).

* Now deploy your code to the GitHup repository
* On the AWS console go to CodeBuild then start the compilation process by clicking on the "Start compilation" button in the CodeBuild dashboard.

When a build fails, you can check the logs to see any errors that may have occurred.

When the compilation works, type the following command to get the external IP address to test:
```bash
    kubectl get services simple-jwt-api -o wide
```

### Add tests to compilation
Pre-deployment testing allows you to test the code before production.

You can test with tests that pass and then with tests that do not pass to observe the compilation.

1. The tests are in the test_main.py file, modify them.
2. Open the buildspec.yml file.
3. In the prebuild section, add the following lines:
```bash
    - pip3 install -r requirements.txt
    - python -m pytest test_main.py
```
4. Save the file and push to Github.

If your tests pass, then the application will be deployed. Otherwise the application will not be deployed.

## Authors
This project is the result of the work of the Udacity team and me.

The base of the project was made by the Udacity team and I made corrections to make it work

- [Udacity](https://www.udacity.com/)
- [Bakary FOFANA](https://github.com/FOFANA12)

## Acknowledgements

 - [Udacity](https://www.udacity.com/) 