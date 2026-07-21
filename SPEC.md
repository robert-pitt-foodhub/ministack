# OpenSearch CloudFormation Domain Support Specification

- **Status:** Draft for implementation
- **Target resource:** `AWS::OpenSearchService::Domain`
- **Target project:** MiniStack
- **Reference contract:** [AWS CloudFormation template reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-opensearchservice-domain.html)

## 1. Purpose

MiniStack SHALL support provisioning Amazon OpenSearch Service domains declared as
`AWS::OpenSearchService::Domain` resources in CloudFormation templates. The feature SHALL allow
CDK-generated stacks containing an OpenSearch domain to complete creation, update, replacement,
and deletion instead of failing with `Unsupported resource type`.

This specification defines CloudFormation-to-OpenSearch property mapping, physical identity,
intrinsic return values, lifecycle behavior, compatibility handling, failure behavior, and the
tests required for conformance.

## 2. Normative language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD
NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described
in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

**Implementation-defined** means behavior that an implementation may choose but must document and
apply consistently. No core behavior in this specification is implementation-defined unless it is
explicitly labeled as such.

## 3. Problem statement

MiniStack already implements the OpenSearch Service management plane, including domain CRUD,
configuration updates, tags, account-scoped state, optional Docker-backed data planes, and VPC
endpoint-shaped responses. Its CloudFormation provisioner registry does not contain
`AWS::OpenSearchService::Domain`. A stack containing that type therefore enters `CREATE_FAILED`
and normally rolls back to `ROLLBACK_COMPLETE`.

The implementation MUST bridge the CloudFormation resource to the existing OpenSearch state and
lifecycle helpers. It MUST NOT create a second, CloudFormation-only domain store.

## 4. Goals and non-goals

### 4.1 Goals

The implementation MUST:

1. Register `AWS::OpenSearchService::Domain` with create, update, and delete handlers.
2. Support explicit and CloudFormation-generated domain names.
3. Preserve a domain's physical identity across in-place stack updates.
4. Replace the domain when `DomainName` changes.
5. Accept every top-level property currently documented by AWS for the resource.
6. Apply properties already modeled by MiniStack's OpenSearch API to the shared domain record.
7. Retain accepted properties that have no runtime semantics so ordinary CDK output remains
   deployable and inspectable.
8. Reconcile tags on stack updates, including removing tags deleted from the template.
9. Expose the required `Ref` and `Fn::GetAtt` values.
10. Support the existing MiniStack public and VPC endpoint shapes.
11. Preserve normal CloudFormation rollback and stack-event behavior on handler failures.

### 4.2 Non-goals

The implementation MUST NOT:

1. Add support for the legacy `AWS::Elasticsearch::Domain` resource type.
2. Emulate OpenSearch capacity, availability zones, dedicated masters, UltraWarm, cold storage,
   service software deployment, snapshots, log delivery, Cognito, IAM Identity Center, encryption,
   or fine-grained access control beyond retaining and reporting their configuration.
3. Validate that referenced subnet, security group, Cognito, KMS, CloudWatch Logs, or IAM resources
   exist or are mutually compatible.
4. Create real VPC networking, DNS, routing, TLS, IAM enforcement, or network isolation for a
   domain.
5. Implement `DomainEndpointV2`, IPv6 connectivity, or the nested return attributes
   `AdvancedSecurityOptions.*` and `IdentityCenterOptions.*`.
6. Reconfigure or replace a running Docker-backed OpenSearch container when an in-place domain
   property changes.
7. Add or change the README support tables as part of this work.
8. Reproduce AWS's asynchronous provisioning delays or every AWS interruption classification.

## 5. System overview

### 5.1 Components

The feature consists of:

- A CloudFormation provisioner entry for `AWS::OpenSearchService::Domain`.
- Create, update, and delete provisioner functions in
  `ministack/services/cloudformation/provisioners.py`.
- Shared OpenSearch lifecycle helpers or equivalent calls into
  `ministack/services/opensearch.py`.
- OpenSearch domain-record support for retaining compatibility-only properties.
- CloudFormation regression tests in `tests/test_cfn.py`.

### 5.2 Responsibility boundaries

The CloudFormation provisioner SHALL own:

