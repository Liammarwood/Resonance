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


package eks;

import com.pulumi.Context;
import com.pulumi.Pulumi;
import com.pulumi.core.Output;
import com.pulumi.resources.CustomResourceOptions;
import com.pulumi.resources.Resource;

import com.pulumi.aws.cloudwatch.LogGroup;
import com.pulumi.aws.cloudwatch.LogGroupArgs;
import com.pulumi.aws.ec2.FlowLog;
import com.pulumi.aws.ec2.FlowLogArgs;
import com.pulumi.aws.ec2.LaunchTemplate;
import com.pulumi.aws.ec2.LaunchTemplateArgs;
import com.pulumi.aws.ec2.inputs.LaunchTemplateBlockDeviceMappingArgs;
import com.pulumi.aws.ec2.inputs.LaunchTemplateBlockDeviceMappingEbsArgs;
import com.pulumi.aws.ec2.inputs.LaunchTemplateMetadataOptionsArgs;
import com.pulumi.aws.ec2.inputs.LaunchTemplateTagSpecificationArgs;
import com.pulumi.aws.eks.AccessEntry;
import com.pulumi.aws.eks.AccessEntryArgs;
import com.pulumi.aws.eks.AccessPolicyAssociation;
import com.pulumi.aws.eks.AccessPolicyAssociationArgs;
import com.pulumi.aws.eks.Addon;
import com.pulumi.aws.eks.AddonArgs;
import com.pulumi.aws.eks.Cluster;
import com.pulumi.aws.eks.ClusterArgs;
import com.pulumi.aws.eks.NodeGroup;
import com.pulumi.aws.eks.NodeGroupArgs;
import com.pulumi.aws.eks.PodIdentityAssociation;
import com.pulumi.aws.eks.PodIdentityAssociationArgs;
import com.pulumi.aws.eks.inputs.AccessPolicyAssociationAccessScopeArgs;
import com.pulumi.aws.eks.inputs.ClusterAccessConfigArgs;
import com.pulumi.aws.eks.inputs.ClusterEncryptionConfigArgs;
import com.pulumi.aws.eks.inputs.ClusterEncryptionConfigProviderArgs;
import com.pulumi.aws.eks.inputs.ClusterVpcConfigArgs;
import com.pulumi.aws.eks.inputs.NodeGroupLaunchTemplateArgs;
import com.pulumi.aws.eks.inputs.NodeGroupScalingConfigArgs;
import com.pulumi.aws.eks.inputs.NodeGroupUpdateConfigArgs;
import com.pulumi.aws.iam.Role;
import com.pulumi.aws.iam.RoleArgs;
import com.pulumi.aws.iam.RolePolicy;
import com.pulumi.aws.iam.RolePolicyArgs;
import com.pulumi.aws.iam.RolePolicyAttachment;
import com.pulumi.aws.iam.RolePolicyAttachmentArgs;
import com.pulumi.aws.kms.Alias;
import com.pulumi.aws.kms.AliasArgs;
import com.pulumi.aws.kms.Key;
import com.pulumi.aws.kms.KeyArgs;

import com.pulumi.awsx.ec2.Vpc;
import com.pulumi.awsx.ec2.VpcArgs;
import com.pulumi.awsx.ec2.enums.NatGatewayStrategy;
import com.pulumi.awsx.ec2.enums.SubnetType;
import com.pulumi.awsx.ec2.inputs.NatGatewayConfigurationArgs;
import com.pulumi.awsx.ec2.inputs.SubnetSpecArgs;

import java.util.Arrays;
import java.util.List;
import java.util.Map;

/**
 * Production-grade, hardened EKS:
 *  - 3-AZ VPC, nodes + control-plane ENIs in private subnets, one NAT per AZ, VPC flow logs
 *  - Private API endpoint (public access only if CIDRs are explicitly allowed)
 *  - KMS envelope encryption of Kubernetes Secrets (key rotation on)
 *  - All control-plane log types to CloudWatch with retention
 *  - Access entries (API auth mode) — no aws-auth ConfigMap, no implicit creator admin
 *  - Managed node group: AL2023, IMDSv2 required (hop limit 1), encrypted gp3, SSM instead of SSH
 *  - EKS Pod Identity for workload IAM; VPC CNI with NetworkPolicy enforcement enabled
 */
public class App {

