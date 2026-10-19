# Quickstall

> A practical guide to rebuilding a Windows and Ubuntu WSL DevOps workstation.

## Jump to a section

- [Why I made this](#why-i-made-this)
- [The setup journey](#the-setup-journey)
- [WSL and Git](#1-wsl-and-git)
- [AWS CLI](#2-aws-cli)
- [Python](#3-python)
- [Docker Desktop](#4-docker-desktop)
- [Terraform](#5-terraform)
- [Kubernetes CLI and Helm](#6-kubernetes-cli-and-helm)
- [Azure CLI](#7-azure-cli)
- [Java and Jenkins](#8-java-and-jenkins)
- [Credentials and project files](#credentials-and-project-files)
- [Later additions](#later-additions)

## Why I made this

I recently moved my work to a new laptop. Rebuilding a development environment takes more than installing programs: projects must be cloned again, tools need separate installs in Windows and Ubuntu, PATH settings can be missed, and local credentials or environment files do not come down from Git.

I started Quickstall to turn that move into a learning journey. The goal is to install tools one at a time, verify each one, and understand which shell it belongs in. This is a working guide that will grow as my projects need more tools.

## The setup journey

1. Prepare Windows and install Ubuntu with WSL 2.
2. Install and verify tools in Windows PowerShell.
3. Install and verify their Linux versions in Ubuntu.
4. Clone projects into a development folder.
5. Restore credentials securely and configure each project when needed.

Windows and Ubuntu are separate environments. A Windows install does not automatically install the same CLI inside Ubuntu. Docker Desktop is the exception here: WSL integration lets Ubuntu use its engine.

## 1. WSL and Git

Check WSL in PowerShell:

    wsl --status
    wsl --list --verbose

If Ubuntu is missing:

    wsl --install -d Ubuntu

Restart if prompted, launch Ubuntu, and create your Linux user.

Install Git:

    # PowerShell
    winget install --exact --id Git.Git
    git --version

    # Ubuntu
    sudo apt update
    sudo apt install -y git
    git --version

Set your Git author identity in the environment where you commit:

    git config --global user.name "Your Name"
    git config --global user.email "you@example.com"

Clone a project:

    git clone https://github.com/OWNER/REPOSITORY.git

Clone creates the project folder and Git metadata. Do not run git init first. Private repositories require GitHub authentication.

## 2. AWS CLI

    # PowerShell
    irm https://awscli.amazonaws.com/v2/install.ps1 | iex
    aws --version

    # Ubuntu
    sudo apt update
    sudo apt install -y unzip
    curl -fsSL https://awscli.amazonaws.com/v2/install.sh | bash
    aws --version

Installing the CLI does not configure AWS credentials.

Guide: [AWS CLI installation](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

## 3. Python

    # PowerShell
    winget install 9NQ7512CXL7T -e --accept-package-agreements --disable-interactivity
    python --version
    python -c "print('Python is working in Windows PowerShell')"

    # Ubuntu
    python3 --version
    python3 -c "print('Python is working')"

Ubuntu includes Python 3.

## 4. Docker Desktop

    # PowerShell
    winget install --exact --id Docker.DockerDesktop

Open Docker Desktop. Under **Settings → Resources → WSL Integration**, enable Ubuntu and wait for the engine to start. Verify in both shells:

    docker --version
    docker run hello-world

If Ubuntu reports a socket permission error, add the Linux user to the Docker group, run wsl --shutdown in PowerShell, restart Docker Desktop, and reopen Ubuntu. The Docker group has root-equivalent access inside Ubuntu.

Guide: [Docker Desktop with WSL](https://docs.docker.com/desktop/features/wsl/)

## 5. Terraform

    # PowerShell
    winget install --exact --id Hashicorp.Terraform
    terraform --version

HashiCorp downloads returned 404 on this laptop until the VPN was connected. If Terraform is not found after installation, reopen PowerShell; if needed locate terraform.exe under $env:LOCALAPPDATA\Microsoft\WinGet\Packages and add its folder to PATH.

On Ubuntu 26.04, the HashiCorp APT repository returned 404 for the resolute release and was disabled. Use the official Linux archive if APT does not list your release. This example pins the version tested here; change it to match your project:

    VERSION="1.16.6"
    curl -fSLO "https://releases.hashicorp.com/terraform/${VERSION}/terraform_${VERSION}_linux_amd64.zip"
    curl -fSLO "https://releases.hashicorp.com/terraform/${VERSION}/terraform_${VERSION}_SHA256SUMS"
    grep "terraform_${VERSION}_linux_amd64.zip" "terraform_${VERSION}_SHA256SUMS" | sha256sum -c -
    sudo apt install -y unzip
    unzip "terraform_${VERSION}_linux_amd64.zip"
    sudo install -m 0755 terraform /usr/local/bin/terraform
    rm terraform "terraform_${VERSION}_linux_amd64.zip" "terraform_${VERSION}_SHA256SUMS"
    terraform --version

Guide: [Install Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)

## 6. Kubernetes CLI and Helm

Install kubectl in PowerShell:

    winget install --exact --id Kubernetes.kubectl
    kubectl version --client

Install kubectl in Ubuntu, verifying the downloaded checksum:

    KUBECTL_VERSION="$(curl -L -s https://dl.k8s.io/release/stable.txt)"
    curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
    curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl.sha256"
    echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    rm kubectl kubectl.sha256
    kubectl version --client

Install Helm in PowerShell:

    winget install --exact --id Helm.Helm
    helm version

Install Helm in Ubuntu:

    sudo apt-get update
    sudo apt-get install -y curl gpg apt-transport-https
    curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
    echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    sudo apt-get update
    sudo apt-get install -y helm
    helm version

kubectl and Helm are clients; they need a configured Kubernetes cluster to manage workloads. Helm does not need Java.

Guides: [kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) · [Helm installation](https://helm.sh/docs/intro/install/)

## 7. Azure CLI

    # PowerShell
    winget install --exact --id Microsoft.AzureCLI
    az version

Ubuntu installation:

    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
    sudo mkdir -p /etc/apt/keyrings
    curl -sLS https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/microsoft.gpg > /dev/null
    sudo chmod go+r /etc/apt/keyrings/microsoft.gpg
    AZ_DIST="$(lsb_release -cs)"
    echo "Types: deb
    URIs: https://packages.microsoft.com/repos/azure-cli/
    Suites: ${AZ_DIST}
    Components: main
    Architectures: $(dpkg --print-architecture)
    Signed-By: /etc/apt/keyrings/microsoft.gpg" | sudo tee /etc/apt/sources.list.d/azure-cli.sources
    sudo apt-get update
    sudo apt-get install -y azure-cli
    az version

Installation does not sign in to Azure. Run az login when you are ready to authenticate.

Guide: [Azure CLI on Linux](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-apt)

## 8. Java and Jenkins

Java 21 is installed for Jenkins. Helm does not require Java.

Install Java:

    # PowerShell
    winget install --exact --id EclipseAdoptium.Temurin.21.JDK
    java --version

    # Ubuntu
    sudo apt update
    sudo apt install -y fontconfig openjdk-21-jre
    java -version

Guide: [Jenkins Java support](https://www.jenkins.io/doc/book/platform-information/support-policy-java/)

### Jenkins on Windows

Download and launch the official LTS installer:

    $jenkinsMsi = "$env:TEMP\jenkins.msi"
    Invoke-WebRequest -Uri "https://get.jenkins.io/windows-stable/latest/jenkins.msi" -OutFile $jenkinsMsi
    Start-Process -FilePath "msiexec.exe" -ArgumentList "/i `"$jenkinsMsi`"" -Verb RunAs -Wait

In the wizard, select the Temurin Java 21 home folder. Keep port 8080 if available. Check the service and retrieve the initial password locally:

    Get-Service jenkins
    Get-Content "C:\ProgramData\Jenkins\.jenkins\secrets\initialAdminPassword"

Open http://localhost:8080. Never save the password, cloud credentials, or .env values in this README or in Git. If Jenkins says it is offline during plugin setup, finish without plugins and configure network access later.

### Jenkins in Ubuntu / WSL

The native Jenkins APT installation was not completed during this migration because the package host returned HTTP 503 for its signing key. Do not bypass signature verification; retry using the official guide when the host is reachable.

Another option is to run the official Jenkins image from the Ubuntu shell. Docker Desktop is integrated with WSL, so this uses the Docker Desktop engine and a persistent named volume. Windows Jenkins already uses port 8080, so this maps the container to 8081:

    docker volume create jenkins_wsl_home
    docker run -d --name jenkins-wsl --restart unless-stopped -p 8081:8080 -v jenkins_wsl_home:/var/jenkins_home jenkins/jenkins:lts-jdk21
    docker logs -f jenkins-wsl

Open http://localhost:8081. Retrieve the initial password locally:

    docker exec jenkins-wsl cat /var/jenkins_home/secrets/initialAdminPassword

A fresh Jenkins controller does not restore jobs, plugins, credentials, or build history. Back up the old Jenkins home directory and restore it securely if you need those items.

Guides: [Windows](https://www.jenkins.io/doc/book/installing/windows/) · [Linux](https://www.jenkins.io/doc/book/installing/linux/) · [Docker](https://www.jenkins.io/doc/book/installing/docker/)

## Credentials and project files

Cloning a repository does not restore ignored .env files, local development variables, AWS profiles, Azure sign-ins, or Terraform state.

- Keep .env, dev.tfvars, cloud credentials, access tokens, and Jenkins passwords out of Git.
- Restore secrets from a password manager or encrypted transfer. Do not paste them into chat or this README.
- Check git status --short before committing to ensure sensitive files are ignored.
- Windows and Ubuntu CLI configurations are separate.

## Later additions

- Prometheus and Grafana for monitoring
- A local Kubernetes cluster such as kind or Minikube
- Jenkins plugins and job pipelines
- Project-specific cloud credentials and Terraform state

## Learning note

The main lesson from this move is that a laptop setup is more than a list of downloads. Each tool has a home, a PATH, a version, and sometimes a separate identity or configuration in Windows and Linux. Installing one tool, checking it, and writing down what happened makes the next setup less stressful and the commands easier to understand.

Build → learn → break → fix → understand.