- CloudFormation physical-name generation.
- Translation between CloudFormation and OpenSearch property names.
- Deciding whether an update is in-place or a replacement.
- Tag-set reconciliation.
- Construction of CloudFormation resource attributes.

The OpenSearch service SHALL own:

- Domain validation and account-scoped storage.
- ARN, endpoint, status, and configuration record construction.
- Optional data-plane container creation and teardown.
- OpenSearch API visibility and tag storage.

The generic CloudFormation engine SHALL continue to own dependency ordering, intrinsic resolution,
stack status, events, rollback, and removal of resources deleted from a template. This feature
SHOULD NOT introduce OpenSearch-specific logic into the generic engine.

## 6. Resource contract

### 6.1 Physical identifier

The CloudFormation physical resource identifier and `Ref` value MUST be the domain name.

If `DomainName` is present, the provisioner MUST use its resolved string value unchanged after
normal validation by the OpenSearch service.

If `DomainName` is absent, the provisioner MUST generate a name that:

- is generated once during initial creation;
- remains unchanged during all later in-place updates;
- contains only lowercase ASCII letters, digits, and hyphens;
- begins with a lowercase ASCII letter;
- is between 3 and 28 characters inclusive;
- is derived from the stack name and logical resource ID and includes a collision-resistant suffix;
- does not regenerate merely because `_update_resource` invokes the update handler.

The implementation SHOULD reuse the existing CloudFormation physical-name conventions while
ensuring truncation does not remove the collision-resistant suffix.

### 6.2 ARN and resource ID

The domain ARN MUST use the existing MiniStack OpenSearch form:

```text
arn:aws:es:{region}:{account_id}:domain/{domain_name}
```

The OpenSearch resource ID MUST be:

```text
{account_id}/{domain_name}
```

The values MUST use the account and region active for the provisioning request.

### 6.3 CloudFormation return values

The provisioner MUST return these attributes:

| Contract | Value |
|---|---|
| `Ref` / physical ID | Domain name |
| `Fn::GetAtt Arn` | Domain ARN |
| `Fn::GetAtt DomainArn` | Domain ARN |
| `Fn::GetAtt DomainEndpoint` | The domain's public endpoint or existing `Endpoints.vpc` value |
| `Fn::GetAtt Id` | `{account_id}/{domain_name}` |

`Arn` and `DomainArn` MUST be identical. `DomainEndpoint` MUST be a non-empty host-and-port value
compatible with the endpoint already returned by MiniStack's OpenSearch `DescribeDomain` API. It
MUST NOT claim that MiniStack provides AWS DNS, TLS, or VPC network isolation.

Unsupported `Fn::GetAtt` names remain subject to the generic CloudFormation engine's existing
fallback behavior and are not part of this feature's conformance contract.

## 7. Property contract

### 7.1 Accepted top-level properties

The resource handler MUST accept and retain these resolved CloudFormation properties:

- `AccessPolicies`
- `AdvancedOptions`
- `AdvancedSecurityOptions`
- `AIMLOptions`
- `AutomatedSnapshotPauseOptions`
- `ClusterConfig`
- `CognitoOptions`
- `DeploymentStrategyOptions`
- `DomainEndpointOptions`
- `DomainName`
- `EBSOptions`
- `EncryptionAtRestOptions`
- `EngineVersion`
- `IdentityCenterOptions`
- `IPAddressType`
- `LogPublishingOptions`
- `NodeToNodeEncryptionOptions`
- `OffPeakWindowOptions`
- `SkipShardMigrationWait`
- `SnapshotOptions`
- `SoftwareUpdateOptions`
- `Tags`
- `VPCOptions`

The list above is the conformance baseline captured by this specification. An unrecognized or
future top-level property SHOULD be accepted and retained as opaque compatibility configuration
instead of failing the stack, provided doing so does not weaken validation of `DomainName` or
corrupt shared OpenSearch state.

### 7.2 Property mapping

The provisioner MUST construct an OpenSearch create payload as follows:

| CloudFormation property | OpenSearch representation |
|---|---|
| `DomainName` | `DomainName`, using the generated name when omitted |
| `Tags` | `TagList` |
| All other properties already modeled by the OpenSearch service | Same-named field |
| Recognized but unmodeled properties | Retained compatibility configuration |

