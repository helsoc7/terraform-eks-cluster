# Terraform für EKS-Cluster
Wir wollen ein EKS-Cluster mit Terraform erstellen. Dazu verwenden wir das EKS-Module von Terraform. 
1. Als erstes definieren wir wieder den aws-Provider mit der Region `eu-central-1`, wobei wir die Region durch eine Variable in der Datei variables.tf definieren.
```
provider "aws" {
  region = var.region
}
```
```
variable "region" {
  description = "Default AWS Region"
  type        = string
  default     = "eu-central-1"
}
```
2. Als nächstes definieren wir uns einen Data Source Block. Im spezifischen nämlich die `aws_availability_zones`, die uns Informationen über die AZs in der AWS-Region gibt, die wir in unserer Konfiguration zuvor festgelegt haben. In unserem Fall suchen wir nach den verfügbaren AZs in `eu-central-1`, bei denen keine spezielle Anmeldung erforderlich ist (`value=["opt-in-not-required"]`).
```
data "aws_availability_zones" "availibility_zones" {
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}
```
3. Als nächstes definieren wir eine lokale Variable innerhalb unserer Terraform-Konfiguration. In diesem Fall ist es der Cluster-Name, welcher wiederum über die Variable `cluster_name`, die in der Datei `variables.tf` definiert ist.
```
locals {
  cluster_name = var.cluster_name
}
```
variables.tf:
```
variable "cluster_name" {
  description = "Name of the Cluster"
  type = string
  default = "budgetbook"
}
```
4. Im vierten Schritt definieren wir das VPC-Module, welches das VPC erstellt, in das unser EKS-Cluster später existiert. Hier werden die Netzwerkkonfigurationen und Eigenschaften definiert, die notwendig sind, um eine sichere und gut strukturierte Netzwerkumgebung für das Kubernetes-Cluster zu gewährleisten. Verwendet wird in diesem Fall das Terraform-Modul unter der source `terraform-aws-modules/vpc/aws`. Das ist ein Community-Modul, das speziell dafür entwickelt wurde, das Erstellen/Verwalten von AWS VPCs zu vereinfachen. Der Name wird hier dynamisch auf dem Namen des EKS-Clusters erstellt. Der IP-Adressraum des VPCs (`cidr`) ist mit `10.0.0.0/16` angegeben. Das ist eine Standard-Auswahl und erlaubt bis zu 65536 IP-Adressen im privaten Netzwerkraum. Als nächstes werden die AZs ausgewählt, wobei die ersten drei verfügbaren AZs gewählt werden aus dem data-source-Block. Als nächstes werden Listen von Subnetzen erstellt, die innerhalb des VPCs erstellt werden sollen. Dabei sind die privaten Subnetze für interne Ressourcen gedacht, die nicht direkt aus dem Internet erreichbar sein sollen, während öffentliche Subnetze für Ressourcen benutzt werden, die eine direkte Verbindung zum Internet benötigen, wie z.B. ein Load Balancer. Als nächstes werden die Einstellungen für die NAT Gateways gesetzt, die es Instanzen in privaten Subnetzen ermöglichen, auf das Internet zuzugreifen, ohne direkt aus dem Internet erreichbar zu sein. `true` bedeutet, dass das NAT Gateway aktiviert ist und mit `single_nat_gateway` gibt es nur ein Gateway für alle Subnetze. Außerdem wird mit `enable_dns_hostname` die DNS-Auflösung aktiviert, was ziemlich nützlich werden kann für die interne Vernetzung der AWS-Dienste. Innerhalb des Modules führen wir außerdem ein Subnet-Tagging durch. Hier werden sie Tags `public_subnet_tags` und `private_subnet_tags` den Sunetzen zugewiesen, um sie als Teil eines bestimmten Kubernetes-Cluster zu kennzeichnen. Die sind nämlich wichtig für die Integration durch Kubernetes-Dienste. Indem wir `"kubernetes.io/cluster/${local.cluster_name}" = "shared"` zeigen wir an, dass das Subnetz zum spezifizierten Cluster gehört und von Clusterressourcen genutzt werden kann. Für das public Subnet gibt `"kubernetes.io/role/elb" = 1 `an, dass dieses Subnetz für externe Load Balancer genutzt werden kann. Für das private Subnet zeigt wiederum `"kubernetes.io/role/internal-elb" = 1` an, dass dieses für interne Load Balancer genutzt werden soll.
```
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "${local.cluster_name}-vpc"

  cidr = "10.0.0.0/16"

  azs = slice(data.aws_availability_zones.availibility_zones.names, 0, 3)

  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                      = 1
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"             = 1
  }

}
```
5. Danach definieren wir das eks-Module für das Einrichten und Verwalten des AWS EKS-Clusters. Das Module verwendet das `terraform-aws-modules/eks/aws`-Module aus der Terraform-Registry. Der Cluster-Name wird dabei auf die lokale Variable `local.cluster_name` gesetzt. Die Cluster-Version gibt eine spezifische Version von Kubernetes an. Die VPC-ID it die ID des VPCs, innerhalb derer das EKS Cluster erstellt wird. Wir nehmen in diesem Fall die ID vom vorher definierten VPC-Module (`module.vpc.vpc_id`). Als Subnet-Ids nehmen wir die privaten Subnetze aus dem definierten VPC-Module (`module.vpc.private_subnets`), um die Sicherheit zu erhöhen, indem die Nodes nicht direkt aus dem Internet erreichbar sind. Als nächstes legen wir fest, dass der Endpunkt des Kubernetes API-Servers vom Internet aus zugänglich sein soll (`cluster_endpoint_public_access = true`). Mit `enable_cluster_creator_admin_permissions = true` erhält das AWS-Konto mit dem wir das Cluster erstellen die Administratorrechte für das Cluster. Als nächstes werden die Standardwerte für `eks_managed_node_group`gesetzt und zwar auf `ami_type` auf ein Amazon Linux 2. Außerdem erstellen wir ein Dictionary von Node Groups, die im Cluster erstellt werden sollen. In diesem Fall gibt es eine Node Group mit dem Namen `nodegroup-1`, den Instanz-Typen `t3.small`und eine gewisse `min_size`, `max_size` und `desired_size`, die alle auf 1 gesetzt sind. Das heißt wir haben immer 1 Node laufen. 
```
module "eks" {
  source = "terraform-aws-modules/eks/aws"

  cluster_name    = local.cluster_name
  cluster_version = "1.29"

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  enable_cluster_creator_admin_permissions = true

  eks_managed_node_group_defaults = {
    ami_type = "AL2_x86_64"
  }

  eks_managed_node_groups = {
    one = {
      name           = "nodegroup-1"
      instance_types = ["t3.small"]

      min_size     = 1
      max_size     = 1
      desired_size = 1
    }
  }
}
```
6. Als nächstes wollen wir den CSI (Elastic Block Store Container Storage Interface) Driver im EKS-Cluster konfigurieren. Dazu legen wir zunächst die AWS IAM Policy Data Source an. 
```
data "aws_iam_policy" "ebs_csi_policy" {
  arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
}
```
Hier steht das für den Daten-Block, der die Daten über eine bestehende IAM-Policy in AWS lädt. Diese werden benötigt für den EBS CSI Driver. Hier steht `arn` für ARN (Amazon Resource Name), also der spezifischen IAM-Policy, die die Berechtigungen für den Betrieb des EBS CSI Drivers in EKS definiert. Danach richten wir die IAM Role mit OIDC (OpenID Connect) ein. 
```
module "iam-assumable-role-with-oidc" {
  source                        = "terraform-aws-modules/iam/aws//modules/iam-assumable-role-with-oidc"
  create_role                   = true
  role_name                     = "AmazonEKSTFEBSCSIRole-${module.eks.cluster_name}"
  provider_url                  = module.eks.oidc_provider
  role_policy_arns              = [data.aws_iam_policy.ebs_csi_policy.arn]
  oidc_fully_qualified_subjects = ["system:serviceaccount:kube-system:ebs-csi-controller-sa"]
}
```
In diesem Modul verwenden wir das Terraform-Modul, das zur Erstellung einer IAM-Rolle mit OIDC-Integration genutz wird. Außerdem geben wir mit `create_role` an, dass wir eine IAM-Rolle erstellen. Der `role_name` gibt den Namen der Rolle an. Die `provider_url` ist die URL des OIDC-Providers, die von EKS für Authentifizierungszwecke bereitgestellt wird. Die `role_policy_arns` sind die ARNs der IAM-Policies, die der Rolle zugewiesen werden sollen, also hier die Policy für den EBS CSI Driver. Außerdem sind die `oidc_fully_qualified_subjects` die Service Accounts, die diese Rolle annehmen dürfen, spezifisch also für den EBS CSI Controller in Kubernetes. Danach richten wir das AWS EKS Addon für den EBS CSI Driver an mithilfe der folgenden Ressource:
```
resource "aws_eks_addon" "ebs-csi" {
  cluster_name             = module.eks.cluster_name
  addon_name               = "aws-ebs-csi-driver"
  addon_version            = "v1.29.1-eksbuild.1"
  service_account_role_arn = module.iam-assumable-role-with-oidc.iam_role_arn
  tags = {
    "eks_addon" = "ebs-csi"
    "terraform" = "true"
  }
}
```
Wir geben dabei den `cluster_name` an, sowie den `addon_name` und die `addon_version`. Außerdem wird die `service_account_role_arn` angegeben, also die ARN der IAM-Rolle, die für den Betrieb des Addons verwendet wird. Als letztes werden Tags definiert für das Addon. Diese Konfiguration sorgt dafür, dass der EBS CSI Driver korrekt im EKS-Cluster installiert und konfiguriert wird, mit allen erforderlichen Berechtigungen, um EBS-Volumes effizient zu verwalten. Durch die Verwendung von OIDC und IAM-Rollen erhöht sich die Sicherheit, indem Kubernetes Service Accounts direkt IAM-Rollen annehmen können, ohne dass AWS-Zugangsschlüssel erforderlich sind.
7. Dann wollen eine `null_resource` definieren mit einem `local-exec` Provisioner. Damit können wir Skripte/Befehle auf der lokalen Maschine ausführen, die nicht direkt durch Terraform-Ressourcen abgedeckt werden können. resource `null_resource` `kubectl: null_resource` ist ein spezieller Ressourcentyp in Terraform, der keine eigentlichen Aktionen auf einer Infrastruktur ausführt. Stattdessen wird er verwendet, um benutzerdefinierte Logik in das Terraform-Skript einzufügen, typischerweise über Provisioner. In diesem Fall wird der `null_resource` als Träger für den `local-exec` Provisioner verwendet. provisioner "local-exec": Dies ist ein Provisioner-Typ, der einen Befehl auf der Maschine ausführt, von der aus Terraform ausgeführt wird. Er wird oft verwendet, um Anpassungen oder Konfigurationen vorzunehmen, die außerhalb der Terraform-Managementsphäre liegen.
command: Der spezifische Befehl, der ausgeführt wird. Hier wird der AWS CLI Befehl `aws eks --region ${var.region} update-kubeconfig --name ${local.cluster_name}` verwendet. Dieser Befehl aktualisiert die lokale Kubernetes-Konfigurationsdatei (kubeconfig), um den Zugang zum EKS Cluster zu ermöglichen. Es wird der Cluster-Name und die AWS-Region verwendet, die als Variablen angegeben sind. depends_on: Dieses Argument ist eine Liste von Ressourcen, von denen die null_resource abhängig ist. In diesem Fall ist die Ressource vom module.eks abhängig. Dies stellt sicher, dass der local-exec Provisioner erst ausgeführt wird, nachdem der EKS Cluster vollständig eingerichtet wurde. Ohne diese Abhängigkeit könnte Terraform versuchen, den local-exec Befehl auszuführen, bevor der Cluster bereit ist, was zu Fehlern führen würde. Die Verwendung von null_resource mit local-exec in diesem Kontext ermöglicht es, administrative oder konfigurative Aufgaben wie das Aktualisieren der kubeconfig durchzuführen, die notwendig sind, um mit dem neu erstellten EKS Cluster zu interagieren.
```
resource "null_resource" "kubectl" {
  provisioner "local-exec" {
    command = "aws eks --region ${var.region} update-kubeconfig --name ${local.cluster_name}"
  }
  depends_on = [module.eks]
}
```
8. Wir definiere ferner noch Ausgaben in der `output.tf`:
```
output "availibility_zones" {
  value = data.aws_availability_zones.availibility_zones.names
}

output "cluster_endpoint" {
  description = "Endpoint for EKS control plane"
  value       = module.eks.cluster_endpoint
}
```
Hier ist es der Wert von `data.aws_availability_zones.availibility_zones.names`, also eine Liste der Namen der Verfügbarkeitszonen, die von der Data Source aws_availability_zones abgerufen werden. Dies kann hilfreich sein, um zu verstehen, in welchen AWS-Availability Zones die Ressourcen verteilt wurden oder verteilt werden können.
Der Wert dieser Ausgabevariablen ist `module.eks.cluster_endpoint`, der den Endpunkt des Kubernetes Control Plane angibt, der vom EKS-Modul bereitgestellt wird. Dieser Endpunkt ist wichtig, um direkte Interaktionen mit dem Kubernetes Cluster zu ermöglichen, beispielsweise beim Konfigurieren von kubectl oder anderen API-basierten Interaktionen mit dem Cluster.
9. Dann wieder `tf init`, `tf plan` und `tf apply` ausführen... dann sollte irgendwann das Cluster da sein :-) 