    private static final String EKS_ADMIN_POLICY =
            "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy";

    public static void main(String[] args) {
        Pulumi.run(App::stack);
    }

    private static void stack(Context ctx) {
        var cfg = ctx.config();
        var clusterName = cfg.get("clusterName").orElse("prod-eks");
        var k8sVersion = cfg.get("kubernetesVersion").orElse("1.35");
        var adminRoleArn = cfg.require("adminRoleArn");
        var vpcCidr = cfg.get("vpcCidr").orElse("10.0.0.0/16");
        var publicCidrs = csv(cfg.get("publicAccessCidrs").orElse(""));
        var instanceTypes = csv(cfg.get("nodeInstanceTypes").orElse("m7i.large"));
        int minSize = Integer.parseInt(cfg.get("nodeMinSize").orElse("3"));
        int desiredSize = Integer.parseInt(cfg.get("nodeDesiredSize").orElse("3"));
        int maxSize = Integer.parseInt(cfg.get("nodeMaxSize").orElse("9"));
        int diskGb = Integer.parseInt(cfg.get("nodeDiskGb").orElse("50"));
        int logRetention = Integer.parseInt(cfg.get("logRetentionDays").orElse("90"));

        // ---------------------------------------------------------------- KMS
        var secretsKey = new Key("eks-secrets-key", KeyArgs.builder()
                .description("Envelope encryption for Kubernetes Secrets in " + clusterName)
                .enableKeyRotation(true)
                .deletionWindowInDays(30)
                .build(),
                CustomResourceOptions.builder().protect(true).build());

        new Alias("eks-secrets-alias", AliasArgs.builder()
                .name("alias/" + clusterName + "-secrets")
                .targetKeyId(secretsKey.keyId())
                .build());

        // ---------------------------------------------------------------- VPC
        var vpc = new Vpc("eks-vpc", VpcArgs.builder()
                .cidrBlock(vpcCidr)
                .numberOfAvailabilityZones(3)
                .enableDnsHostnames(true)
                .enableDnsSupport(true)
                .natGateways(NatGatewayConfigurationArgs.builder()
                        .strategy(NatGatewayStrategy.OnePerAz)   // no single-AZ egress SPOF
                        .build())
                .subnetSpecs(
                        SubnetSpecArgs.builder()
                                .type(SubnetType.Public)
                                .cidrMask(22)
                                .tags(Map.of("kubernetes.io/role/elb", "1"))
                                .build(),
                        SubnetSpecArgs.builder()
                                .type(SubnetType.Private)
                                .cidrMask(19)                     // large: VPC CNI uses a VPC IP per pod
                                .tags(Map.of("kubernetes.io/role/internal-elb", "1"))
                                .build())
                .tags(Map.of("Name", clusterName + "-vpc"))
                .build());

        // VPC flow logs -> CloudWatch
        var flowLogGroup = new LogGroup("vpc-flow-logs", LogGroupArgs.builder()
                .name("/aws/vpc/" + clusterName + "/flow-logs")
                .retentionInDays(logRetention)
                .build());

        var flowLogRole = new Role("vpc-flow-logs-role", RoleArgs.builder()
                .assumeRolePolicy(trustPolicy("vpc-flow-logs.amazonaws.com", false))
                .build());

        new RolePolicy("vpc-flow-logs-policy", RolePolicyArgs.builder()
                .role(flowLogRole.id())
                .policy(flowLogGroup.arn().applyValue(arn -> """
                        {"Version":"2012-10-17","Statement":[{"Effect":"Allow",
                          "Action":["logs:CreateLogStream","logs:PutLogEvents",
                                    "logs:DescribeLogGroups","logs:DescribeLogStreams"],
                          "Resource":["%s","%s:*"]}]}""".formatted(arn, arn)))
                .build());

        new FlowLog("vpc-flow-log", FlowLogArgs.builder()
                .vpcId(vpc.vpcId())
                .trafficType("ALL")
                .logDestination(flowLogGroup.arn())
                .iamRoleArn(flowLogRole.arn())
                .build());

        // ---------------------------------------------------- Control plane
        var clusterRole = new Role("eks-cluster-role", RoleArgs.builder()
                .assumeRolePolicy(trustPolicy("eks.amazonaws.com", true))
                .build());
        var clusterPolicy = attach("eks-cluster-policy", clusterRole,
                "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy");

        // Pre-create so retention is enforced (EKS would otherwise create it with "never expire")
        var controlPlaneLogs = new LogGroup("eks-control-plane-logs", LogGroupArgs.builder()
                .name("/aws/eks/" + clusterName + "/cluster")
                .retentionInDays(logRetention)
                .build());

        var vpcConfig = ClusterVpcConfigArgs.builder()
                .subnetIds(vpc.privateSubnetIds())
                .endpointPrivateAccess(true)
                .endpointPublicAccess(!publicCidrs.isEmpty());
        if (!publicCidrs.isEmpty()) {
            vpcConfig.publicAccessCidrs(publicCidrs);
        }

        var cluster = new Cluster("eks", ClusterArgs.builder()
                .name(clusterName)
                .version(k8sVersion)
                .roleArn(clusterRole.arn())
                .vpcConfig(vpcConfig.build())
                .accessConfig(ClusterAccessConfigArgs.builder()
                        .authenticationMode("API")
                        .bootstrapClusterCreatorAdminPermissions(false)
                        .build())
                .encryptionConfig(ClusterEncryptionConfigArgs.builder()
                        .resources("secrets")
                        .provider(ClusterEncryptionConfigProviderArgs.builder()
                                .keyArn(secretsKey.arn())
                                .build())
                        .build())
                .enabledClusterLogTypes("api", "audit", "authenticator", "controllerManager", "scheduler")
                .build(),
                CustomResourceOptions.builder()
                        .dependsOn(clusterPolicy, controlPlaneLogs)
                        .protect(true)
                        .build());

        // ------------------------------------------------ Cluster access (RBAC)
        var adminEntry = new AccessEntry("admin-access", AccessEntryArgs.builder()
                .clusterName(cluster.name())
                .principalArn(adminRoleArn)
                .type("STANDARD")
                .build());

        new AccessPolicyAssociation("admin-access-policy", AccessPolicyAssociationArgs.builder()
                .clusterName(cluster.name())
                .principalArn(adminRoleArn)
                .policyArn(EKS_ADMIN_POLICY)
                .accessScope(AccessPolicyAssociationAccessScopeArgs.builder().type("cluster").build())
                .build(),
                CustomResourceOptions.builder().dependsOn(adminEntry).build());

        // ------------------------------------- Add-ons needed before nodes join
        var vpcCni = addon("vpc-cni", cluster,
                "{\"enableNetworkPolicy\":\"true\"}", List.of());
        var kubeProxy = addon("kube-proxy", cluster, null, List.of());
        var podIdentityAgent = addon("eks-pod-identity-agent", cluster, null, List.of());

        // ------------------------------------------------------------ Nodes
        var nodeRole = new Role("eks-node-role", RoleArgs.builder()
                .assumeRolePolicy(trustPolicy("ec2.amazonaws.com", false))
                .build());
        var nodeAttachments = List.<Resource>of(
                attach("node-worker", nodeRole, "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"),
                attach("node-ecr", nodeRole, "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly"),
                attach("node-cni", nodeRole, "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"),
                // Shell access via SSM Session Manager — no SSH keys, no port 22
                attach("node-ssm", nodeRole, "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"));

        var launchTemplate = new LaunchTemplate("eks-nodes-lt", LaunchTemplateArgs.builder()
                .namePrefix(clusterName + "-nodes-")
                .metadataOptions(LaunchTemplateMetadataOptionsArgs.builder()
                        .httpEndpoint("enabled")
                        .httpTokens("required")          // IMDSv2 only
                        .httpPutResponseHopLimit(1)      // pods cannot reach node IMDS/credentials
                        .build())
                .blockDeviceMappings(LaunchTemplateBlockDeviceMappingArgs.builder()
                        .deviceName("/dev/xvda")
                        .ebs(LaunchTemplateBlockDeviceMappingEbsArgs.builder()
                                .volumeSize(diskGb)
                                .volumeType("gp3")
                                .encrypted("true")
                                .deleteOnTermination("true")
                                .build())
                        .build())
                .tagSpecifications(LaunchTemplateTagSpecificationArgs.builder()
                        .resourceType("instance")
                        .tags(Map.of("Name", clusterName + "-node"))
                        .build())
                .build());

        var nodeGroupDeps = new java.util.ArrayList<Resource>(nodeAttachments);
        nodeGroupDeps.addAll(List.of(vpcCni, kubeProxy, podIdentityAgent));

        var nodeGroup = new NodeGroup("general", NodeGroupArgs.builder()
                .clusterName(cluster.name())
                .nodeGroupNamePrefix("general-")
                .nodeRoleArn(nodeRole.arn())
                .subnetIds(vpc.privateSubnetIds())
                .amiType("AL2023_x86_64_STANDARD")
                .capacityType("ON_DEMAND")
                .instanceTypes(instanceTypes)
                .scalingConfig(NodeGroupScalingConfigArgs.builder()
                        .minSize(minSize)
                        .desiredSize(desiredSize)
                        .maxSize(maxSize)
                        .build())
                .updateConfig(NodeGroupUpdateConfigArgs.builder()
                        .maxUnavailablePercentage(33)
                        .build())
                .launchTemplate(NodeGroupLaunchTemplateArgs.builder()
                        .id(launchTemplate.id())
                        .version(launchTemplate.latestVersion().applyValue(v -> Integer.toString(v)))
                        .build())
                .labels(Map.of("workload", "general"))
                .build(),
                CustomResourceOptions.builder()
                        .dependsOn(nodeGroupDeps)
                        // let Cluster Autoscaler / Karpenter own desired size
                        .ignoreChanges("scalingConfig.desiredSize")
                        .build());

        // ------------------------------- Add-ons that need running nodes
        addon("coredns", cluster, null, List.of(nodeGroup));

        // EBS CSI driver with least-privilege IAM via Pod Identity
        var ebsCsiRole = new Role("ebs-csi-role", RoleArgs.builder()
                .assumeRolePolicy(trustPolicy("pods.eks.amazonaws.com", true))
                .build());
        attach("ebs-csi-policy", ebsCsiRole,
                "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy");

        var ebsCsiIdentity = new PodIdentityAssociation("ebs-csi-identity",
                PodIdentityAssociationArgs.builder()
                        .clusterName(cluster.name())
                        .namespace("kube-system")
                        .serviceAccount("ebs-csi-controller-sa")
                        .roleArn(ebsCsiRole.arn())
                        .build(),
                CustomResourceOptions.builder().dependsOn(podIdentityAgent).build());

        addon("aws-ebs-csi-driver", cluster, null, List.of(nodeGroup, ebsCsiIdentity));

        // ---------------------------------------------------------- Outputs
        ctx.export("clusterName", cluster.name());
        ctx.export("clusterEndpoint", cluster.endpoint());
        ctx.export("clusterSecurityGroupId",
                cluster.vpcConfig().applyValue(v -> v.clusterSecurityGroupId().orElse("")));
        ctx.export("vpcId", vpc.vpcId());
        ctx.export("privateSubnetIds", vpc.privateSubnetIds());
        ctx.export("nodeRoleArn", nodeRole.arn());
        ctx.export("kubeconfigCommand", Output.format(
                "aws eks update-kubeconfig --name %s --role-arn %s", cluster.name(), adminRoleArn));
    }