At minimum, the shared domain record MUST apply the existing semantics for `EngineVersion`,
`ClusterConfig`, `EBSOptions`, `AccessPolicies`, `CognitoOptions`, `EncryptionAtRestOptions`,
`NodeToNodeEncryptionOptions`, `AdvancedOptions`, `DomainEndpointOptions`,
`AdvancedSecurityOptions`, `VPCOptions`, `SnapshotOptions`, `OffPeakWindowOptions`, and
`SoftwareUpdateOptions`.

`Tags` MUST NOT be copied into the domain status object as `Tags`; they MUST use the existing
OpenSearch ARN-keyed tag store so `ListTags` observes them.

### 7.3 Compatibility-only properties

Recognized properties without behavior in the OpenSearch emulator MUST be retained without
causing stack failure. Retention MUST satisfy both conditions:

1. The resolved property remains in the CloudFormation resource's stored `Properties`, as it does
   for other resource types.
2. The OpenSearch domain state retains an account-scoped deep copy sufficient for a later
   CloudFormation update to compare, replace, remove, or preserve the value.

Compatibility-only properties MUST NOT be advertised as implemented OpenSearch functionality.
They MAY be stored in a private, underscore-prefixed domain-record field so they are excluded from
`DescribeDomain` responses that do not define those fields. If an existing OpenSearch response
shape already supports a property, the implementation SHOULD expose it there instead of keeping a
duplicate private value.

The implementation MUST deep-copy retained dictionaries and lists. Mutating a caller-owned
template object after provisioning MUST NOT mutate domain state.

### 7.4 Validation

The handler MUST preserve existing OpenSearch validation for explicit and generated domain names.
An invalid explicit name or a duplicate name MUST fail the resource and surface the error through
the normal CloudFormation resource and stack failure events.

The handler SHOULD NOT add deep AWS schema validation for compatibility-only properties. Intrinsic
functions MUST be resolved by the CloudFormation engine before the provisioner receives them.

## 8. Lifecycle workflows

### 8.1 Create

Creation MUST perform these steps in order:

1. Resolve or generate the domain name.
2. Translate the CloudFormation properties into the OpenSearch representation.
3. Create the domain through shared OpenSearch lifecycle logic.
4. Store compatibility-only configuration.
5. Apply the complete requested tag set.
6. Return the physical ID and attributes from the created record.

On success, `DescribeDomain`, `ListDomainNames`, and `ListTags` MUST observe the new resource before
the CloudFormation resource reaches `CREATE_COMPLETE`.

If creation fails after allocating domain state, the provisioner MUST remove the partial domain,
its tags, change-progress state, and any optional data-plane containers before propagating the
failure. Generic stack rollback MUST remain safe if it subsequently calls delete.

### 8.2 In-place update

An update MUST preserve the existing physical ID unless the resolved `DomainName` differs from the
physical ID.

For an in-place update, the handler MUST:

1. Compute the full desired property set from `new_props`; omitted properties are removals, not
   requests to retain stale values.
2. Apply all modeled configuration changes to the existing domain record.
3. Restore the existing OpenSearch default for a modeled optional property removed from the
   template, or remove it from the record when the OpenSearch response contract permits omission.
4. Replace the retained compatibility configuration with the new compatibility-only property set.
5. Reconcile tags by key so added and changed tags are present and removed tags are absent.
6. Refresh the returned CloudFormation attributes from the resulting domain record.

Configuration application MAY complete immediately. The implementation MUST leave
`DescribeDomainChangeProgress` in a completed, internally consistent state when it records an
in-place change.

Except for `DomainName`, all accepted property changes SHALL be treated as in-place emulator
updates. This intentionally does not reproduce every AWS `Update requires` classification.
Changing `EngineVersion` MUST update the retained/modelled version without requiring an actual
Docker image change.

Submitting an update whose resolved properties and tags are unchanged MUST be idempotent and MUST
NOT create a second domain or data-plane container.

### 8.3 Replacement

When `new_props.DomainName` is present and differs from the current physical ID, the update handler
MUST perform a replacement. Removing an explicit `DomainName` also MUST perform a replacement with
a newly generated name. The update handler MUST retain or receive the logical resource ID so the
replacement name can follow the same naming contract as initial creation.

