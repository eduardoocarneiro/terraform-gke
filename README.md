# Provisionando um cluster GKE com Terraform (Google Cloud)

O Google Kubernetes Engine (GKE) é um serviço Kubernetes totalmente gerenciado para implantação, gerenciamento e escalonamento de aplicações em contêineres na Google Cloud.

Neste tutorial vamos implantar um cluster GKE de dois nós usando Terraform. Este cluster será distribuído em várias zonas para manter a alta disponibilidade. Em seguida configuraremos o **kubectl** usando o *output* do Terraform para implantar um painel do Kubernetes no cluster.

# Pré-requisitos

O tutorial a seguir assume que você tem alguma familiaridade com kubernetes e kubectl. Assume também que você possui familiaridade com os comandos básicos do Terraform, tais como **terraform plan** e **terraform apply**.

Para este tutorial nós vamos precisar de:

* Uma [conta GCP](console.cloud.google.com)
* Um gcloud SDK configurado
* kubectl

### gcloud SDK

Para o Terraform rodar as operações no seu ambiente, você precisa instalar e configurar o **gcloud SDK tool**. Para instalá-lo siga [essas instruções](https://cloud.google.com/sdk/docs/quickstarts) de acordo com o seu sistema operacional.

Após a instalação você deverá iniciá-lo com o seguinte comando:

    $ gcloud init

Isso irá autorizar o SDK a acessar a cloud da Google usando suas credenciais de acesso e adicionar o SDK ao PATH. Esse passo requer que você efetue o login e selecione o projeto que você deseja trabalhar.

Por último, adicione sua conta ao **Application Default credentials (ADC)**. Isso permitirá que o Terraform acesse essas credenciais para provisionar os recursos necessários na nuvem da Google.

    $ gcloud auth application-default login

### kubectl

Para instalar o **kubectl** siga [Essas instruções](https://kubernetes.io/docs/tasks/tools/) e escolha o tipo de instalação baseada no seu sistema operacional.

# Configurando e inicializando o Terraform workspace

No seu terminal, clone o [seguinte repositório](https://github.com/eduardoocarneiro/terraform-gke.git). Ele contém o exemplo de configuração usado neste tutorial.

    $ git clone https://github.com/eduardoocarneiro/terraform-gke.git

Agora você pode explorar esse repositório acessando o seu diretório.

    $ cd terraform-gke

Você irá encontrar quatro arquivos usados para provisionar uma VPC, as sub-redes e um cluster GKE.

* [vpc.tf](https://github.com/eduardoocarneiro/terraform-gke/blob/main/vpc.tf) provisiona uma VPC e uma sub-rede. Uma nova VPC será criada, então ela não impactará em recursos já existentes. Esse arquivo gera como saída **region**.
* [gke.tf](https://github.com/eduardoocarneiro/terraform-gke/blob/main/gke.tf) Provisiona um cluster GKE e um **managed node pool** separado. Isso permite que você customize o seu cluster kubernetes profile, isso é interessante quando alguns Pods requerem mais recursos que outros. Você pode ler mais sobre iso [aqui](https://cloud.google.com/kubernetes-engine/docs/concepts/node-pools). O número de nós no node pool é definido na variável **gke_num_nodes** desse arquivo.
* [terraform.tfvars](https://github.com/eduardoocarneiro/terraform-gke/blob/main/terraform.tfvars) Esse arquivo é um template para as variáveis **project_id** e **region**
* [versions.tf](https://github.com/eduardoocarneiro/terraform-gke/blob/main/versions.tf) Configura as versões do Terraform para "ao menos 0.14".

### Atualize o seu arquivo terraform.tfvars

Substitua os valores necessários no seu **terraform.tfvars** com o seu **project_id** e **region**. O Terraform irá usar esses valores para *alcançar* o seu projeto enquanto provisiona os recursos necessários. Seu arquivo **terraform.tfvars** deverá parecer com esse abaixo:

    # terraform.tfvars
    project_id = "REPLACE_ME"
    region     = "us-central1"

O **project_id** pode ser encontrado com o comando abaixo:

    $ gcloud config get-value project

A região por padrão é **us-central1**. Você pode ver a lista completa das regiões [aqui](https://cloud.google.com/compute/docs/regions-zones).

### Inicialize o Terraform workspace

Agora inicialize seu Terraform workspace. Isso irá baixar os providers necessários e inicializar o seu ambiente de trabalho conforme os valores do seu arquivo **terraform.tfvars**.

    $ terraform init
    
    Initializing the backend...

    Initializing provider plugins...
    - Reusing previous version of hashicorp/google from the dependency lock file
    - Installing hashicorp/google v3.52.0...
    - Installed hashicorp/google v3.52.0 (signed by HashiCorp)

    Terraform has been successfully initialized!

    You may now begin working with Terraform. Try running "terraform plan" to see
    any changes that are required for your infrastructure. All Terraform commands
    should now work.

    If you ever set or change modules or backend configuration for Terraform,
    rerun this command to reinitialize your working directory. If you forget, other
    commands will detect it and remind you to do so if necessary.

# Provisionando o cluster

[Compute Engine API](https://console.developers.google.com/apis/api/compute.googleapis.com/overview) e [Kubernetes Engine API](https://console.cloud.google.com/apis/api/container.googleapis.com/overview) são requeridos para o comando **terraform apply** funcionar corretamente. Ambos precisam ser habilitados no seu projeto antes de continuarmos.

Após iniciar as APIs, no diretório que você iniciou o Terraform, rode o comando **terraform apply**. A saída do seu terminal  deve indicar que o plano está sendo executado e que recursos estão sendo criados.

    $ terraform apply
    
    An execution plan has been generated and is shown below.
    Resource actions are indicated with the following symbols:
    + create

    Terraform will perform the following actions:

    ## ...

    Plan: 4 to add, 0 to change, 0 to destroy.

    ## ...

Você pode perceber que o Terraform irá provisionar a VPC, sub-rede, o cluster GKE e o node pool. Confirme o *apply* com a opção **yes**. Caso deseje executar sem a necessidade de confirmação execute o comando com a opção de auto-approve:

    $ terraform apply -auto-approve

Esse processo levará aproximadamente 10 minutos. Após o sucesso na execução, o seu terminal exibirá as saídas (outputs) definidas em [vpc.tf](https://github.com/eduardoocarneiro/terraform-gke/blob/main/vpc.tf) e [gke.tf](https://github.com/eduardoocarneiro/terraform-gke/blob/main/gke.tf).

    Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

    Outputs:

    kubernetes_cluster_host = "35.232.196.187"
    kubernetes_cluster_name = "dos-terraform-edu-gke"
    project_id = "dos-terraform-edu"
    region = "us-central1"

# Configurando o kubectl

Agora que você provisionou seu cluster GKE, você precisa configurar o kubectl para administrar esse cluster. Rode o seguinte comando para recuperar as credenciais de acesso para o seu cluster e automaticamente configurar o kubectl.

    gcloud container clusters get-credentials $(terraform output -raw kubernetes_cluster_name) --region $(terraform output -raw region)

O **Kubernetes ckuster name** e **region** correspondem às variáveis de saída (outputs) mostradas após o *Terraform run* ter sido executado com sucesso.
