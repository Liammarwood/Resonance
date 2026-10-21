# Black_Jack

This repository now includes a Unity MVP script set for the **Resonance** endless arcade game under `Assets/Scripts`:

- `GameManager`
- `RingController`
- `PulseSpawner`
- `Pulse`
- `ScoreManager`
- `InputManager`
- `UIManager`

These scripts implement the core loop, difficulty scaling, score/combo logic, and start/game-over state flow for an Android portrait setup.

## Android CI release workflow

`/.github/workflows/android-release.yml` builds Android APK + AAB on every push to `master` and publishes them in a GitHub Release tagged `v1.0.<run_number>`.

Required repository secrets:

- `UNITY_LICENSE`
- `UNITY_EMAIL`
- `UNITY_PASSWORD`


;
 = addon("eks-pod-identity-agent", cluster, null, 




// App.kt
// Secure, highly available EKS cluster with Pulumi (Kotlin on the Pulumi Java SDK).
// Mixes three Pulumi packages in one program:
//   - com.pulumi:aws         -> VPC, subnets, NAT, KMS, IAM, EKS add-on
//   - com.pulumi:eks         -> EKS cluster + managed node group (higher-level component)
//   - com.pulumi:kubernetes  -> aws-node patch + Cilium Helm release
//
// Required stack config:
//   pulumi config set aws:region ap-southeast-2
//   pulumi config set publicAccessCidrs "203.0.113.10/32,198.51.100.0/24"
// Optional:
//   pulumi config set k8sVersion 1.34
//   pulumi config set ciliumVersion 1.18.3
//   pulumi config set adminRoleArn arn:aws:iam::123456789012:role/PlatformAdmin
//   pulumi config set nodeInstanceType m6i.large

package myproject

import com.pulumi.Context
import com.pulumi.Pulumi
import com.pulumi.aws.ec2.Eip
import com.pulumi.aws.ec2.EipArgs
import com.pulumi.aws.ec2.InternetGateway
import com.pulumi.aws.ec2.InternetGatewayArgs
import com.pulumi.aws.ec2.NatGateway
import com.pulumi.aws.ec2.NatGatewayArgs
import com.pulumi.aws.ec2.RouteTable
import com.pulumi.aws.ec2.RouteTableArgs
import com.pulumi.aws.ec2.RouteTableAssociation
import com.pulumi.aws.ec2.RouteTableAssociationArgs
import com.pulumi.aws.ec2.Subnet
import com.pulumi.aws.ec2.SubnetArgs
import com.pulumi.aws.ec2.Vpc
import com.pulumi.aws.ec2.VpcArgs
import com.pulumi.aws.ec2.inputs.RouteTableRouteArgs
import com.pulumi.aws.eks.inputs.AccessPolicyAssociationAccessScopeArgs
import com.pulumi.aws.eks.inputs.AddonPodIdentityAssociationArgs
import com.pulumi.kubernetes.meta.v1.inputs.ObjectMetaArgs
import com.pulumi.kubernetes.storage.v1.StorageClass
import com.pulumi.kubernetes.storage.v1.StorageClassArgs
import com.pulumi.aws.eks.inputs.NodeGroupScalingConfigArgs
import com.pulumi.aws.eks.inputs.NodeGroupTaintArgs
import com.pulumi.aws.eks.inputs.NodeGroupUpdateConfigArgs
import com.pulumi.aws.iam.Role
import com.pulumi.aws.iam.RoleArgs
import com.pulumi.aws.iam.RolePolicyAttachment
import com.pulumi.aws.iam.RolePolicyAttachmentArgs
import com.pulumi.aws.kms.Key
import com.pulumi.aws.kms.KeyArgs
import com.pulumi.core.Output
import com.pulumi.eks.Cluster
import com.pulumi.eks.ClusterArgs
import com.pulumi.eks.ManagedNodeGroup
import com.pulumi.eks.ManagedNodeGroupArgs
import com.pulumi.eks.enums.AuthenticationMode
import com.pulumi.eks.inputs.AccessEntryArgs
import com.pulumi.eks.inputs.AccessPolicyAssociationArgs
import com.pulumi.eks.inputs.CoreDnsAddonOptionsArgs
import com.pulumi.kubernetes.Provider
import com.pulumi.kubernetes.ProviderArgs
import com.pulumi.kubernetes.apps.v1.DaemonSetPatch
import com.pulumi.kubernetes.apps.v1.DaemonSetPatchArgs
import com.pulumi.kubernetes.apps.v1.inputs.DaemonSetSpecPatchArgs
import com.pulumi.kubernetes.core.v1.inputs.PodSpecPatchArgs
import com.pulumi.kubernetes.core.v1.inputs.PodTemplateSpecPatchArgs
import com.pulumi.kubernetes.helm.v3.Release
import com.pulumi.kubernetes.helm.v3.ReleaseArgs
import com.pulumi.kubernetes.helm.v3.inputs.RepositoryOptsArgs
import com.pulumi.kubernetes.meta.v1.inputs.ObjectMetaPatchArgs
import com.pulumi.resources.ComponentResourceOptions
import com.pulumi.resources.CustomResourceOptions
import com.pulumi.aws.eks.Addon as AwsEksAddon
import com.pulumi.aws.eks.AddonArgs as AwsEksAddonArgs

fun main() {
    Pulumi.run { ctx -> stack(ctx) }
}

private fun stack(ctx: Context) {
    val cfg = ctx.config()
    val region = ctx.config("aws").require("region")
    val name = "secure-eks"

    val publicAccessCidrs = cfg.require("publicAccessCidrs")
        .split(",").map { it.trim() }.filter { it.isNotEmpty() }
    require(publicAccessCidrs.none { it == "0.0.0.0/0" }) {
        "publicAccessCidrs must not contain 0.0.0.0/0"
    }
    val k8sVersion = cfg.get("k8sVersion").orElse("1.34")
    val ciliumVersion = cfg.get("ciliumVersion").orElse("1.18.3")
    val instanceType = cfg.get("nodeInstanceType").orElse("m6i.large")
    val adminRoleArn = cfg.get("adminRoleArn").orElse(null)

    // Three AZs for availability (control plane is multi-AZ by default; nodes/NAT follow these)
    val azs = listOf("a", "b", "c").map { "$region$it" }
    val baseTags = mapOf("Project" to name, "ManagedBy" to "pulumi")

    // ---------------------------------------------------------------------
    // Networking (pulumi-aws): VPC, 3 public + 3 private subnets, NAT per AZ
    // ---------------------------------------------------------------------
    val vpc = Vpc("$name-vpc", VpcArgs.builder()
        .cidrBlock("10.0.0.0/16")
        .enableDnsHostnames(true)
        .enableDnsSupport(true)
        .tags(baseTags + ("Name" to "$name-vpc"))
        .build())

    val igw = InternetGateway("$name-igw", InternetGatewayArgs.builder()
        .vpcId(vpc.id())
        .tags(baseTags)
        .build())

    val publicRt = RouteTable("$name-public-rt", RouteTableArgs.builder()
        .vpcId(vpc.id())
        .routes(RouteTableRouteArgs.builder().cidrBlock("0.0.0.0/0").gatewayId(igw.id()).build())
        .tags(baseTags)
        .build())

    val publicSubnets = mutableListOf<Subnet>()
    val privateSubnets = mutableListOf<Subnet>()

    azs.forEachIndexed { i, az ->
        val pub = Subnet("$name-public-$i", SubnetArgs.builder()
            .vpcId(vpc.id())
            .availabilityZone(az)
            .cidrBlock("10.0.${i}.0/24")
            .mapPublicIpOnLaunch(false)
            .tags(baseTags + mapOf(
                "Name" to "$name-public-$az",
                "kubernetes.io/role/elb" to "1"))
            .build())
        RouteTableAssociation("$name-public-rta-$i", RouteTableAssociationArgs.builder()
            .subnetId(pub.id()).routeTableId(publicRt.id()).build())

        // One NAT gateway per AZ so losing an AZ doesn't cut egress for the others
        val eip = Eip("$name-nat-eip-$i", EipArgs.builder().domain("vpc").tags(baseTags).build())
        val nat = NatGateway("$name-nat-$i", NatGatewayArgs.builder()
            .allocationId(eip.id())
            .subnetId(pub.id())
            .tags(baseTags)
            .build(), CustomResourceOptions.builder().dependsOn(igw).build())

        // Large private subnets: Cilium ENI mode hands out real VPC IPs to pods
        val priv = Subnet("$name-private-$i", SubnetArgs.builder()
            .vpcId(vpc.id())
            .availabilityZone(az)
            .cidrBlock("10.0.${(i + 1) * 32}.0/19")
            .tags(baseTags + mapOf(
                "Name" to "$name-private-$az",
                "kubernetes.io/role/internal-elb" to "1"))
            .build())
        val privRt = RouteTable("$name-private-rt-$i", RouteTableArgs.builder()
            .vpcId(vpc.id())
            .routes(RouteTableRouteArgs.builder().cidrBlock("0.0.0.0/0").natGatewayId(nat.id()).build())
            .tags(baseTags)
            .build())
        RouteTableAssociation("$name-private-rta-$i", RouteTableAssociationArgs.builder()
            .subnetId(priv.id()).routeTableId(privRt.id()).build())

        publicSubnets += pub
        privateSubnets += priv
    }

    val publicSubnetIds = Output.all(publicSubnets.map { it.id() })
    val privateSubnetIds = Output.all(privateSubnets.map { it.id() })

    // ---------------------------------------------------------------------
    // KMS key for envelope encryption of Kubernetes secrets
    // ---------------------------------------------------------------------
    val secretsKey = Key("$name-secrets-kms", KeyArgs.builder()
        .description("EKS secrets envelope encryption for $name")
        .enableKeyRotation(true)
        .deletionWindowInDays(30)
        .tags(baseTags)
        .build())

    // ---------------------------------------------------------------------
    // Node IAM role (pulumi-aws)
    // ---------------------------------------------------------------------
    val nodeRole = Role("$name-node-role", RoleArgs.builder()
        .assumeRolePolicy("""
            {"Version":"2012-10-17","Statement":[{"Effect":"Allow",
             "Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
        """.trimIndent())
        .tags(baseTags)
        .build())

    listOf(
        "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
        "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
        // Needed by aws-node during bootstrap and by the Cilium operator for ENI IPAM.
        // Hardening option: move this to IRSA/Pod Identity for the cilium-operator only.
        "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy",
        // SSM Session Manager instead of SSH
        "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore",
    ).forEachIndexed { i, arn ->
        RolePolicyAttachment("$name-node-policy-$i", RolePolicyAttachmentArgs.builder()
            .role(nodeRole.name()).policyArn(arn).build())
    }

    // ---------------------------------------------------------------------
    // EKS cluster (pulumi-eks)
    // ---------------------------------------------------------------------
    val accessEntries = if (adminRoleArn != null) mapOf(
        "platform-admin" to AccessEntryArgs.builder()
            .principalArn(adminRoleArn)
            .accessPolicies(mapOf(
                "cluster-admin" to AccessPolicyAssociationArgs.builder()
                    .policyArn("arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy")
                    .accessScope(AccessPolicyAssociationAccessScopeArgs.builder().type("cluster").build())
                    .build()))
            .build()
    ) else emptyMap()

    val cluster = Cluster(name, ClusterArgs.builder()
        .version(k8sVersion)
        .vpcId(vpc.id())
        .publicSubnetIds(publicSubnetIds)
        .privateSubnetIds(privateSubnetIds)
        .nodeAssociatePublicIpAddress(false)
        // API endpoint: private for in-VPC traffic, public only from allow-listed CIDRs
        .endpointPrivateAccess(true)
        .endpointPublicAccess(true)
        .publicAccessCidrs(publicAccessCidrs)
        // Access entries instead of the deprecated aws-auth ConfigMap
        .authenticationMode(AuthenticationMode.Api)
        .accessEntries(accessEntries)
        .encryptionConfigKeyArn(secretsKey.arn())
        .enabledClusterLogTypes(listOf("api", "audit", "authenticator", "controllerManager", "scheduler"))
        .createOidcProvider(true)
        .skipDefaultNodeGroup(true)
        // Don't let Pulumi manage the VPC CNI add-on; Cilium replaces it below
        .useDefaultVpcCni(true)
        // CoreDNS is adopted as a managed add-on after Cilium is running (see bottom)
        .corednsAddonOptions(CoreDnsAddonOptionsArgs.builder().enabled(false).build())
        .tags(baseTags)
        .build())

    // ---------------------------------------------------------------------
    // Managed node group across 3 AZs, tainted until Cilium is ready
    // ---------------------------------------------------------------------
    val nodeGroup = ManagedNodeGroup("$name-ng", ManagedNodeGroupArgs.builder()
        .cluster(cluster)
        .nodeRoleArn(nodeRole.arn())
        .subnetIds(privateSubnetIds)
        .amiType("AL2023_x86_64_STANDARD")
        .instanceTypes(listOf(instanceType))
        .capacityType("ON_DEMAND")
        .enableIMDSv2(true)
        .scalingConfig(NodeGroupScalingConfigArgs.builder()
            .minSize(3).desiredSize(3).maxSize(9).build())
        .updateConfig(NodeGroupUpdateConfigArgs.builder().maxUnavailable(1).build())
        // Cilium removes this taint once its agent is ready on the node,
        // so app pods never start on a node without Cilium networking.
        .taints(listOf(NodeGroupTaintArgs.builder()
            .key("node.cilium.io/agent-not-ready")
            .value("true")
            .effect("NO_EXECUTE")
            .build()))
        .build(), ComponentResourceOptions.builder().dependsOn(cluster).build())

    // ---------------------------------------------------------------------
    // Kubernetes provider (pulumi-kubernetes) wired to the new cluster
    // ---------------------------------------------------------------------
    val k8s = Provider("$name-k8s", ProviderArgs.builder()
        .kubeconfig(cluster.kubeconfigJson())
        .enableServerSideApply(true)
        .build())

    // Disable the bootstrapped aws-node DaemonSet by giving it a node selector no node has
    // (this is the approach Cilium's EKS install guide uses).
    val awsNodePatch = DaemonSetPatch("$name-disable-aws-node", DaemonSetPatchArgs.builder()
        .metadata(ObjectMetaPatchArgs.builder()
            .name("aws-node")
            .namespace("kube-system")
            .annotations(mapOf("pulumi.com/patchForce" to "true"))
            .build())
        .spec(DaemonSetSpecPatchArgs.builder()
            .template(PodTemplateSpecPatchArgs.builder()
                .spec(PodSpecPatchArgs.builder()
                    .nodeSelector(mapOf("io.cilium/aws-node-enabled" to "true"))
                    .build())
                .build())
            .build())
        .build(), CustomResourceOptions.builder()
            .provider(k8s)
            .dependsOn(nodeGroup)
            .build())

    // ---------------------------------------------------------------------
    // Cilium (ENI IPAM mode, native routing, WireGuard, Hubble)
    // ---------------------------------------------------------------------
    val cilium = Release("cilium", ReleaseArgs.builder()
        .name("cilium")
        .chart("cilium")
        .version(ciliumVersion)
        .namespace("kube-system")
        .repositoryOpts(RepositoryOptsArgs.builder().repo("https://helm.cilium.io/").build())
        .values(mapOf<String, Any>(
            "eni" to mapOf("enabled" to true),
            "ipam" to mapOf("mode" to "eni"),
            "routingMode" to "native",
            "egressMasqueradeInterfaces" to "eth+",
            // Encrypt pod-to-pod traffic between nodes
            "encryption" to mapOf("enabled" to true, "type" to "wireguard"),
            // HA operator spread across nodes
            "operator" to mapOf("replicas" to 2),
            "hubble" to mapOf(
                "enabled" to true,
                "relay" to mapOf("enabled" to true),
            ),
            // Restart pods that came up before Cilium so they get Cilium-managed IPs
            "cni" to mapOf("exclusive" to true),
        ))
        .build(), CustomResourceOptions.builder()
            .provider(k8s)
            .dependsOn(awsNodePatch, nodeGroup)
            .build())

    // ---------------------------------------------------------------------
    // Adopt CoreDNS as an EKS managed add-on once pod networking works (pulumi-aws)
    // ---------------------------------------------------------------------
    AwsEksAddon("$name-coredns", AwsEksAddonArgs.builder()
        .clusterName(cluster.eksCluster().apply { it.name() })
        .addonName("coredns")
        .resolveConflictsOnCreate("OVERWRITE")
        .resolveConflictsOnUpdate("OVERWRITE")
        .tags(baseTags)
        .build(), CustomResourceOptions.builder().dependsOn(cilium).build())

    // ---------------------------------------------------------------------
    // EBS CSI driver (pulumi-aws) using EKS Pod Identity for its IAM permissions
    // ---------------------------------------------------------------------
    val clusterName = cluster.eksCluster().apply { it.name() }

    // Agent that hands out Pod Identity credentials to pods
    val podIdentityAgent = AwsEksAddon("$name-pod-identity-agent", AwsEksAddonArgs.builder()
        .clusterName(clusterName)
        .addonName("eks-pod-identity-agent")
        .resolveConflictsOnCreate("OVERWRITE")
        .resolveConflictsOnUpdate("OVERWRITE")
        .tags(baseTags)
        .build(), CustomResourceOptions.builder().dependsOn(cilium).build())

    val ebsCsiRole = Role("$name-ebs-csi-role", RoleArgs.builder()
        .assumeRolePolicy("""
            {"Version":"2012-10-17","Statement":[{"Effect":"Allow",
             "Principal":{"Service":"pods.eks.amazonaws.com"},
             "Action":["sts:AssumeRole","sts:TagSession"]}]}
        """.trimIndent())
        .tags(baseTags)
        .build())

    val ebsCsiPolicy = RolePolicyAttachment("$name-ebs-csi-policy", RolePolicyAttachmentArgs.builder()
        .role(ebsCsiRole.name())
        .policyArn("arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy")
        .build())

    val ebsCsi = AwsEksAddon("$name-ebs-csi", AwsEksAddonArgs.builder()
        .clusterName(clusterName)
        .addonName("aws-ebs-csi-driver")
        .resolveConflictsOnCreate("OVERWRITE")
        .resolveConflictsOnUpdate("OVERWRITE")
        .podIdentityAssociations(AddonPodIdentityAssociationArgs.builder()
            .serviceAccount("ebs-csi-controller-sa")
            .roleArn(ebsCsiRole.arn())
            .build())
        .tags(baseTags)
        .build(), CustomResourceOptions.builder()
            .dependsOn(podIdentityAgent, ebsCsiPolicy, cilium)
            .build())

    // Encrypted gp3 as the cluster's default StorageClass
    StorageClass("gp3", StorageClassArgs.builder()
        .metadata(ObjectMetaArgs.builder()
            .name("gp3")
            .annotations(mapOf("storageclass.kubernetes.io/is-default-class" to "true"))
            .build())
        .provisioner("ebs.csi.aws.com")
        .volumeBindingMode("WaitForFirstConsumer") // create the volume in the pod's AZ
        .allowVolumeExpansion(true)
        .reclaimPolicy("Delete")
        .parameters(mapOf("type" to "gp3", "encrypted" to "true"))
        .build(), CustomResourceOptions.builder()
            .provider(k8s)
            .dependsOn(ebsCsi)
            .build())

    // ---------------------------------------------------------------------
    // Outputs
    // ---------------------------------------------------------------------
    ctx.export("clusterName", cluster.eksCluster().apply { it.name() })
    ctx.export("vpcId", vpc.id())
    ctx.export("kubeconfig", cluster.kubeconfig().asSecret())
}