Replacement MUST proceed as follows:

1. Validate and create the new domain with the complete new property set.
2. Apply its tags and construct its attributes.
3. Delete the old domain only after the new domain is created successfully.
4. Return the new domain name as the physical ID.

If creation of the replacement fails, the old domain MUST remain present and unchanged. Any partial
new domain state MUST be cleaned up.

Adding an explicit `DomainName` to an auto-named resource MUST replace it when the requested name
differs from the existing physical ID. If the newly explicit value exactly equals the existing
physical ID, the update MAY remain in place because no externally observable identity changes.

VPC configuration changes, including adding or removing `VPCOptions`, SHALL be handled in place for
this emulator even where AWS documents an interruption or replacement distinction.

### 8.4 Delete

Deletion MUST remove:

- the account-scoped domain record;
- tags keyed by its ARN;
- change-progress state; and
- OpenSearch and Dashboards containers associated with that domain, when present.

Delete MUST be idempotent. A missing domain MUST be treated as successful cleanup so stack deletion
and rollback can complete after partial creation or manual resource removal.

After successful deletion, `DescribeDomain` MUST return `ResourceNotFoundException`, and the domain
MUST be absent from `ListDomainNames`.

## 9. VPC behavior

`VPCOptions` is in scope because the existing OpenSearch service already models it.

The provisioner MUST accept resolved `SubnetIds` and `SecurityGroupIds`, including values obtained
through `Ref` from MiniStack EC2 resources. The domain record MUST use the existing OpenSearch
normalization that supplies endpoint and descriptive VPC fields.

When VPC options are active:

- the CloudFormation `DomainEndpoint` attribute MUST use the value stored in `Endpoints.vpc`;
- the normal OpenSearch `DescribeDomain` VPC response shape MUST remain intact; and
- no existence, ownership, routing, or reachability check is required for the supplied IDs.

This is control-plane compatibility only. Tests MUST NOT assert real VPC connectivity.

## 10. Failure and recovery model

| Failure | Required behavior |
|---|---|
| Invalid explicit domain name | Resource enters failure through normal CloudFormation events; no domain remains |
| Duplicate domain name | Create or replacement fails; the pre-existing domain is not deleted or modified |
| Failure after partial create | Partial record, tags, progress state, and containers are cleaned up |
| Missing domain during update | Update fails with an operator-visible reason; it MUST NOT silently create an unrelated replacement unless `DomainName` changed |
| Missing domain during delete | Delete succeeds idempotently |
| Unsupported resource behavior inside an accepted property | Property is retained; stack creation does not fail solely because behavior is not emulated |
| Data-plane Docker unavailable | Preserve existing OpenSearch fallback to the stub endpoint; CloudFormation creation may still succeed |

Errors propagated to stack events MUST identify the logical resource and retain the actionable
OpenSearch error message. Errors and logs MUST NOT include secret values.

## 11. Observability and security

The implementation SHOULD emit existing CloudFormation lifecycle events and existing OpenSearch
service logs rather than adding a second status mechanism.

Debug and error logs SHOULD include the stack name, logical resource ID, domain name when known,
operation, and error class. They MUST NOT serialize full property dictionaries because
`AdvancedSecurityOptions.MasterUserOptions.MasterUserPassword` can contain plaintext credentials.

Passwords and other sensitive nested property values:

- MUST NOT appear in CloudFormation status reasons or returned attributes;
- MUST NOT appear in application logs;
- MAY be retained in the same in-memory/persisted state boundary as other resolved CloudFormation
  resource properties, because compatibility retention is required; and
- MUST NOT be duplicated into additional public response fields.

The feature MUST preserve account and region isolation already provided by MiniStack's OpenSearch
and CloudFormation state containers.

## 12. Reference algorithms

### 12.1 Create

```text
create(logical_id, props, stack_name):
    name = props.DomainName if present else generate_valid_name(stack_name, logical_id)
    desired = deep_copy(props)
    payload = map_to_opensearch_create(desired, DomainName=name, Tags->TagList)

    try:
        record = shared_opensearch_create(payload)
        retain_compatibility_properties(name, desired)
        attrs = attributes_from(record)
        return (name, attrs)
    catch error:
        cleanup_domain_if_created(name)
        raise error
```