    // ------------------------------------------------------------ helpers

    private static Addon addon(String name, Cluster cluster, String configJson, List<Resource> deps) {
        var args = AddonArgs.builder()
                .clusterName(cluster.name())
                .addonName(name)
                .resolveConflictsOnCreate("OVERWRITE")
                .resolveConflictsOnUpdate("OVERWRITE");
        if (configJson != null) {
            args.configurationValues(configJson);
        }
        return new Addon("addon-" + name, args.build(),
                CustomResourceOptions.builder().dependsOn(deps).build());
    }

    private static RolePolicyAttachment attach(String name, Role role, String policyArn) {
        return new RolePolicyAttachment(name, RolePolicyAttachmentArgs.builder()
                .role(role.name())
                .policyArn(policyArn)
                .build());
    }

    private static String trustPolicy(String service, boolean tagSession) {
        var actions = tagSession ? "[\"sts:AssumeRole\",\"sts:TagSession\"]" : "\"sts:AssumeRole\"";
        return """
                {"Version":"2012-10-17","Statement":[{"Effect":"Allow",
                  "Principal":{"Service":"%s"},"Action":%s}]}""".formatted(service, actions);
    }

    private static List<String> csv(String value) {
        return Arrays.stream(value.split(","))
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .toList();
    }
}