### 12.2 Update

```text
update(logical_id, physical_id, old_props, new_props, stack_name):
    old_name_was_explicit = DomainName is present in old_props
    new_name_is_explicit = DomainName is present in new_props
    replacement_required = (
        (new_name_is_explicit and new_props.DomainName != physical_id)
        or (old_name_was_explicit and not new_name_is_explicit)
    )

    if replacement_required:
        replacement_props = deep_copy(new_props)
        if DomainName is absent:
            replacement_props.DomainName = generate_valid_name(stack_name, logical_id)
        new_id, attrs = create_replacement(replacement_props)
        try:
            delete(physical_id, old_props)
        catch error:
            delete(new_id, replacement_props)
            raise error
        return (new_id, attrs)

    require domain physical_id exists
    apply_modeled_desired_state(physical_id, old_props, new_props)
    replace_compatibility_state(physical_id, new_props)
    reconcile_tags(physical_id, old_props.Tags or [], new_props.Tags or [])
    return (physical_id, attributes_from(current_domain_record))
```

Implementations MAY structure these operations differently, but MUST preserve their externally
observable ordering and failure guarantees.

## 13. Validation matrix

The pull request MUST add deterministic tests covering the following cases.

| Case | Required assertions |
|---|---|
| Explicit-name create | Stack reaches `CREATE_COMPLETE`; physical ID and `Ref` equal the name; domain is visible through OpenSearch APIs |
| Auto-name create | Generated name satisfies the OpenSearch regex and length; it remains identical after an in-place update |
| Full CDK-style properties | Every documented top-level property is accepted; modeled fields are visible; unmodeled fields are retained; creation does not fail |
| Tags on create | `ListTags` returns exactly the declared tag set |
| In-place update | Physical ID is unchanged; modeled properties, compatibility properties, and attributes reflect desired state |
| Tag reconciliation | Added/changed tags appear and removed tags disappear |
| Property removal | Removed modeled and compatibility-only properties do not remain stale |
| Domain-name replacement | New domain exists, old domain is absent, and physical ID and attributes change |
| Failed replacement | Duplicate or invalid new name fails while the original domain remains intact |
| Return values | Stack outputs resolve `Ref`, `Arn`, `DomainArn`, `DomainEndpoint`, and `Id` correctly |
| VPC template | EC2 `Ref` values resolve into `VPCOptions`; `DomainEndpoint` equals the stored VPC endpoint; no connectivity assertion is made |
| Delete | Stack deletion removes domain, tags, progress state, and any test-owned data-plane resources |
| Idempotent delete | Manually absent domain does not prevent stack deletion or rollback |
| Invalid create and rollback | Stack reports the resource failure and leaves no partial OpenSearch state |
| Secret redaction | A failing resource containing a sentinel master password does not expose the sentinel in logs, events, or attributes |

The CloudFormation integration tests SHOULD use the existing `cfn` and OpenSearch boto3 fixtures and
the existing asynchronous stack waiter. They MUST run successfully without Docker and without
`OPENSEARCH_DATAPLANE=1`.

An OPTIONAL Docker-backed smoke test MAY verify that initial CloudFormation creation starts a usable
data plane and deletion tears it down. It MUST NOT be a required default-CI test because the
management-plane contract is sufficient for this feature and Docker image availability would make
the test nondeterministic.

## 14. Definition of done

The feature is complete only when:

1. `AWS::OpenSearchService::Domain` no longer produces `Unsupported resource type`.
2. Create, in-place update, replacement, and idempotent delete conform to this specification.
3. CloudFormation and direct OpenSearch API views refer to the same account-scoped domain record.
4. All documented top-level properties deploy successfully and are retained according to Section
   7.
5. `Ref` and all in-scope attributes resolve correctly for public and VPC-shaped domains.
6. Tag changes converge to the exact template-declared set.
7. Partial failures do not leak domains, tags, progress records, or containers.
8. The validation matrix's required tests pass with the repository's standard test command.
9. Existing `tests/test_opensearch.py` and CloudFormation tests remain green.
10. No README files are modified for this feature.
